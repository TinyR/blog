---
title: HBase 数据写入server端分析
tags: []
categories: []
date: 2016-07-30 12:07:02
---
## HBase批量处理操作

HBase client端批处理操作，最终通过RPC,远程调用HRegionServer的multi方法。

## HRegionServer multi(MultiAction<R> multi)方法
### 主要流程：
1. 对每个Region下面的action进行排序
2. 对action进行分类，delete与put放一起，最终将会调用HRegion的batchMutate方法
3. 其他操作比如：get,append调用其他方法

## HRegion batchMutate方法
将会调用doMiniBatchMutation方法

## HRegion doMiniBatchMutation方法
### 主要流程
1. 获得尽可能多的行锁，且至少一个。
2. 更新delete/put操作所对应keyvalue的timestamps为最新。获得最新的mvcc number作为write number.
3. 写memstore
4. 构建 WAL edit
5. 向WAL添加edit但不sync wal.
6. 释放行锁
7. sync wal
8. 更新mvcc。





