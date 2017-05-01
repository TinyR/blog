---
title: HBase重试机制
tags: [HBase]
categories: [HBase]
date: 2016-07-29 13:27:00
---
## 重试机制
**HBase在执行RPC失败之后会执行重试机制。**


### 配置
hbase.client.retries.number:重试的最大次数
hbase.client.pause: