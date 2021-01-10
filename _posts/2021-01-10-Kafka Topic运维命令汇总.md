---
layout:     post 
title:      Kafka Topic运维命令汇总
subtitle:   消息队列 
date:       2021-01-10 
author:     Jay 
header-img: img/post-bg-hacker.jpg 
catalog:    true 
tags:
    - Kafka
---

# Kafka Topic运维命令

### 1.创建主题
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic topic-create --partitions 4 --replication-factor 2
```
##### 1.1查看主题信息
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --describe --topic topic-create

输出：
Topic: topic-create     PartitionCount: 4       ReplicationFactor: 2    Configs:
        Topic: topic-create     Partition: 0    Leader: 1       Replicas: 1,2   Isr: 1,2
        Topic: topic-create     Partition: 1    Leader: 2       Replicas: 2,0   Isr: 2,0
        Topic: topic-create     Partition: 2    Leader: 0       Replicas: 0,1   Isr: 0,1
        Topic: topic-create     Partition: 3    Leader: 1       Replicas: 1,0   Isr: 1,0
```
##### 1.2 指定分区副本分配方案创建主题
```shell
相同的副本分配方案
kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic topic-create-same --replica-assignment 1:2,2:0,0:1,1:0

不同的副本分配方案
kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic topic-create-diff  --replica-assignment 2:0,0:1,1:2,2:1

同一分区的副本节点不能重复 error
kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic topic-create-error  --replica-assignment 0:0,1:1,2:2,3:3
```
##### 1.3指定config参数创建主题
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic topic-config --partitions 1 --replication-factor 1 --config cleanup.policy=compact --config max.message.bytes=10000
```
##### 1.4创建主题时避免topic重名异常
`--if-not-exists`
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic topic-config --partitions 1 --replication-factor 1 --config cleanup.policy=compact --config max.message.bytes=10000 --if-not-exists
```
##### 1.5创建主题时避免使用 . 或者 _
```shell
 kafka-topics.sh --zookeeper localhost:2181/kafka --create --replication-factor 1 --partitions 1 --topic topic.1_2

输出：
WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
Created topic topic.1_2.
```
### 2.查看主题
列举主题：
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --list
```
##### 2.1查看多个主题
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --describe --topic topic-config,topic-demo
```
##### 2.2查看所有覆盖配置的主题
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --describe --topics-with-overrides
```
##### 2.3查看有问题的主题
判断主题是否包含失效分区，副本不同步。
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --describe --under-replicated-partitions
```
##### 2.4查看主题是否包含没有leader副本的分区
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --describe --unavailable-partitions
```
### 3.修改主题
##### 3.1修改主题分区数
只支持增加分区数。对于基于消息key进行分区的topic而言，建议在一开始就设置好分区数量，避免以后对其进行调整，影响消息分区逻辑和消息顺序性。
```shell
 kafka-topics.sh --zookeeper localhost:2181/kafka --alter --topic topic-config --partitions 3
WARNING: If partitions are increased for a topic that has a key, the partition logic or ordering of the messages will be affected
```
##### 3.2修改主题配置
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --alter --topic topic-config --config max.message.bytes=20000
WARNING: Altering topic configuration from this script has been deprecated and may be removed in future releases.
         Going forward, please use kafka-configs.sh for this functionality
Updated config for topic topic-config.
```
##### 3.3删除主题特定配置，恢复默认配置
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --alter --topic topic-config --delete-config segment.bytes
```
### 4.删除主题
```shell
kafka-topics.sh --zookeeper localhost:2181/kafka --create --replication-factor 1 --partitions 1 --topic topic-delete

删除主题
kafka-topics.sh --zookeeper localhost:2181/kafka --delete --topic topic-delete
```




