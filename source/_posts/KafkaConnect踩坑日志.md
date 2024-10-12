---
title: Kafka Connect踩坑日志
date: 2023-05-21 12:30:00
tags: 
    - 技术
    - kafka
---


## 问题列表：

### 问题1: jdbc大表抽取需要开启游标模式

问题描述：
    使用kafka-connect-jdbc插件的时候，发现kafka connect进程出现OOM的问题。后来通过DUMP文件分析出来是大对象导致，阅读代码后发现，可以通过开启游标模式来解决这个问题。

解决方案：
    对于mysql而言，jdbc连接参数设置useCursorFetch=true

### 问题2: jdbc connector bulk模式会因为rebalance导致restart task

问题描述：
    在使用jdbc插件抽取数据时，sink connect报错重复主键异常。

解决方案：
    修改kafka connect和kafka-connect-jdbc源码，避免出现无效的rebalance
    增加幂等机制，如果rebalance还是发生了，通过kafka消息通知用户，让用户手动重启task，从而去避免重复数据的产生