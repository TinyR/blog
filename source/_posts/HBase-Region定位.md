---
title: HBase Region定位
tags: [HBase]
categories: [HBase,源码分析]
date: 2016-07-24 16:22:33
---
## 什么时候需要Region定位？  
　　**每个RegionServer包含多个的Region，每个Region包含一定范围内的行。**HBase client执行Get、Put等操作，都需要定位Region，与Region所在的RegionServer建立连接，进行RPC调用。  
## Region定位过程
### Catalog Table
HBase有两个特殊的表**-ROOT-**和**.MEAT.**。  
- -ROOT-保存了.META.表的位置。-ROOT-的位置信息存储在Zookeeper中。  
- .MEAT.保存了一系列Region信息。

### 一次查找流程
1. Client从Zookeeper查找-ROOT-所在的RegionServer为RS1  
2. Client与RS1建立通信，查找-ROOT-，找到rowkey所在的.META.的RegionServer为RS2  
3. Client与RS2建立通信，查找-META-，找到rowkey所属Region的RegionServer为RS3  

### 缓存
Client可以缓存Region location。

## 源码分析
### HConnectionImplementation的locateRegion方法：   
```java
private HRegionLocation locateRegion(final byte [] tableName,
      final byte [] row, boolean useCache, boolean retry)
    throws IOException {
      if (this.closed) throw new IOException(toString() + " closed");
      if (tableName == null || tableName.length == 0) {
        throw new IllegalArgumentException(
            "table name cannot be null or zero length");
      }
      ensureZookeeperTrackers();
      if (Bytes.equals(tableName, HConstants.ROOT_TABLE_NAME)) {
		//RootRegionTracker 监测保存在zookeeper中的-ROOT-所在的RegionServer
		//如果tableName为-ROOT-，则通过RootRegionTracker获得-ROOT-所在的RegionServer
        try {
          ServerName servername = this.rootRegionTracker.waitRootRegionLocation(this.rpcTimeout);
          LOG.debug("Looked up root region location, connection=" + this +
            "; serverName=" + ((servername == null)? "": servername.toString()));
          if (servername == null) return null;
          return new HRegionLocation(HRegionInfo.ROOT_REGIONINFO,
            servername.getHostname(), servername.getPort());
        } catch (InterruptedException e) {
          Thread.currentThread().interrupt();
          return null;
        }
      } else if (Bytes.equals(tableName, HConstants.META_TABLE_NAME)) {
		//如果tableName为-META-，则通过调用locateRegionInMeta在-ROOT-表中找到META表所在RS
        return locateRegionInMeta(HConstants.ROOT_TABLE_NAME, tableName, row,
            useCache, metaRegionLock, retry);
      } else {
        // Region not in the cache - have to go to the meta RS
		//如果tableName为普通表，则通过调用locateRegionInMeta在META表中找到tableName所在RS
        return locateRegionInMeta(HConstants.META_TABLE_NAME, tableName, row,
            useCache, userRegionLock, retry);
      }
    }
```
#### locateRegion方法理解：
1. RootRegionTracker 监测保存在zookeeper中的-ROOT-所在的RegionServer  
2. 如果tableName为-ROOT-，则通过RootRegionTracker获得-ROOT-所在的RegionServer  
3. 如果tableName为-META-，则通过调用locateRegionInMeta在-ROOT-表中找到META表所在RegionServer
4. 如果tableName为普通表，则通过调用locateRegionInMeta在META表中找到tableName所在RegionServer  

### HConnectionImplementation的locateRegionInMeta方法
```java
    private HRegionLocation locateRegionInMeta(final byte [] parentTable,
      final byte [] tableName, final byte [] row, boolean useCache,
      Object regionLockObject, boolean retry)
    throws IOException {
      HRegionLocation location;
      // If we are supposed to be using the cache, look in the cache to see if
      // we already have the region.
	  // 如果使用缓存的话，尝试从缓存中获取RegionServer的位置
      if (useCache) {
        location = getCachedLocation(tableName, row);
        if (location != null) {
          return location;
        }
      }

      int localNumRetries = retry ? numRetries : 1;
      // build the key of the meta region we should be looking for.
      // the extra 9's on the end are necessary to allow "exact" matches
      // without knowing the precise region names.
      byte [] metaKey = HRegionInfo.createRegionName(tableName, row,
        HConstants.NINES, false);

	  //重试机制
      for (int tries = 0; true; tries++) {
        if (tries >= localNumRetries) {
          throw new NoServerForRegionException("Unable to find region for "
            + Bytes.toStringBinary(row) + " after " + numRetries + " tries.");
        }

        HRegionLocation metaLocation = null;
        try {
          // locate the root or meta region
		  //递归找到上一层表(root或者meta)所在RegionServer位置
          metaLocation = locateRegion(parentTable, metaKey, true, false);
          // If null still, go around again.
          if (metaLocation == null) continue;
          HRegionInterface server =
            getHRegionConnection(metaLocation.getHostname(), metaLocation.getPort());

          Result regionInfoRow = null;
          if (useCache) {
			
            if (Bytes.equals(parentTable, HConstants.META_TABLE_NAME)
                && (getRegionCachePrefetch(tableName))) {
              // This block guards against two threads trying to load the meta
              // region at the same time. The first will load the meta region and
              // the second will use the value that the first one found.
			  //有可能其他线程缓存了meta表的位置
              synchronized (regionLockObject) {
                // Check the cache again for a hit in case some other thread made the
                // same query while we were waiting on the lock.
                location = getCachedLocation(tableName, row);
                if (location != null) {
                  return location;
                }
                // If the parent table is META, we may want to pre-fetch some
                // region info into the global region cache for this table.
                prefetchRegionCache(tableName, row);
              }
            }
            location = getCachedLocation(tableName, row);
            if (location != null) {
              return location;
            }
          } else {
            // If we are not supposed to be using the cache, delete any existing cached location
            // so it won't interfere.
            deleteCachedLocation(tableName, row);
          }

          // Query the root or meta region for the location of the meta region
		  //查找RegionServer位置
          regionInfoRow = server.getClosestRowBefore(
          metaLocation.getRegionInfo().getRegionName(), metaKey,
          HConstants.CATALOG_FAMILY);

          if (regionInfoRow == null) {
            throw new TableNotFoundException(Bytes.toString(tableName));
          }
          byte [] value = regionInfoRow.getValue(HConstants.CATALOG_FAMILY,
              HConstants.REGIONINFO_QUALIFIER);
          if (value == null || value.length == 0) {
            throw new IOException("HRegionInfo was null or empty in " +
              Bytes.toString(parentTable) + ", row=" + regionInfoRow);
          }
          // convert the row result into the HRegionLocation we need!
          HRegionInfo regionInfo = (HRegionInfo) Writables.getWritable(
              value, new HRegionInfo());
          // possible we got a region of a different table...
          if (!Bytes.equals(regionInfo.getTableName(), tableName)) {
            throw new TableNotFoundException(
                  "Table '" + Bytes.toString(tableName) + "' was not found, got: " +
                  Bytes.toString(regionInfo.getTableName()) + ".");
          }
          if (regionInfo.isSplit()) {
            throw new RegionOfflineException("the only available region for" +
              " the required row is a split parent," +
              " the daughters should be online soon: " +
              regionInfo.getRegionNameAsString());
          }
          if (regionInfo.isOffline()) {
            throw new RegionOfflineException("the region is offline, could" +
              " be caused by a disable table call: " +
              regionInfo.getRegionNameAsString());
          }

          value = regionInfoRow.getValue(HConstants.CATALOG_FAMILY,
              HConstants.SERVER_QUALIFIER);
          String hostAndPort = "";
          if (value != null) {
            hostAndPort = Bytes.toString(value);
          }
          if (hostAndPort.equals("")) {
            throw new NoServerForRegionException("No server address listed " +
              "in " + Bytes.toString(parentTable) + " for region " +
              regionInfo.getRegionNameAsString() + " containing row " +
              Bytes.toStringBinary(row));
          }

          // Instantiate the location
          String hostname = Addressing.parseHostname(hostAndPort);
          int port = Addressing.parsePort(hostAndPort);
          location = new HRegionLocation(regionInfo, hostname, port);
		  
		  //缓存location
          cacheLocation(tableName, location);

          return location;
        } catch (TableNotFoundException e) {
          // if we got this error, probably means the table just plain doesn't
          // exist. rethrow the error immediately. this should always be coming
          // from the HTable constructor.
          throw e;
        } catch (IOException e) {
          if (e instanceof RemoteException) {
            e = RemoteExceptionHandler.decodeRemoteException((RemoteException) e);
          }
          if (tries < numRetries - 1) {
            if (LOG.isDebugEnabled()) {
              LOG.debug("locateRegionInMeta parentTable=" +
                Bytes.toString(parentTable) + ", metaLocation=" +
                ((metaLocation == null)? "null": "{" + metaLocation + "}") +
                ", attempt=" + tries + " of " +
                this.numRetries + " failed; retrying after sleep of " +
                ConnectionUtils.getPauseTime(this.pause, tries) + " because: " + e.getMessage());
            }
          } else {
            throw e;
          }
          // Only relocate the parent region if necessary
          if(!(e instanceof RegionOfflineException ||
              e instanceof NoServerForRegionException)) {
            relocateRegion(parentTable, metaKey);
          }
        }
        try{
          Thread.sleep(ConnectionUtils.getPauseTime(this.pause, tries));
        } catch (InterruptedException e) {
          Thread.currentThread().interrupt();
          throw new IOException("Giving up trying to location region in " +
            "meta: thread is interrupted.");
        }
      }
    }
```

#### locateRegionInMeta方法理解
1.缓存  
2.递归查找  
3.重试机制  

### 缓存实现
#### 数据结构
**两层Map**  
HConnectionImplementation  
成员变量：
private final Map<HashedBytes, SoftValueSortedMap<byte[], HRegionLocation>> cachedRegionLocations  
Key为tableName。  
Value为该table的map of cached locations。  

SoftValueSortedMap:
内部使用TreeMap实现
key:某个Region的startKey
value:HRegionLocation




