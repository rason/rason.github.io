---
title: Kafka集群搭建
date: 2017-07-12 11:09:52
categories: BigData
tags: Kafka
description: Kafka集群搭建
---

## 前言

在上一篇文章中我们已经搭建好了ZooKeeper集群，现在就可以开始搭建Kafka集群了。

Kafka的很多管理信息都放在ZooKeeper中，这是大规模分布式系统设计中常用的一种技巧，比如Storm也是采用类似的思路。通过这样的方式，代理服务器成为完全无状态的，无须记载任何状态信息，这样对于消息系统的容错性和可扩展性都有很大的好处。

Kafka使用ZooKeeper保存的管理信息和实现的功能包括：

- 侦测代理服务器和消息消费者的动态加入和删除。

- 当动态加入或者删除代理服务器以及消息消费者后对消息系统进行负载均衡。

- 维护消费者和消息Topic以及数据分片的相互关系，并保存消费者当前读取消息的Offset。

- 数据副本管理信息。

## Kafka安装

### 1. 下载

[下载](https://www.apache.org/dyn/closer.cgi?path=/kafka/0.11.0.0/kafka_2.11-0.11.0.0.tgz)最近的稳定版本，然后解压。

```
> tar -xzf kafka_2.11-0.11.0.0.tgz
> cd kafka_2.11-0.11.0.0
```

### 2. 启动服务

Kafka需要用到ZooKeeper，所以我们要先启动ZooKeeper服务。上篇文章中我们已经搭建好了ZooKeeper集群，继续沿用。

修改`conf/server.properties`文件zookeeper配置：

```
zookeeper.connect=zoo1:2181,zoo2:2181,zoo3:2181
```

启动Kafka服务：

```
> bin/kafka-server-start.sh config/server.properties

...
[2017-07-11 14:21:09,780] INFO [Kafka Server 0], started (kafka.server.KafkaServer)
```

### 创建一个Topic

创建一个只有一个分片和一个副本的`test` Topic用于测试：

```
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```

查看topic 列表

```
> bin/kafka-topics.sh --list --zookeeper localhost:2181
test
```

另外，我们也可以选择配置代理服务器向一个不存在的topic发送消息时自动创建topic。

### 发送测试消息

运行生产者然后在控制台输入一些消息发送到代理服务器：

```
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
This is a message
This is another message
```

### 启动一个消费者

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
This is a message
This is another message
```

如果你将上述命令都运行在不同的终端中，那么你现在应该可以在生产者终端中输入消息，并看到它们出现在消费者终端。

这些命令行工具都有可选参数，不加任何参数运行这些命令可以看到用法信息。

<!-- more -->

### 搭建多个代理服务器集群

到目前为止我们已经运行了单一的代理服务器，但这没什么意思。对于Kafka来说，单一的代理就是集群大小为1，所以多运行几个无需作过多的改变，这就是使用ZooKeeper的好处，文章开头已经阐述。现在我们来运行三个代理节点（依然在这台机器上）。

首先，为每个代理创建一个配置文件：

```
> cp config/server.properties config/server-1.properties
> cp config/server.properties config/server-2.properties
```

然后，修改配置文件下列属性：

```
config/server-1.properties:
    broker.id=1
    listeners=PLAINTEXT://:9093
    log.dir=/tmp/kafka-logs-1
 
config/server-2.properties:
    broker.id=2
    listeners=PLAINTEXT://:9094
    log.dir=/tmp/kafka-logs-2
```

其中`broker.id`属性是集群中每个节点的唯一标识。我们还需要修改端口和日志路径，避免端口冲突和日志覆盖。

启动两个新的节点：

```
> bin/kafka-server-start.sh config/server-1.properties &
...
> bin/kafka-server-start.sh config/server-2.properties &
...
```

现在，我们来创建一个副本因子为3的topic：

```
> bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
```

*注意*，副本因子`-replication-factor`不能大于代理数。

好，现在我们已经有了一个集群，那么我们怎么知道哪个代理在做什么？运行`describe topics`命令看看：

```
> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
    Topic: my-replicated-topic  Partition: 0    Leader: 1   Replicas: 1,2,0 Isr: 1,2,0
```

第一行是分片信息汇总，后面的每一行代表每个分片的信息。由于我们创建的topic只有一个分片，所以只有一条信息。

- **Leader** ： 代表接受该分片读写请求的节点。每个节点都会成为每个随机选择分片的Leader。
- **Replicas** ： 代表拥有该分区日志副本的节点列表，无论它们是不是Leader，或者是否存在，也就是挂了的节点也可能存在这里。
- **Isr** ：  "in-sync" ISR集合。是Replicas列表的子集合，代表还在存活的，有可能被选举为Leader。

从上面的内容可以看出，节点1是该topic唯一分片的leader。

我们可以运行相同的命令看看开头`test` topic的详细信息：

```
> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
Topic:test  PartitionCount:1    ReplicationFactor:1 Configs:
    Topic: test Partition: 0    Leader: 0   Replicas: 0 Isr: 0
```

显而易见，这个topic没有其他副本，只有在server 0中，因为集群中只有这台机器。

好，现在来发布一些消息到新的topic：

```
> bin/kafka-console-producer.sh --broker-list localhost:9092 --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

然后消费这些信息：

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

现在来测试一下容错性，将作为Leader的代理1进程杀掉：

```
> ps aux | grep server-1.properties
7564 ttys002    0:15.91 /System/Library/Frameworks/JavaVM.framework/Versions/1.8/Home/bin/java...
> kill -9 7564
```

这样将会从ISR集合中选举一个leade，而节点1将不再在ISR集合中，因为已经挂了。

```
> bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
Topic:my-replicated-topic   PartitionCount:1    ReplicationFactor:3 Configs:
    Topic: my-replicated-topic  Partition: 0    Leader: 2   Replicas: 1,2,0 Isr: 2,0
```

但是消息依然可以被消费，即使最初写的leader已经挂掉了：

```
> bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --from-beginning --topic my-replicated-topic
...
my test message 1
my test message 2
^C
```

## 总结

本文主要学习了Kafka集群的搭建，通过这项工作，应该重点理解Topic分片和ISR副本管理机制，还应理解ZooKeeper所起的作用。
