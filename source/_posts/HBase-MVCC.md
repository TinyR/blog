---
title: HBase MVCC
tags: []
categories: []
date: 2016-07-29 15:49:19
---
## 为什么需要MVCC?
简单来说就是为了**提高性能**。  
HBase保证了行级别的ACID。考虑下面情况：  
### Write-Write同步
假如有两个并发线程同时对HBase同一行进行写操作。我们可以采用排他锁的方式进行同步，流程如下：  
1. 获得行锁  
2. 写Write-Ahead-Log(WAL)  
3. 更新MemStore  
4. 释放锁  

### Read-Write 同步
对于写操作，我们可以通过获得排他锁的方式保证ACID。但是对于读操作呢？一种方式是读也需要获得排他锁，然而这样的话读写操作都需要先获得行级排他锁，使得**读写操作都变慢了**。  
HBase使用Multiversion Concurrency Control(MVCC)来是读操作避免获得行锁。
## HBase MVCC
HBase的MVCC是这样的:  
写操作：  
1. 获得行锁，然后每个写操作赋予一个write number  
2. 每个写入的data cell都赋予这个write number    
读操作  
1. 每个读操作都记录一个read timstamp,也叫read point  
2. 读的时候会进行比较，返回的data cell满足write number <= read point。这样就不会读到读操作之后发生的写操作的数据了  
所以整个写流程变成：  
1. 获得行锁  
2. 获得新的write number  
3. 写Write-Ahead-Log(WAL)  
4. 更新MemStore  
5. 结束write number
6. 释放行锁  

## HBase实现



## 参考
Apache HBase Internals: Locking and Multiversion COncurrency Control