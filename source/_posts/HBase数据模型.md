---
title: HBase数据模型
date: 2016-07-24 13:45:55
tags: [HBase]
categories: [HBase]
---

　　我觉得关于HBase、Bigtable数据模型的最好阐述来源于论文《*Bigtable: A Distributed Storage System for Structured Data*》section2 Data Model:
>A Bigtable is a sparse, distributed, persistent multidimensional sorted map.
　　

　　关键词:***sparse distributed persistent multi-dimensional sorted map***  
　　以上几个关键词的详细阐述可参考博客[Understanting HBase and BigTable](http://jimbojw.com/wiki/index.php?title=Understanding_Hbase_and_BigTable)。   
　　Bigtable论文继续阐述:
>The map is indexed by a row key, column key, and a timestamp; each value in the map is an uninterpreted array of bytes.

>(row:string,column:string,time:int64) -> string

　　HBase使用的数据模型与Bigtable类似。  
　　了解了HBase、Bigtable的数据模型，再去看HBase的概念图与物理视图，便能很好地理解。
   
　　参考:   
　　*Bigtable: A Distributed Storage System for Structured Data*   
　　[Understanting HBase and BigTable](http://jimbojw.com/wiki/index.php?title=Understanding_Hbase_and_BigTable)  
　　[HBase官方文档，Data Model部分](http://hbase.apache.org/book.html#datamodel)
