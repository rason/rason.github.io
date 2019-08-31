---
title: Spring Kafka 的思考
date: 2019-07-02 14:58:06
tags:
---

Kafka 在高吞吐量的场景应用比较广泛, 但是可能会存在数据丢失, 那么 Kafka 究竟有多可靠? 本人只是简单实用过 Spring 集成的 Kafka, 并不是很熟悉, 所以下面思考的内容可能会有错.

#### 生产者

##### 1.我怎么知道发送到 Broker 的消息发送成功了?

Spring 中的 `KafkaTemplate` 的发送方法返回 `ListenableFuture` 可以设置回调, 发送成功或者失败都是有回调的.

如果发送失败了怎么办? 在回调的失败方式中打个日志就算了, 还是重新再发送, 如果一直失败会不会死循环了? 

目前我还没遇到发送失败的情况, 可能是样本还不够大, 这个问题还要深入思考.

另外, Spring Kafka 中有个配置项 `spring.kafka.producer.retries` 可以设置 **重试** 的次数, 这个是使用生产者内部的重试机制. 比如 Broker 返回 `LEADER_NOT_AVAILABLE` 错误, 生产者会自动进行重试. 但是重试可能会导致消息的重复, 在 `0.11.0.0` 版本之后, Kafka 提供了幂等的特性. 要不要用幂等的特性也是个问题, 不知道对吞吐量影响有多大, 所以一般情况还是让消费者来做幂等好了.

#####  2.发送成功了 Broker 还有没有可能丢失消息?

如果 Broker 只有一份数据, 机器挂了, 是不是就丢失了? 当然 Kafka 消息是保存在硬盘中, 一般情况下重启之后消息还是存在的. 但是 Broker 在收到消息之后不是马上就刷盘的, 在刷盘之前挂了那消息就真的丢失了.

所以, 数据复制才是靠谱的选择, 挂了一份还有其他的. 然而, 数据复制就必然会遇到数据一致性的问题, Spring Kafka 给了我们几个发送确认选择:

- `acks=0` 消息发送出去之后就认为写入成功了.
- `acks=1` 默认方式, Leader 节点写入后确认. 
- `acks=all/-1` 所有节点写入后确认.

显然, 这又是吞吐量与可靠性的一次 Trade Off.

#### 消费者

从本质上说, 保证了生产者到 Broker 之间的可靠, 那么 Kafka 其实就已经可靠了. 因为数据持久化在硬盘之后, 消费者可以根据情况控制偏移量来消费消息.

因此, 在消费者方面应该思考怎么刚好消费一次.

##### 1.什么情况下会出现消息重复消费?

当我们消费了消息, 但是偏移量没有提交, 那就会出现消息重复消息. 那么, 什么情况下会出现消费了消息, 但是偏移量没有被提交?

- 消息被消费后, 提交偏移量之前, 进程挂了
- `0.10.0.0` 之前的版本,消费者没有单独的心跳进程, 如果到了 `session.timeout.ms` 时间, 获取的一批数据还没处理完, 会认为消费者挂了, 提交偏移量会失败

##### 2.会不会出现消息还没有被消息, 但是偏移量被提交了, 导致消息 "丢失"?

这里说的丢失指的是被跳过处理了. 因为只要消息被成功保存到 Broker 中,消费者可以通过偏移量指定消费哪条消息, 并没有真的丢失.

如果 `enable.auto.commit=true`, 当我们拉取到的消息使用另外的线程去处理, 但是另外的线程可能因其他原因没处理成功, 这种情况下就可能会出现消息"丢失".

另外, 我们也可以想一下, 如果消息处理异常了, 但是还是自动提交了, 这种消息我们该怎么办? 重试, 记录日志或者是投递一个"死信队列"?

#### Broker

##### 1.副本因子

显然, Broker 要保证数据的可靠性就需要有副本, 并且最好不要在同一台机器上. `default.replication.factor` 用于设置副本数, 当副本数为 N 的时候, 可以容忍 N-1 个副本挂掉.

##### 2.脏副本的选举

`unclean.leader.election.enable` 0.11.0.0 之前的版本, 默认为 true; 之后的版本默认为 false. 如果设置为 true, 当没有同步副本可用的时候, 不同步的副本会成为 leader, 意味着有数据丢失. 如果设置为 false, 则意味着系统会处于不可用的状态, 该部分没有 leader 提供服务. 需要在可用性和一致性之间做取舍.

##### 3.最小同步副本数

`min.insync.replicas` 如果 broker 数为3, 最小同步副本数为2. 当2个同步副本中的一个出现问题, 集群便不会再接受生产者的发送消息请求. 同时客户端会收到 `NotEnoughReplicasException`. 此时, 消费者还可以继续读取存在的数据. 唯一的同步副本变成只读.

