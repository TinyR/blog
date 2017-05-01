---
title: HBase MemStore
tags: []
categories: []
date: 2016-07-30 17:30:16
---
## MemStore

## 为什么需要MemStore
1. memstore在内存中，提高HBase写性能。
2. 通过memstore先对keyValue排好序，然后将排好序的keyValue一同刷写。

## MemStore内部
### KeyValue存放  
　　MemStore内部使用**KeyValueSkipListSet**存放KeyValue。KeyValueSkipListSet本质上是一个类似**SkipListSet**，内部使用**ConcurrentSkipListMap**实现。  

### 添加KeyValue
```java
  long add(final KeyValue kv) {
    KeyValue toAdd = maybeCloneWithAllocator(kv);
    return internalAdd(toAdd);
  }

  private long internalAdd(final KeyValue toAdd) {
    long s = heapSizeChange(toAdd, addToKVSet(toAdd));
    timeRangeTracker.includeTimestamp(toAdd);
    this.size.addAndGet(s);
    return s;
  }

  private boolean addToKVSet(KeyValue e) {
    boolean b = this.kvset.add(e);
    setOldestEditTimeToNow();
    return b;
  }

```



### MemStore GC优化 


### MemStore MVCC相关

### MemStore flush


