---
title: Kafka producer 源码阅读（一）
date: 2020-02-04 18:01:51
tags:
---

最近花了几天时间阅读了 Kafka producer 的源码，内部的细节问题还是挺多的，还没有一行行细扣，只是看了下主要的脉络，尝试梳理一下其中的逻辑。

主要流程涉及以下几个核心类：

- KafkaProducer
- RecordAccumulator
- Sender
- NetworkClient
- org.apache.kafka.common.network.Selector
- KafkaChannel

核心链路可以简单概括为四点：

1. 通过 `KafkaProducer` 发送的消息会先到 `RecordAccumulator`，RecordAccumulator 是一个消息累积器，我们知道 Kafka 消息是可以批量发送的
2. `RecordAccumulator` 中的消息批次满了或者新建了一个批次会唤醒 `Sender` 线程对消息进行发送，当然 `linger.ms` 时间到了也会对消息进行发送
3. `Sender` 线程通过 `NetworkClient` 构造一个发送请求，然后调用 `org.apache.kafka.common.network.Selector` 的 `send` 方法注册感兴趣的写事件
4. 随后 `NetworkClient` 的 `poll` 方法会调用 `org.apache.kafka.common.network.Selector` 的 `poll` 方法把请求真正发出去


### KafkaProducer

KafkaProducer 是将消息发布到 Kafka 群集的 Kafka 客户端。producer 是线程安全的，跨线程共享单个 producer 实例通常比拥有多个实例更快。因此，我们使用 Springboot 集成 Kafka 的时候，默认只会创建一个 KafkaProducer 实例。

当我们通过 `org.springframework.kafka.core.KafkaTemplate#send` 发送消息时，实际会调用到的就是 `org.apache.kafka.clients.producer.KafkaProducer#doSend` 方法。

```
private Future<RecordMetadata> doSend(ProducerRecord<K, V> record, Callback callback) {
        TopicPartition tp = null;

        throwIfProducerClosed();
        // 首选确保 topic 的元数据可用
        ClusterAndWaitTime clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);
        
        long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
        Cluster cluster = clusterAndWaitTime.cluster;

        // key 和 record 序列化
        byte[] serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
        byte[] serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
       
        // 根据发送的 record 计算发送到哪个分区
        int partition = partition(record, serializedKey, serializedValue, cluster);
        tp = new TopicPartition(record.topic(), partition);


        // 确保消息不能太大
        int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(apiVersions.maxUsableProduceMagic(),
                compressionType, serializedKey, serializedValue, headers);
        ensureValidRecordSize(serializedSize);

        // 将消息添加到消息累积器
        RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
                serializedValue, headers, interceptCallback, remainingWaitMs);

        // 如果批次满了或者新建了一个批次，就会唤醒 sender 线程
        if (result.batchIsFull || result.newBatchCreated) {
            this.sender.wakeup();
        }
        return result.future;
    }
```

代码我精简了部分内容，可以看出发送方法主要就是干了以下几件事情：

1. 确保 topic 的元数据可用，如果元数据没有的话会先发起元数据请求，因为 KafkaProducer 需要知道一些元数据才能知道发送到哪个 broker 和哪个分区
2. 对分区 key 和消息 record 进行序列化，我们可以通过 `key.serializer` 和 `value.serializer` 两个配置项自定义序列化方式
3. 计算发送到 topic 的哪个分区，默认情况下如果没有指定分区 key，则会通过轮询的方式负载均衡发送到分区，如果指定了分区 key，则根据 key 的哈希对分区数取模计算出分区
4. 确保消息的大小不能大于 `max.request.size` 和 `buffer.memory` 配置的值
5. 将消息添加到记录累积器，这一步是整个方法的核心，在 Kafka 中消息的发送是按批次发送的，当然一个批次可以只有一条消息，这一步骤我们下面展开分析
6. 如果批次满了或者新建了一个批次，就会唤醒 sender 线程，sender 线程会对消息进行发送

另外，我们留意到 accumulator.append 方法还传入了一个 remainingWaitMs 参数，这是因为 Kafka 提供了一个 `max.block.ms` 参数来让我们控制 `KafkaProducer.send()` 和 `KafkaProducer.partitionsFor()` 方法最长的阻塞时间。

### RecordAccumulator

RecordAccumulator 相当于一个队列，将要发送的记录累积要发送到服务器的 `MemoryRecords` 实例中。RecordAccumulator 是有内存限制的，当内存不够用的时候，append 就会被阻塞，这就是为什么会有 `max.block.ms` 这个配置项。

在开始看代码之前，我们先了解一个 RecordAccumulator 保存消息的数据结构：

```
ConcurrentMap<TopicPartition, Deque<ProducerBatch>> batches;
```

也就是说，每个主题的每个分区，都会对应一个双端队列，而队列里面保存的是 ProducerBatch ，ProducerBatch 就是保存一个批次消息的对象。因为 Kafka broker 中的消息是按分区来存储的，所有只有同一个分区的消息才会打成一个批次，这个应该很好理解。

现在，我们来看一下 `RecordAccumulator.append` 方法： 

```
public RecordAppendResult append(TopicPartition tp,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Header[] headers,
                                     Callback callback,
                                     long maxTimeToBlock) throws InterruptedException {
        ByteBuffer buffer = null;
        if (headers == null) headers = Record.EMPTY_HEADERS;
        try {
            // 检查我们是否有在进行中的 batch ，如果有就添加到一个 ProducerBatch 中
            Deque<ProducerBatch> dq = getOrCreateDeque(tp);
            synchronized (dq) {
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
                if (appendResult != null)
                    return appendResult;
            }

            // 如果没有进行中的 record batch ，那么就创建一个新的 batch，开辟内存的大小为 batch.size 和本条记录的最大值
            byte maxUsableMagic = apiVersions.maxUsableProduceMagic();
            int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
            buffer = free.allocate(size, maxTimeToBlock);

            synchronized (dq) {
				// 由于 dq 是同步的，所以在多线程并发的时候，其他线程可能已经帮我们创建出一个新的 ProducerBatch，所以进入同步代码块之后需要再次尝试 append
                RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
                if (appendResult != null) {
                    return appendResult;
                }

                // 创建新的 ProducerBatch 并将消息添加到该 ProducerBatch 中
                MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
                ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());
                FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, headers, callback, time.milliseconds()));

                // 将 ProducerBatch 添加到 Deque 中
                dq.addLast(batch);
                incomplete.add(batch);

                // Don't deallocate this buffer in the finally block as it's being used in the record batch
                buffer = null;
                return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true);
            }
        } finally {
            if (buffer != null)
                free.deallocate(buffer);
        }
    }
```

主要逻辑应该还是比较清晰的：

1. 首先获取到该 `TopicPartition` 对应的 `Deque<ProducerBatch>` ，然后 `tryAppend` 方法中会取出 Deque<ProducerBatch> 中最后一个 `ProducerBatch`，如果不为空则尝试添加到该 ProducerBatch 中，如果为空则继续往下走
2. 没有可用 ProducerBatch ，则创建一个新的，开辟内存的大小为 `batch.size` 和本条记录的最大值
3. 同步代码块中会再次调用 `tryAppend` 方法，是因为在多线程并发的时候，其他线程可能已经帮我们创建出一个新的 ProducerBatch
4. 创建新的 ProducerBatch 并将消息添加到该 ProducerBatch 中，消息实际是通过 `MemoryRecordsBuilder` 写到前面开辟的 `buffer` 中，其实就是写内存
5. 返回 append 的结果，注意这个结果很重要，因为上面的 KafkaProducer 就是根据这个返回结果判断是否需要唤醒 sender 线程

### Sender

Sender 是向 Kafka 集群发送生产请求的后台线程。该线程会发出元数据请求以更新其群集元数据，然后将产生的请求发送到适当的节点。跟踪代码会发现，Sender 线程是在 KafkaProducer 构造方法中启动的。

我们来看一下 Sender 线程的 `run` 方法:

```
public void run() {
    // 主循环，直接 close 方法被调用
    while (running) {
        try {
            runOnce();
        } catch (Exception e) {
            log.error("Uncaught error in kafka producer I/O thread: ", e);
        }
    }
}

void runOnce() {
    long currentTimeMs = time.milliseconds();
    long pollTimeout = sendProducerData(currentTimeMs);
    client.poll(pollTimeout, currentTimeMs);
}
```

简单的几行代码，包含的信息量其实是很大的：

1. Sender 是不断地循环的，直到 close 方法被调用，意思就是不断地循环检查有没有数据需要发送
2. `sendProducerData` 看名字就知道是发送数据
3. `poll` 方法其实才是真正发生 socket 读写的地方，所以 sendProducerData 其实相当于构造请求，注册写事件

接着看一下 sendProducerData 方法：

```
    private long sendProducerData(long now) {
        Cluster cluster = metadata.fetch();
        // 获取其分区准备好发送的节点列表
        RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

        // 如果有哪个分区的 Leader 分区未知，则强制元数据更新
        if (!result.unknownLeaderTopics.isEmpty()) {
            for (String topic : result.unknownLeaderTopics)
                this.metadata.add(topic);
            this.metadata.requestUpdate();
        }

        // 移出掉还没有创建好连接的节点
        Iterator<Node> iter = result.readyNodes.iterator();
        long notReadyTimeout = Long.MAX_VALUE;
        while (iter.hasNext()) {
            Node node = iter.next();
            if (!this.client.ready(node, now)) {
                iter.remove();
                notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));
            }
        }

        // 以节点为维度整理出需要发送的 ProducerBatch 列表
        Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes, this.maxRequestSize, now);
        addToInflightBatches(batches);

        // 单个分区顺序性保证
        if (guaranteeMessageOrder) {
            for (List<ProducerBatch> batchList : batches.values()) {
                for (ProducerBatch batch : batchList)
                    this.accumulator.mutePartition(batch.topicPartition);
            }
        }

        accumulator.resetNextBatchExpiryTime();
        List<ProducerBatch> expiredInflightBatches = getExpiredInflightBatches(now);
        List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(now);
        expiredBatches.addAll(expiredInflightBatches);

   
        if (!expiredBatches.isEmpty())
            log.trace("Expired {} batches in accumulator", expiredBatches.size());
        for (ProducerBatch expiredBatch : expiredBatches) {
            String errorMessage = "Expiring " + expiredBatch.recordCount + " record(s) for " + expiredBatch.topicPartition
                + ":" + (now - expiredBatch.createdMs) + " ms has passed since batch creation";
            failBatch(expiredBatch, -1, NO_TIMESTAMP, new TimeoutException(errorMessage), false);
            if (transactionManager != null && expiredBatch.inRetry()) {
                transactionManager.markSequenceUnresolved(expiredBatch.topicPartition);
            }
        }
        sensors.updateProduceRequestMetrics(batches);

        // 如果有任何节点准备发送+具有可发送数据，会使用0超时进行轮询，这样可以立即循环并尝试发送更多数据。
        // 否则，超时将是下一批到期时间与检查数据可用性的延迟时间之间的较小值。
        long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
        pollTimeout = Math.min(pollTimeout, this.accumulator.nextExpiryTimeMs() - now);
        pollTimeout = Math.max(pollTimeout, 0);
        if (!result.readyNodes.isEmpty()) {
            pollTimeout = 0;
        }

        // 发送生产请求
        sendProduceRequests(batches, now);
        return pollTimeout;
    }
```

主要逻辑在代码上都有注解，需要解析的是返回值 `pollTimeout`。这个返回值会用于主循环的 `client.poll` 方法中，代表 poll 阻塞的时间。

- 如果某个分区已经准备好发送，则阻塞时间为0；
- 如果某个分区已经积累了一些数据但尚未准备好，则阻塞时间为“现在”与其“延迟到期时间”的时差；
- 否则，阻塞为“现在”与“元数据到期时间”的时差；

所以，当我们设置了 `linger.ms` 的值时，不需要等到批次满了也会根据这个延迟时间来进行发送。

接着我们看下 `sendProduceRequest` 方法：
```
private void sendProduceRequest(long now, int destination, short acks, int timeout, List<ProducerBatch> batches) {
        ...
        ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0,
                requestTimeoutMs, callback);
        client.send(clientRequest, now);
        log.trace("Sent produce request to {}: {}", nodeId, requestBuilder);
    }
```

简单来讲，就是构造一个客户端请求 `ClientRequest` ，然后调用客户端的 send 方法（org.apache.kafka.clients.KafkaClient#send）进行发送。

### NetworkClient

`NetworkClient` 是 `KafkaClient` 的实现类，也就是说上面的 send 方法实际调用的就是 NetworkClient 的 send 方法。

NetworkClient 用于异步请求/响应网络IO的网络客户端，Kafka 的生产者和消费者就是通过这个类来进行网络操作。

```
    private void doSend(ClientRequest clientRequest, boolean isInternalRequest, long now, AbstractRequest request) {
        String destination = clientRequest.destination();
        RequestHeader header = clientRequest.makeHeader(request.version());
        Send send = request.toSend(destination, header);
        InFlightRequest inFlightRequest = new InFlightRequest(
                clientRequest,
                header,
                isInternalRequest,
                request,
                send,
                now);
        this.inFlightRequests.add(inFlightRequest);
        selector.send(send);
    }
```

该方法比较简单，构造一个发送到目标 broker 数据的 Send 实体，然后调用 `org.apache.kafka.common.network.Selector#send` 方法。

### Selector

Selector 其实是包装了 Java NIO, 是一个 nioSelector 接口，用于执行无阻塞的多连接网络I/O。与  `NetworkSend` 和  `NetworkReceive` 协同工作，以传输网络请求和响应。

我们先看了解一下其用法：

通过执行以下操作，可以创建一个连接添加到 nioSelector

```
selector.connect(42, new InetSocketAddress("google.com", server.port), 64000, 64000);
```

`connect` 方法的调用不会阻塞在创建TCP连接，因此 connect 方法仅开始启动连接。成功调用此方法并不意味着已建立有效连接。发送请求、接收响应、处理连接完成以及现有连接上的断开都是使用 `poll()` 调用完成的。

比如：

```
nioSelector.send(new NetworkSend(myDestination, myBytes));
nioSelector.send(new NetworkSend(myOtherDestination, myOtherBytes));
nioSelector.poll(TIMEOUT_MS);
```

了解 Selector 的用法之后，我们再看回 Sender 主循环的代码应该就是茅塞顿开的感觉：

```
void runOnce() {
    long currentTimeMs = time.milliseconds();
    long pollTimeout = sendProducerData(currentTimeMs);
    client.poll(pollTimeout, currentTimeMs);
}
```

由于篇幅关系，到这里 poll 方法就暂不展开分析了。

### 总结

我好像把总结写在文章开头了。
