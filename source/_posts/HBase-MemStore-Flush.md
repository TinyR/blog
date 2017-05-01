---
title: HBase MemStore Flush
tags: []
categories: []
date: 2016-07-31 19:15:21
---
## MemStore Flush
HBase基于LSM引擎。更新的数据会首先写入MemStore，当满足一定条件以后，MemStore会刷写到磁盘，生成一个HFile文件。**MemStore的flush单元是Region。即当MemStore flush时，同一Region下的所有MemStore都会被flush。**

### 触发条件
1. MemStore级别限制：当一个MemStore大小达到上限(hbase.hregion.memstore.flush.size)，该region下的所有MemStore都会被刷写到磁盘。
2. RegionServer级别：当总的MemStore占用内存超过hbase.regionserver.global.memstore.upperlimit,触发flush,从MemStore占用内存最多的region开始。直到内存占用比低于hbase.regionserver.global.memstore.lowerLimit停止。
3. 当WAL日志的数量达到hbase.regionserver.max.logs，会触发flush,flush的顺序基于MemStore的时间。直到WAL数量降到hbase.regionserver.max.logs一下。
4. 手动触发。shell命令flush 'tablename'或者flush 'region name'。

### flush流程
分成三个阶段：  
1. prepare阶段：将MemStore内数据集kvset作为快照snapshot，新建一个新的kvset，后续的写入操作都写入kvset中。prepare阶段需要获得region的updateLock的写锁。
2. flush阶段:将prepare阶段生成的snapshot持久化为临时文件，放到tmp目录下。
3. commit阶段:将flush阶段生成的文件从tmp目录移到对应的ColumnFamily目录下。




