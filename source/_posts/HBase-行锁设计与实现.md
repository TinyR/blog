---
title: HBase 行锁设计与实现
tags: [HBase]
categories: [HBase]
date: 2016-07-29 15:49:42
---
## HBase保证行级别的修改的原子性。

## HBase行锁实现
基于HBase 0.94  
现版本已经有所变化

### HRegion
![HRegion rowLock](https://github.com/TinyR/images/blob/master/HBase/HRegion_rowLock.PNG?raw=true)

lockedRows:存放rowKey与rowlock的键值对。  
lockIds:存放lock id与rowKey的键值对。  
lockIdGenerator:用于生成lock id


### 获取锁
　　obtainRowLock(final byte [] row) 调用 internalObtainRowLock(final HashedBytes rowKey, boolean waitForLock)
#### 主要流程
1. 新建锁rowLatch
2. 循环，直到获得锁(除非!waitForLock)：rowLatch放入到lockedRows中，成功则表示获得锁，如果锁已经被其他线程获得，则重新尝试。
3. 如果获得锁成功，循环，直到生成未被使用的lock id

#### 源码
```java
private Integer internalObtainRowLock(final HashedBytes rowKey, boolean waitForLock)
      throws IOException {
    checkRow(rowKey.getBytes(), "row lock");
    startRegionOperation();
    try {
		// 1. 新建锁
      CountDownLatch rowLatch = new CountDownLatch(1);

      
		//循环，直到获得锁(除非!waitForLock)
      while (true) {
        CountDownLatch existingLatch = lockedRows.putIfAbsent(rowKey, rowLatch);
        if (existingLatch == null) {
          break;
        } else {
          // row already locked
          if (!waitForLock) {
            return null;
          }
          try {
            if (!existingLatch.await(this.rowLockWaitDuration,
                            TimeUnit.MILLISECONDS)) {
              throw new IOException("Timed out on getting lock for row=" + rowKey);
            }
          } catch (InterruptedException ie) {
            LOG.warn("internalObtainRowLock interrupted for row=" + rowKey);
            InterruptedIOException iie = new InterruptedIOException();
            iie.initCause(ie);
            throw iie;
          }
        }
      }

      // loop until we generate an unused lock id
		//循环，直到生成未被使用的lock id
      while (true) {
        Integer lockId = lockIdGenerator.incrementAndGet();
        HashedBytes existingRowKey = lockIds.putIfAbsent(lockId, rowKey);
        if (existingRowKey == null) {
          return lockId;
        } else {
          // lockId already in use, jump generator to a new spot
          lockIdGenerator.set(rand.nextInt());
        }
      }
    } finally {
      closeRegionOperation();
    }
  }


```

### 释放锁
releaseRowLock(final Integer lockId)
#### 主要流程
1. 从lockids中删除lockid
2. 从lockedRows中删除rowLatch
3. 唤醒等待获得锁的其他线程

```java
  public void releaseRowLock(final Integer lockId) {
    if (lockId == null) return; // null lock id, do nothing
	// 从lockids中删除lockid
    HashedBytes rowKey = lockIds.remove(lockId);
    if (rowKey == null) {
      LOG.warn("Release unknown lockId: " + lockId);
      return;
    }
	// 从lockedRows中删除rowLatch
    CountDownLatch rowLatch = lockedRows.remove(rowKey);
    if (rowLatch == null) {
      LOG.error("Releases row not locked, lockId: " + lockId + " row: "
          + rowKey);
      return;
    }
	// 唤醒等待获得锁的其他线程
    rowLatch.countDown();
  }
```

## 参考
HBase-0.94.27源码

