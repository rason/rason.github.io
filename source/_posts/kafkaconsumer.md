---
title: KafkaConsumer 基本知识
date: 2020-02-11 18:18:52
tags:
---

在开始阅读 Kafka Consumer 源码之前，先来温习一下 `KafkaConsumer` 的基本知识，本文主要翻译自官方文档。

### 偏移量和消费位置

- **偏移量**：Kafka 为分区中的每条记录维护着一个数字的偏移量，该偏移量充当着该记录在在分区中的唯一标识符，注意这是以分区为维度的。
- **消费位置**：表示的接下来将要读取的记录的偏移量。

例如，一个消费者的位置为5，表示已经消费了从偏移量从0到4之间的记录，下一个将要消费的是偏移量为5的记录。

### 消费者组和主题订阅

Kafka 中的消费者组相当于一个线程池的概念，组内的每个消费者相当于线程池中的一个线程。这些消费者可以运行在同一台机器上，也可以运行在不同的机器上，从而提供了扩展性和容错性。所有共享同一个 `group.id` 的消费者实例属于同一个消费者组。

消费者组中的每个消费者都可以通过 ` subscribe` 接口动态地订阅一系列的主题。一个主题中的一条消息只会被消费者组中的一个消费者消费，但是可以被不同消费者组中的多个消费者消费。这是因为一个主题的一个分区只能被一个消费者组的一个消费者消费，这句话可能有点绕。

例如：一个主题有 4 个分区，如果一个消费者组内有 4 个消费者，那么就是 1 个消费者消费 1 个分区；如果一个消费者组内只有 2 个消费者，那么就是 1 个消费者消费 2 个分区。

消费者组中的成员关系是被动态维护的：如果一个消费者挂了，该消费者被分配的分区将会被分配给消费组内其他的消费者。同样，如果有新的消费者加入该组，已有消费者的分区会被转移到新消费者，这叫做 **组内再平衡**。组内再平衡同样用于当新的分区被添加到一个被订阅的主题，或者当新创建的主题与订阅的正则表达式相匹配时。

从概念上讲，你可以认为一个消费者组是一个碰巧由多个进程组成的一个单个逻辑订阅者。作为一个多订阅者系统，Kafka 自然支持在不复制数据的情况下为给定主题拥有任意个消者组。

另外，当发生再平衡时，可以通过 `ConsumerRebalanceListener` 监听器来提醒消费者，完成应用程序级别的逻辑，比如状态清理，手动提交偏移量。

也可以使用 `assign(Collection)` 方法为消费者手动分配特定的分区，但是在这种情况下，动态分区分配和消费者组协作将被禁用。

### 检测消费者故障

在订阅了一组主题后，调用 `poll(Duration)` 方法时，消费者就会自动加入组。`poll` 方法还被设计成消费者保活。只要你持续的调用 `poll`，该消费者将会保持在组中，持续的从被分配的分区中接收到消息。在底层，该消费者会发送定期心跳信号给服务器。如果消费者挂了或者在 `session.timeout.ms` 时间内没有发送心跳信号，则该消费者将会被认为挂了，并且它的分区将会被重新分配。

消费者还可能会 "livelock" 的情况，即持续发送心跳，但没有调用 `poll` 去获取数据。在这种情况下，为了避免分区被消费者持续占据，Kafka 用 `max.poll.interval.ms` 配置提供了一个 "livelock" 发现机制。基本上，如果你在配置的 `max.poll.interval.ms` 内没有调用 `poll` 方法，该消费者就会主动的离开组，以便其它的消费者可以接管它的分区。当这种情况发生时，你可能会看到一个偏移量提交故障（通过调用 `commitSync()` 方法抛出的 `CommitFailedException` 异常提示），这是一个安全机制，即保证只有组内活跃的成员才能提交偏移量。因此，为了保持在组里，你必须持续的调用 `poll` 方法。

消费者提供两个配置项来控制 `poll` 循环：

- **max.poll.interval.ms**: 通过增加两个轮询的之间的时间间隔，可以留给消费者更多时间来处理 `poll(Duration)` 方法返回的记录。缺点是，增加此值可能会延迟组重新平衡，因为当超过这个时间还没有调用 `poll` 方法时消费者才会离开组，触发新的一轮再平衡。

- **max.poll.records**: 使用这个设置限定单次调用 `poll` 返回的记录总数。通过调整这个值，可以减少轮询间隔，这将减少组重新平衡的影响。

对于消息处理时间变化不可预测的情况，这些配置项可能还不够。推荐将消息处理转移到其他线程的方式来处理这些情况，但是必须采取一些措施来保证提交的偏移量不会超过实际位置。通常，我们需要禁用自动提交，然后在线程完成对记录的处理之后（取决于所需的传递语义），通过手动提交已处理记录的偏移量。还要注意，需要通过调用 `pause` 方法暂停分区，以便在线程处理完之前返回的记录之前，不会从 `poll` 方法接收新记录。


### 使用示例

#### 自动提交偏移量

下面是一个自动提交偏移量的 KafkaConsumer API 示例。

```
 Properties props = new Properties();
 props.put("bootstrap.servers", "localhost:9092");
 props.put("group.id", "test");
 props.put("enable.auto.commit", "true");
 props.put("auto.commit.interval.ms", "1000");
 props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
 props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
 KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
 consumer.subscribe(Arrays.asList("foo", "bar"));
 while (true) {
     ConsumerRecords<String, String> records = consumer.poll(100);
     for (ConsumerRecord<String, String> record : records)
         System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
 }
```

通过 `bootstrap.servers` 指定一个或多个 broker 的列表来建立与集群的连接。此列表仅用于发现群集中的其余 broker，不必是群集中所有服务器的列表，设置多个可以防止某些 broker 挂了获取不到元数据。

设置 `enable.auto.commit=ture` 开启自动提交偏移量，自动提交频率由 `auto.commit.interval.ms` 控制。

#### 手动控制偏移量

除了自动提交偏移量之外，用户也可以手动控制当记录真正被消费了才提交偏移量。当消息的消费与某些处理逻辑耦合时，这非常有用，因此在完成处理之前，不应将消息视为已消费。

```
 Properties props = new Properties();
 props.setProperty("bootstrap.servers", "localhost:9092");
 props.setProperty("group.id", "test");
 props.setProperty("enable.auto.commit", "false");
 props.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
 props.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
 KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
 consumer.subscribe(Arrays.asList("foo", "bar"));
 final int minBatchSize = 200;
 List<ConsumerRecord<String, String>> buffer = new ArrayList<>();
 while (true) {
     ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
     for (ConsumerRecord<String, String> record : records) {
         buffer.add(record);
     }
     if (buffer.size() >= minBatchSize) {
         insertIntoDb(buffer);
         consumer.commitSync();
         buffer.clear();
     }
 }
```

在这个例子中，当消费累积到 200 条之后批量插入数据库，然后才手工提交偏移量。如果像上个例子那样自动提交，还没插入数据库偏移量可能就已经被提交了，如果插入数据库失败，那么可能就丢失了这些数据。

不过，这个例子虽然用了手工提交，但是也有可能在插入数据库之后消费者挂了，偏移量没有被提交，那么消息可能会被重复消费。以这种方式使用，Kafka 提供了 **“至少一次”** 的交付保证，因为每个记录可能会交付一次，但在失败的情况下可能会重复。

**注意：使用自动提交偏移量也可以提供 “至少一次” 传递，但要求你必须在任何后续调用之前或在关闭消费者之前消费完调用 `poll(Duration)` 方法返回的所有数据。否则，提交的偏移量可能会超过消费的位置，从而导致丢失记录。使用手动提交偏移的好处是，你可以直接控制什么时候将记录视为 “已消费”。**

上面的示例使用 `commitSync` 会将所有接收到的记录标记为已提交。在某些情况下，你可能希望通过显式指定偏移量来更好地控制提交了哪些记录。在下面的示例中，我们在处理完每个分区中的记录后提交偏移量。

```
 try {
     while(running) {
         ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(Long.MAX_VALUE));
         for (TopicPartition partition : records.partitions()) {
             List<ConsumerRecord<String, String>> partitionRecords = records.records(partition);
             for (ConsumerRecord<String, String> record : partitionRecords) {
                 System.out.println(record.offset() + ": " + record.value());
             }
             long lastOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
             consumer.commitSync(Collections.singletonMap(partition, new OffsetAndMetadata(lastOffset + 1)));
         }
     }
 } finally {
   consumer.close();
 }
```

**注意：提交的偏移量应该始终是应用程序将读取的下一条消息的偏移量。** 因此，在调用 `commitSync（offset）` 时，offset 的值应该是最后一条消息的偏移量加1。

#### 手动分配分区

在前面的示例中，我们订阅了我们感兴趣的主题，同时让 Kafka 基于消费者组中的活跃消费者动态均衡地分配这些主题的分区。然而，某些时候你可能需要更精确的控制已被分配的特定分区。例如：

- 如果进程维护与该分区相关联的某种本地状态（如本地磁盘上的键值存储），那么它应该只获取它在磁盘上维护的分区的记录。
- 如果该进程自身就是高可用，且失败时将会重启。在这种情况下，Kafka 没有必要检测失败和重新分配分区，因为消费进程将会在另一台机器上重启。

要使用此模式，你只需调用 `assign（Collection）`，其中包含要使用的分区的完整列表，而不是使用 `subscribe` 订阅主题。

```
 String topic = "foo";
 TopicPartition partition0 = new TopicPartition(topic, 0);
 TopicPartition partition1 = new TopicPartition(topic, 1);
 consumer.assign(Arrays.asList(partition0, partition1));
```

分配后，就像前面的示例中一样，在循环中调用 `poll` 来消费记录。消费者指定的组仍然用于提交偏移量，但是现在分区集只会在再一次调用 `assign` 时才会改变。手动分区分配不使用组协调，因此消费者挂了不会导致分配的分区重新平衡。每个消费都是独立的，即使它与其他消费者共享一个 groupId 。为了避免偏移量提交冲突，通常应该确保每个消费者实例的 groupId 是唯一的。

注意，不可能将手动分区分配（即使用 `assign` )与通过主题订阅（即使用 `subscribe` )动态分区分配混合使用。

#### 在Kafka之外存储偏移量

我们不是一定要使用 Kafka 内置的偏移量管理，也可以自行选择外部存储来管理偏移量。主要的用途是可以在同一个系统中保存偏移量和消费结果，即以原子方式保存结果和偏移量，并提供 “恰好一次” 语义。

比如： 消费的结果保存在一个关系型数据库，同时在数据库里保存偏移量，那么就可以在同一个事务中提交结果和偏移量。这样的话，要么事务提交成功同时保存消费的内容和更新偏移量，要么不存储结果也不更新偏移量。

管理自己的偏移量，你只需要执行下面的操作：

- 配置 `enable.auto.commit=false`
- 使用每个 `ConsumerRecord` 提供的偏移量来保存你的位置
- 重启时使用 `seek(TopicPartition,long)` 方法来恢复消费者的位置

当分区分配也是手动完成时，这种方式的使用还算简单。如果分区分配是自动完成，需要特别注意处理分区分配改变的情况。可以通过在对 `subscribe(Collection, ConsumerRebalanceListener)` 和 `subscribe(Pattern, ConsumerRebalanceListener)` 的调用中提供一个 `ConsumerRebalanceListener` 实例来完成。

例如，当分区被释放时，消费者可以通过 `ConsumerRebalanceListener.onPartitionsRevoked(Collection)` 来提交这些分区的偏移量。当分区被分配给一个消费者，可以通过 `ConsumerRebalanceListener.onPartitionsAssigned(Collection)` 查询新分区的偏移量，进行消费位置的初始化。 

#### 控制消费者的位置

在大多数用例中，消费者只是从开始到结束消费记录，定期提交它的位置（不论是自动还是手动）。然而，Kafka 允许消费者手动控制它的位置，随意在分区内向前或向后移动位置。这意味着消费者可以重复消费较旧的记录，或者跳过到最近的记录而不消费中间的记录。

有几种情况可能需要手动控制消费位置：

- 一种情况是对时间敏感的记录处理，对于远远落后的记录，消费者其实是不希望一条条追赶上处理所有的记录，而是直接跳到最近的记录。
- 另一个用例是用于如前一节所述维护本地状态的系统。在这样的系统中，消费希望在启动时将它的位置初始化到被保存在本地存储的位置。同样，如果本地状态被销毁，该状态可能通过重新消费所有数据来重新创建状态。

Kafka 可以通过 `seek(TopicPartition,offset)` 为指定分区位置。找到服务器维护的最早和最新偏移的特殊方法可以用 `seekToBegining(Collection)` 和 `seekToEnd(Collection)`。

#### 消费流量控制

如果一个消费者从多个已分配的分区中获取数据，它将会尝试同时从所有分区中消费，从而效地为这些分区提供相同优先级以供消费。然而，在某些情况下消费者可能希望首先全速从其中一些分区中获取数据，只有当这些分区只有很少或者没有数据消费时才开始从其它的分区中获取数据。

Kafka 支持使用 `pause(Collection)` 和 `resume(Collection)` 动态控制消费流，以便在后续的 ` poll(Duration)` 调用中分别暂停指定分区上的消费和恢复指定暂停分区上的消费。

#### 多线程处理

Kafka 的消费者不是线程安全的。需要用户自己确保多线程之间的同步访问。非同步的访问会抛出 `ConcurrentModificationException`。

多线程处理有两种常用方案：

##### 方案1

一个消费者一个线程

优点：

- 实现简单，比较符合目前使用 Consumer API 的习惯
- 多个线程之间没有任何交互，省去了很多保障线程安全方面的开销
- Kafka主题中的每个分区都能保证只被一个线程处理，容易实现分区内的消息消费顺序

缺点：

- 每个线程都维护自己的 `KafkaConsumer` 实例，必然会占用更多的系统资源，如内存、TCP连接等
- 能使用的线程数受限于Consumer订阅主题的总分区数
- 每个线程完整地执行消息获取和消息处理逻辑，一旦消息处理逻辑很重，消息处理速度很慢，很容易出现不必要的Rebalance，引发整个消费者组的消费停滞

##### 方案2

单个消费者或多个消费者 + 多个任务处理线程，即将消息获取和消息处理分离：

- 获取消息的线程可以是一个，也可以是多个，每个线程维护专属的KafkaConsumer实例
- 处理消息则由特定的线程池来做，从而实现消息获取和消息处理的真正解耦

优点：

- 把任务切分成消息获取和消息处理两部分，分别由不同的线程来处理
- 相对于方案1，方案2最大的优势是它的高伸缩性
- 可以独立地调节消息获取的线程数，以及消息处理的线程数，不必考虑两者之间是否相互影响

缺点：

- 实现难度大，因为要分别管理两组线程
- 消息获取和消息处理解耦，无法保证分区内的消费顺序
- 两组线程，使得整个消息消费链路被拉长，最终导致正确位移提交会变得异常困难，可能会出现消息的重复消费