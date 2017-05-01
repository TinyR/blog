---
title: HBase概念视图与物理视图
date: 2016-07-24 14:34:27
tags: [HBase]
categories: [HBase]
---
建议先看上一篇：HBase数据模型  

采用HBase官方文档的示例。  
## 概念视图:  
表webtable:  
![HBase概念视图](https://github.com/TinyR/images/blob/master/HBase/conceptual-view.PNG?raw=true)  

## 物理视图：  
anchor列族:    
![anchor列族](https://github.com/TinyR/images/blob/master/HBase/anchor.PNG?raw=true)

contents列族：   
![contents列族](https://github.com/TinyR/images/blob/master/HBase/contents.PNG?raw=true)   



参考：  
[HBase官方文档，Data Model部分](http://hbase.apache.org/book.html#datamodel)