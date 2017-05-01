---
title: HBase Put操作client端分析
tags: []
categories: []
date: 2016-07-30 12:03:18
---
## HTable Put


void put(Put put)：单个Put操作，单行。  
void put(List<Put> puts):多个Put操作一起发送给服务器端。  
void doPut(Put put):将Put操作放入缓冲区，并判断缓冲区是否满。  
flushCommits():执行RPC，将缓冲区内的put操作发到服务器端。  
void validatePut(final Put put):检查Put有效性  

### 写缓冲区
ArrayList<Put> writeBuffer:写缓冲区  
autoFlush:默认为true，每次调用put方法，最后都会自动调用flushCommits。可通过调用setAutoFlush(boolean autoFlush)进行设置。  
writeBufferSize:缓冲区大小，超过这个值，便会调用flushCommits向服务器端提交。可通过"hbase.client.write.buffer"进行配置  
currentWriteBufferSize:当前缓冲区大小



### 源码分析
#### doPut方法
```java
  private void doPut(Put put) throws IOException{
    validatePut(put);
    writeBuffer.add(put);
    currentWriteBufferSize += put.heapSize();
    if (currentWriteBufferSize > writeBufferSize) {
      flushCommits();
    }
  }
```
##### 主要流程：
1. Put有效性验证
2. 放入writeBuffer中
3. 判断缓冲区是否满，若满，则向服务器提交

#### put(Put put)方法
```java
  public void put(final Put put) throws IOException {
    doPut(put);
    if (autoFlush) {
      flushCommits();
    }
  }
```
##### 分析
　　每次先调用doPut放入缓冲区，再判断autoFlush，autoFlush默认为true，则每次flushCommits都只向服务器提交一个Put操作。

#### put(final List<Put> puts)方法
```java
  public void put(final List<Put> puts) throws IOException {
    for (Put put : puts) {
      doPut(put);
    }
    if (autoFlush) {
      flushCommits();
    }
  }
```
##### 分析
1. 将所有Put操作都放入到writeBuffer中
2. 判断是否autoFlush  
利用到了writeBuffer

#### flushCommits()
调用HConnection的**processBatch**(writeBuffer, tableName, pool, results)方法进行批处理，获得results。  
关于processBatch的分析，可参考另一篇笔记: HBase Client端批量处理操作分析

