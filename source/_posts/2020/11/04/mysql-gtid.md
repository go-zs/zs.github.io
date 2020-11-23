---
title: MySQL GTID详解
date: 2020-11-04 16:25:55
tags:
---

`GTID`是`MySQL`5.6版本的新特性，其全称是`Global Transaction Identifier`，可简化`MySQL`的主从切换以及`Failover`。

<!-- more -->


## 介绍

`GTID`用于在`Binlog`中唯一标识一个事务。当事务提交时，`MySQL`在写binlog的时候，会先写一个特殊的`Binlog Event`，类型为`GTID_Event`，指定下一个事务的`GTID`，然后再写事务的`Binlog`。主从同步时`GTID_Event`和事务的`Binlog`都会传递到从库，从库在执行的时候也是用同样的`GTID`去写`binlog`，这样主从同步以后，就可通过`GTID`确定从库同步到的位置了。也就是说，无论是级联情况，还是一主多从情况，都可以通过`GTID`自动定位，而无需像之前那样通过`File_name`和`File_position`定位了。


## 构成

* GTID = server_uuid:transaction_id 
* server_uuid 来源于 `auto.cnf`,服务的唯一标识，`MySQL`使用机器网卡、当前时间、随机数等拼接成一个128bit的uuid
* transaction_id `InnoDB` 每一个事物都存在一个事务唯一ID, 从 `1` 开始的自增计数，表示在这个主库上执行的第 `n` 个事务
* GTID: 在一组复制中，全局唯一


## 开启GTID

```
gtid_mode=ON(必选)  
enforce-gtid-consistency（必选）
log_bin=ON（可选）--高可用切换，最好设置ON  
log-slave-updates=ON（可选）--高可用切换，最好设置ON

```

## 功能

`GTID`的最大特性就是它的`Failover`能力，如下架构，当主库`A crash`时，需要进行主从切换，将B或C其中一台提升为主，传统模式我们无法确认哪台数据较新，由于同一个事务在每台机器上所在的`binlog`名字和位置都不一样，那么怎么找到C当前同步停止点，对应B的master_log_file和master_log_pos，需要通过程序对比或者借助`MHA`等工具。

`GTID`出现后，这个问题就显得非常简单。由于同一事务的`GTID`在所有节点上的值一致，那么根据C当前停止点的`GTID`就能唯一定位到B上的`GTID`。甚至由于`MASTER_AUTO_POSITION`功能的出现，我们都不需要知道`GTID`的具体值，直接使用`CHANGE MASTER TO MASTER_HOST='xxx'`, `MASTER_AUTO_POSITION=1`命令就可以直接完成`failover`的工作。