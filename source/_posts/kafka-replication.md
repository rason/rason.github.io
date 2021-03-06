---
title: Kafka Replication
date: 2020-01-28 15:24:24
tags:
---

之前研究了一下 Redis 的主从复制，现在对比学习一下 Kafka 的数据复制。

### 副本

Kafka 用 Topic 来组织数据，每个 Topic 可以分为多个分区，每个分区可以有多个副本。Kafka 中的副本跟 Redis 类似，也是分为主从副本，主副本负责读写，从副本不处理客户端的请求，只是一个 BackUp	;当然在 Redis 中主从副本可以做读写分离，但是 Kafka 是不支持读写分离的，也没必要去做读写分离（后面会解析）。

Kafka 中的副本类型：

- Leader 副本：
  每个分区只有一个 Leader 副本。为了保证一致性，所有生产者请求和消费者请求都会经过这个副本。

- Follower 副本：
  除了 Leader 副本之外的都是 Follower 副本。Follower 副本不处理客户端请求，唯一的任务就是从 Leader 那里复制消息，当 Leader 挂了之后，其中一个 Follower 会被提升为 Leader。

### ISR

在 Redis 中，主从副本保持同步是通过主副本向从副本发送命令进行同步的。而 Kafka 则是 Follower 副本向 Leader 副本发送获取数据的请求进行同步，这种请求与消费者为了读取消息发送的请求是一样的。

既然存在数据的复制，那肯定会存在副本间数据不一致的情况。Kafka 为了保证数据的可靠性，定义了**同步的副本（in-sync replicas）**，只有数据被写到了所有的同步副本中才能被消费者读到，这个同步的副本组成的集合称为 **ISR** 。

那么，哪些副本属于 ISR ? 首先 Leader 副本肯定是同步副本，对于 Follower 副本需要满足以下条件：

- 与 Zookeeper 保持会话，即在过去 6s（可 `zookeeper.session.timeout.ms` 配置）内向 Zookeeper 发送过心跳。
- 在过去的 10s 内（可通过 `replica.lag.time.max.ms` 配置）从 Leader 那里获取过消息。
- 在过去的 10s 内从 Leader 那里获取过**最新**的消息。光从 Leader 那里获取消息是不够的，它还必须是几乎零延迟的。

如果跟 Follower 副本不能满足以上任何一点，比如与 Zookeeper 断开连接，或者在 10s 内没有请求获取任何消息，或者获取消息滞后了 10s 以上，那么它就被认为是不同步的。一个不同步的副本通过与 Zookeeper 重新建立连接，并从 Leader 那里获取最新消息，可以重新变成同步的。

> 如果一个或多个副本在同步和非同步状态之间快速切换，说明集群内部出现了问题，通常是 Java 不恰当的垃圾回收配置导致的。不恰当的垃圾回收配置会造成几秒钟的停顿，从而让 broker 与 Zookeeper 之间断开连接，最后变成不同步的，进而发生状态切换。

### 分区分配

现在，我们已经大概了解了 Kafka 的副本机制，那这些副本在 broker 中是怎么保存的呢？

我们的目标是要均衡 broker 的负载，由于数据的读写都是通过 Leader 副本，所以 Leader 副本需要均衡地分配到不同的 broker 上。这样做的好处不仅是负载均衡了，而且当某一台 broker 挂掉之后不至于所有 Leader 副本都不能进行读写，受影响的只有部分 Leader 副本。

举个例子，假设我们有 3 个 broker，每个 Topic 有三个分区，每个分区有 3 个副本。

那么为了实现均衡分配，我们可以先随机选择一个 broker（假设是 0 ），然后使用轮询的方式将分区 Leader 副本分配到 broker 上。于是，分区 0 的 Leader 副本会在 broker 0 上，分区 1 的 Leader 副本会在 broker 1 上，分区 2 的 Leader 副本会在 broker 3 上，以此类推。 

Leader 副本分配完之后，依次分配 Follower 副本。如果分区 0 的 Leader 在 broker 0 上，那么它的第一个 Follower 副本会在 broker 1 上，第二个 Follower 副本会在 broker 2 上。分区 1 的 Leader 在 broker 1 上，那么它的第一个 Follower 副本在 broker 2 上，第二个 Follower 副本在 broker 0 上，以此类推。

最终一个 Topic 三个分区，三个副本分配的结果如下图所示：

![Kafka Replication](/image/kafka-replica.png)

这也解析了上面说到的为什么 Kafka 不需要读写分离，因为负载已经均衡到每一个 broker 上了。

### 首选 Leader

我们试想一下上图的情景，假设 broker 2 挂了，那么 Leader 2 也就挂了，此时 Kafka 会从 broker 0 或者 broker 1 中选择一个 Follower 2 提升为 Leader。当 broker 2 再次启动起来之后，原来的 Leader 2 将变成 Follower 2，此时 broker 2 没有 Leader 副本了， broker 0 或者 broker 1 将会有两个 Leader，这样就会导致负载不均衡了。

为了解决上面的问题， Kafka 中引入**首选 Leader**的概念，也就是优先会成为 Leader 的副本。默认情况下，Kafka 的 `auto.leader.rebalance. enable` 被设为 true，它会检查首选 Leader 是不是当前 Leader ，如果不是，并且该副本是同步的，那么就会触发 Leader 选举，让首选 Leader 成为当前 Leader，让负载变得更均衡。

`auto.leader.rebalance. enable` 参数还需要配合以下两个参数一起使用：

- `leader.imbalance.check.interval.seconds`: 检查负载不均衡的时间间隔，默认 300 秒
- `leader.imbalance.per.broker.percentage`: 每个 broker 上 Leader 不均衡的比例超过多少才会触发选举首选 Leader，默认 10%

那么，哪个是首选 Leader？Kafka 在创建 Topic 时选定的 Leader 就是分区的首选 Leader，我们可以使用 `kafka.topics.sh` 工具查看副本和分区的详细信息，清单里的第一个副本一般就是首选 Leader。不管当前 Leader 是哪一个副本，都不会改变这个事实，即使使用副本分配工具将副本重新分配给其他 broker。如果我们需要手动进行副本分配，第一个指定的副本就是首选 Leader，所以要确保首选 Leader 被均衡地分配到各个 broker 上。

### 选举

前面提到，当一个分区的 Leader 挂了之后，会选举一个 Follower 提升为 Leader。当然，Kafka 会优先从同步的副本中选择一个提升为 Leader ，这样就不会丢失数据，这个选举就是 “完全”的。但如果在首领不可用时其他副本都是不同步的，我们该怎么办?

这个时候，我们需要根据业务情况作出一个权衡，因为：

- 如果不同步的副本不能被提升为新 Leader ，那么分区在旧 Leader (最后一个同步副本)恢复之前是不可用的。
- 如果不同步的副本可以被提升为新 Leader，那么在这个副本变为不同步之后写入旧  Leader 的消息会全部丢失，导致数据不一致。

也就是说，我们需要在可用性和一致性之间做一个决策。Kafka 中提供了 `unclean.leader.election.enable` 参数让我们配置，如果把 `unclean.leader.election.enable` 设为 true，就是允许不同步的副本成为 Leader (也就是“不完全的选举”)，那么我们将面临丢失消息的风险。如果把这个参数设为 false， 就要等待原先的 Leader 重新上线，从而降低了可用性。

### 最小同步副本

根据 Kafka 对可靠性保证的定义，消息只有在被写入到所有同步副本之后才被认为是已提交的，才能被客户端消费。但如果这里的“所有副本”只包含一个同步副本，那么在这个副本变为不可用时，数据就会丢失。

Kafka 提供了一个最小同步副本参数 `min.insync.replicas` 让我们配置。如果要确保已提交的数据被写入不止一个副本，就需要把最少同步副本数量设置为大一点的值。对于一个包含 3 个副本的 Topic ，如果 `min.insync.replicas` 被设为 2，那么至少要存在两个同步副本才能向分区写入数据。如果只有一个副本，那么 broker 就会停止接受生产者的请求，尝试发送数据的生产者会收到 `NotEnoughReplicasException` 异常。消费者仍然可以继续读取已有的数据。

### 总结

- Kafka 副本分为 Leader 和 Follower
- 同步副本集合称为 ISR ，只有被写进所有同步副本的消息才能被消费者读取到
- 分区需要均衡地分配到每个 broker 
- Kafka 的首选 Leader 是为了解决 Leader 迁移之后引发的负载不均衡问题
- 默认可以进行“不完全”选举，需要根据业务情况在可用性和一致性之间进行权衡
- 当同步副本数量小于最小同步副本时，生产者写入数据会收到异常信息，消费者仍可读取已有数据





