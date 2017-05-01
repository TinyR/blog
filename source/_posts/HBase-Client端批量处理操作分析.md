---
title: HBase Client端批量处理操作分析
tags: [HBase,源码分析]
categories: [HBase,源码分析]
date: 2016-07-24 18:29:50
---
## 为什么需要批量处理操作
　　HTable的get(Get get),delete(Delete delete),put(Put put)都是单个操作。**每个Get/Put/Delete实际上都是一个RPC操作。**  
　　当一个应用程序需要进行大量Get/Put/Delete操作，比如：需要不断向服务器put数据。这时候可以通过**将多个数据通过一次RPC操作，提交给服务器，将减少RPC请求**，这样可以大大提高性能。
## 有哪些批处理操作
HBase提供了以下批处理操作：  
public Result[] get(List<Get> gets)  
public void batch(final List<?extends Row> actions, final Object[] results)  
public Object[] batch(final List<? extends Row> actions)  
public void put(final List<Put> puts)  

**Get、Put、Delete都实现了Row接口。**

## 源码分析
get(List<Get> gets)底层调用了batch(final List<? extends Row> actions)  
batch(final List<? extends Row> actions)、batch(final List<?extends Row> actions, final Object[] results)、put(final List<Put> puts)底层都调用了HConnection的processBatch(List<? extends Row> list, final byte[] tableName, ExecutorService pool, Object[] results) 方法。
### HConnection processBatch分析
参数说明：  
list:一系列操作  
tableName：表名  
pool:线程池  
results:用来存放最终结果  
processBatch调用processBatchCallback(List<? extends Row> list, byte[] tableName, ExecutorService pool, Object[] results, Batch.Callback<R> callback)方法。

### HConnection processBatchCallback分析
#### 主要流程  
1. 将操作请求(puts/gets/deletes)按照RegionServer分类  
2. 将每个操作请求集合放入线程池中执行  
3. 收集失败和成功信息  
4. 收集所有的失败操作，进行重试

#### 数据结构
<code>Action<R> action = new Action<R>(row, i);</code>  
MultiAction： 
成员变量：  
```java
  // map of regions to lists of puts/gets/deletes for that region.
  public Map<byte[], List<Action<R>>> actions =
    new TreeMap<byte[], List<Action<R>>>(
      Bytes.BYTES_COMPARATOR);
```   
**有序的Map**  
- key:regionName  
- value:一系列Get/Put/Delete操作  

processBatchCallback中使用的数据结构：  
Map<HRegionLocation,MultiAction<R>> actionByServer = new HashMap<HRegionLocation, MultionAction<R>>();  
- key: HRegionLocation，对应一个RegionServer  
- value: 一系列Region及其操作

#### processBatchBack源码
```java
public <R> void processBatchCallback(
        List<? extends Row> list,
        byte[] tableName,
        ExecutorService pool,
        Object[] results,
        Batch.Callback<R> callback)
    throws IOException, InterruptedException {
      
      //一些检查工作 
      if (results.length != list.size()) {
        throw new IllegalArgumentException(
            "argument results must be the same size as argument list");
      }
      if (list.isEmpty()) {
        return;
      }
		
	  //为后面工作做准备
      HRegionLocation [] lastServers = new HRegionLocation[results.length];
      List<Row> workingList = new ArrayList<Row>(list);
      boolean retry = true;
      // count that helps presize actions array
      int actionCount = 0;
	  
      //可能重试
      for (int tries = 0; tries < numRetries && retry; ++tries) {
		// ...
		
        // 1.按RegionServer分类Get/Put/Delete操作
        Map<HRegionLocation, MultiAction<R>> actionsByServer =
          new HashMap<HRegionLocation, MultiAction<R>>();
        for (int i = 0; i < workingList.size(); i++) {
          Row row = workingList.get(i);
          if (row != null) {
            HRegionLocation loc = locateRegion(tableName, row.getRow());
            byte[] regionName = loc.getRegionInfo().getRegionName();

            MultiAction<R> actions = actionsByServer.get(loc);
            if (actions == null) {
              actions = new MultiAction<R>();
              actionsByServer.put(loc, actions);
            }

            Action<R> action = new Action<R>(row, i);
            lastServers[i] = loc;
            actions.add(regionName, action);
          }
        }

        // 2.发送请求

        Map<HRegionLocation, Future<MultiResponse>> futures =
            new HashMap<HRegionLocation, Future<MultiResponse>>(
                actionsByServer.size());

        for (Entry<HRegionLocation, MultiAction<R>> e: actionsByServer.entrySet()) {
          futures.put(e.getKey(), pool.submit(createCallable(e.getKey(), e.getValue(), tableName)));
        }

        // 3.收集failure和success信息

        for (Entry<HRegionLocation, Future<MultiResponse>> responsePerServer
             : futures.entrySet()) {
          // ...
        }

		// 4. 找到failure，加入到workingList，重试

        retry = false;
        workingList.clear();
        actionCount = 0;
		
        for (int i = 0; i < results.length; i++) {
          // if null (fail) or instanceof Throwable && not instanceof DNRIOE
          // then retry that row. else dont.
          if (results[i] == null ||
              (results[i] instanceof Throwable &&
                  !(results[i] instanceof DoNotRetryIOException))) {

            retry = true;
            actionCount++;
            Row row = list.get(i);
            workingList.add(row);
            deleteCachedLocation(tableName, row.getRow());
          } else {
            if (results[i] != null && results[i] instanceof Throwable) {
              actionCount++;
            }
            // add null to workingList, so the order remains consistent with the original list argument.
            workingList.add(null);
          }
        }
      }

      List<Throwable> exceptions = new ArrayList<Throwable>(actionCount);
      List<Row> actions = new ArrayList<Row>(actionCount);
      List<String> addresses = new ArrayList<String>(actionCount);

      for (int i = 0 ; i < results.length; i++) {
        if (results[i] == null || results[i] instanceof Throwable) {
          exceptions.add((Throwable)results[i]);
          actions.add(list.get(i));
          addresses.add(lastServers[i].getHostnamePort());
        }
      }

      if (!exceptions.isEmpty()) {
        throw new RetriesExhaustedWithDetailsException(exceptions,
            actions,
            addresses);
      }
    }

```

## 参考
HBase-0.94.27源码


