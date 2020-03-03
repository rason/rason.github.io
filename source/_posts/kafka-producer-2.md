---
title: Kafka producer 源码阅读（二）
date: 2020-02-06 22:04:14
tags:
---

这是阅读 Kafka producer 源码的第二篇文章，上篇文章最后提到了 Sender 线程主循环的 `sendProducerData` 方法并没有真正进行网络 IO 操作，真正的 IO 操作是在 `poll` 方法中完成的，这部分代码如果不了解 Java NIO 部分的知识是比较难理解的。

注意：Kafka 也封装了一个 Selector，全路径是 `org.apache.kafka.common.network.Selector`，而 Java NIO 的 Selector 全路径是 `java.nio.channels.Selector`。因为本文主要是分析 Kafka producer 源码，所以用 Selector 代表 Kafka 封装的，Java NIO 的用 nioSelector 来表示。

在开始分析 Kafka producer 源码之前，我们先来简单复习一下 nioSelector。

### nioSelector

先看一个 nioSelector 客户端的例子：

```
Selector nioSelector = Selector.open()
SocketChannel socketChannel = SocketChannel.open();
socketChannel.connect(new InetSocketAddress("localhost", 8089));
SelectionKey key = socketChannel.register(nioSelector, SelectionKey.OP_CONNECT);
```

主要是干了三件事情：

1. 打开一个多路复用选择器，也就是 nioSelector
2. 打开一个 SocketChannel，建立连接
3. 将 socketChannel 注册到 nioSelector，对该 socketChannel 感兴趣的事件是 SelectionKey.OP_CONNECT

简单来说，nioSelector 就是用于监视我们所注册的所有 Channel，当 Channel 上发生注册时感兴趣的事情时，nioSelector 通知应用程序处理请求。一个 nioSelector 实例可以监视多个 Socket Channel，从而达到监视更多连接的作用。如下图所示：

![NIO Selector](/image/nioSelector.png)

将 Channel 注册到 nioSelector ，会为每个 Channel 返回 `SelectionKey`。SelectionKey 是一个标识 Channel 的对象，它包含有关 Channel 状态的信息（例如准备接受请求），我们可以通过 SelectionKey 随时修改在该 Channel 上感兴趣的事件。

简单的复习之后，我们开始探讨 Kafka producer 网络方面的源码。

### 发起建立连接

要发送网络请求，首先得建立连接，所以我们先从建立连接的部分代码开始看，建立连接的方法是 `org.apache.kafka.clients.NetworkClient#initiateConnect`。

```
// node 为 kafka broker 节点
private void initiateConnect(Node node, long now) {
    String nodeConnectionId = node.idString();
    // 设置节点的连接状态为 ConnectionState.CONNECTING
    connectionStates.connecting(nodeConnectionId, now, node.host(), clientDnsLookup);
    // 节点的地址
    InetAddress address = connectionStates.currentAddress(nodeConnectionId);
    log.debug("Initiating connection to node {} using address {}", node, address);
    // 调用 Selector 的连接方法
    selector.connect(nodeConnectionId,
            new InetSocketAddress(address, node.port()),
            this.socketSendBuffer,
            this.socketReceiveBuffer);

}
```

上面这段代码比较简单，就是初始化连接状态、获取节点地址然后调用 `org.apache.kafka.common.network.Selector#connect` 方法建立连接。

```
    public void connect(String id, InetSocketAddress address, int sendBufferSize, int receiveBufferSize) throws IOException {
    	// 确保到该 broker 节点的 channel 还没有注册
        ensureNotRegistered(id);
        // 打开一个 SocketChannel
        SocketChannel socketChannel = SocketChannel.open();
        SelectionKey key = null;
        // 配置 socketChannel 相关属性，如 none block, keepalive 和收发 buffer 大小等
        configureSocketChannel(socketChannel, sendBufferSize, receiveBufferSize);
        // 建立连接
        boolean connected = doConnect(socketChannel, address);
        // 注册通过感兴趣的事件
        key = registerChannel(id, socketChannel, SelectionKey.OP_CONNECT);
        // 正常来说不会立马就成功建立连接，除非是本地客户端服务器
        if (connected) {
            // OP_CONNECT won't trigger for immediately connected channels
            log.debug("Immediately connected to node {}", id);
            immediatelyConnectedKeys.add(key);
            key.interestOps(0);
        }
     
    }
```

这段代码包含了一些信息量：

1. ensureNotRegistered 是通过节点 id 查询是否已经有注册的 channel，可以看出一个 broker 只会建立一个 SocketChannel
2. doConnect 方法调用的是 `java.nio.channels.SocketChannel#connect`，我们配置了 non-blocking 模式并不会立马成功建立连接，除非是本地客户端服务端，所以大部分情况都应该返回 false，稍后必须通过调用 `java.nio.channels.SocketChannel#finishConnect` 方法完成连接操作
3. registerChannel 注册通过感兴趣的建立连接事件，用意就是后面调用 `org.apache.kafka.common.network.Selector#poll` 方法时监控 Channel 可连接时调用 `java.nio.channels.SocketChannel#finishConnect` 方法完成连接

registerChannel 方法还有其他重要信息，我们展开看一下：

```
protected SelectionKey registerChannel(String id, SocketChannel socketChannel, int interestedOps) throws IOException {
	// 将 socketChannel 注册到 nioSelector 上
    SelectionKey key = socketChannel.register(nioSelector, interestedOps);
    // 将 KafkaChannel 附加到注册返回的 SelectionKey 上
    KafkaChannel channel = buildAndAttachKafkaChannel(socketChannel, id, key);
    // 节点 ID 与 channel 的映射关系，也就是一个 broker 节点一个 channel
    this.channels.put(id, channel);
    if (idleExpiryManager != null)
    	// LRU 维护节点连接，闲置的连接可能会被关闭
        idleExpiryManager.update(channel.id(), time.nanoseconds());
    return key;
}
```

这里的重点是 `buildAndAttachKafkaChannel`，Kafka 不仅自己封装了一个 Selector ,还封装了一个 Channel，也就是 `KafkaChannel`。我们看下这个方法的内容：

```
private KafkaChannel buildAndAttachKafkaChannel(SocketChannel socketChannel, String id, SelectionKey key) throws IOException {
	// 创建一个 KafkaChannel
    KafkaChannel channel = channelBuilder.buildChannel(id, key, maxReceiveSize, memoryPool);
    // 将 KafkaChannel 附加到注册返回的 SelectionKey 上
    key.attach(channel);
    return channel;
  
}
```

逻辑也比较简单，先创建一个 KafkaChannel 然后附加到 SelectionKey 上。 这里的 channelBuilder 根据配置会有不同的实现类：

- `PlaintextChannelBuilder`： 明文
- `SslChannelBuilder`：SSL 加密 
- `SaslChannelBuilder`：简单认证

我们以 PlaintextChannelBuilder 为例看一下：

```
public KafkaChannel buildChannel(String id, SelectionKey key, int maxReceiveSize, MemoryPool memoryPool) throws KafkaException {
    PlaintextTransportLayer transportLayer = new PlaintextTransportLayer(key);
    Supplier<Authenticator> authenticatorCreator = () -> new PlaintextAuthenticator(configs, transportLayer, listenerName);
    return new KafkaChannel(id, transportLayer, authenticatorCreator, maxReceiveSize,
            memoryPool != null ? memoryPool : MemoryPool.NONE);
  
}

public class PlaintextTransportLayer implements TransportLayer {
    private final SelectionKey key;
    private final SocketChannel socketChannel;
    private final Principal principal = KafkaPrincipal.ANONYMOUS;

    public PlaintextTransportLayer(SelectionKey key) throws IOException {
        this.key = key;
        this.socketChannel = (SocketChannel) key.channel();
    }
}
```

Kafka 包装了一个传输层，里面包含了之前注册返回的 `SelectionKey` 和注册的 `SocketChannel`，这对我们理解后面的逻辑很关键。

到这里，发起建立连接请求的代码分析的差不多了，但是真正调用 `java.nio.channels.SocketChannel#finishConnect` 完成连接的建立部分代码还没分析。现在，先让我用一张图来梳理一下目前涉及到的几个核心类的关系：

![NIO Selector](/image/kafka-producer-network.png)

### 完成连接的建立

之前已经说过，真正完成连接的建立是在 Sender 线程主循环调用 `org.apache.kafka.clients.NetworkClient#poll` 方法。

```
public List<ClientResponse> poll(long timeout, long now) {
        ensureActive();

        if (!abortedSends.isEmpty()) {
            // 如果由于不支持的版本异常或断开连接而中止发送，需要立即处理
            List<ClientResponse> responses = new ArrayList<>();
            handleAbortedSends(responses);
            completeResponses(responses);
            return responses;
        }

        // 看是否需要先更新集群的元数据
        long metadataTimeout = metadataUpdater.maybeUpdate(now);
        try {
        	// 这是我们要核心关注的，调用 org.apache.kafka.common.network.Selector#poll 方法
            this.selector.poll(Utils.min(timeout, metadataTimeout, defaultRequestTimeoutMs));
        } catch (IOException e) {
            log.error("Unexpected error during I/O", e);
        }

        // 处理完成的动作
        long updatedNow = this.time.milliseconds();
        List<ClientResponse> responses = new ArrayList<>();
        handleCompletedSends(responses, updatedNow);
        handleCompletedReceives(responses, updatedNow);
        handleDisconnections(responses, updatedNow);
        handleConnections();
        handleInitiateApiVersionRequests(updatedNow);
        handleTimedOutRequests(responses, updatedNow);
        completeResponses(responses);

        return responses;
    }
```

这段代码的思路很清晰，就是先调用 `org.apache.kafka.common.network.Selector#poll` 方法进行 IO 读写、连接建立等，然后处理完成的动作。我们来看一下删减版的 `org.apache.kafka.common.network.Selector#poll` 代码，主要关注是怎么完成连接的建立的。

```
public void poll(long timeout) throws IOException {
	int numReadyKeys = select(timeout);
	if (numReadyKeys > 0 || !immediatelyConnectedKeys.isEmpty() || dataInBuffers) {
        Set<SelectionKey> readyKeys = this.nioSelector.selectedKeys();
        // 处理有 IO 事件准备好的 SelectionKey
        pollSelectionKeys(readyKeys, false, endSelect);
        readyKeys.clear();
    } else {
        madeReadProgressLastPoll = true; //no work is also "progress"
    }
}

// 检查 Channel 是否有数据，阻塞到给定的超时时间
private int select(long timeoutMs) throws IOException {
    if (timeoutMs == 0L)
        return this.nioSelector.selectNow();
    else
        return this.nioSelector.select(timeoutMs);
}
```

这里就是 Java NIO 的知识了，通过 nioSelector 检查注册的通过是否有数据可以进行操作，获取相应的 SelectionKey 集合然后进行处理。

```
void pollSelectionKeys(Set<SelectionKey> selectionKeys,
                       boolean isImmediatelyConnected,
                       long currentTimeNanos) {
    // 循序 SelectionKey 进行处理                  	
	for (SelectionKey key : determineHandlingOrder(selectionKeys)) {
		KafkaChannel channel = channel(key);

		// 处理已经完成握手的所有连接
        if (isImmediatelyConnected || key.isConnectable()) {
        	// 这里就是我们要找的代码了，正式完成连接的建立
            if (channel.finishConnect()) {
            	// 将节点 ID 添加到已经完成连接的集合 
                this.connected.add(channel.id());
            } else {
                continue;
            }
        }

        // 如果 Channel 已准备就绪，并且有字节可从 Socket 或 Buffer 读取，并且之前没有已暂存或正在进行的接收，则从 Channel 读取
        if (channel.ready() && (key.isReadable() || channel.hasBytesBuffered()) && !hasStagedReceive(channel)
            && !explicitlyMutedChannels.contains(channel)) {
            NetworkReceive networkReceive;
            while ((networkReceive = channel.read()) != null) {
                madeReadProgressLastPoll = true;
                addToStagedReceives(channel, networkReceive);
            }
        }

        // 如果 Channel 已准备就绪，并且可以写了，则向 Channel 写入
        if (channel.ready() && key.isWritable() && !channel.maybeBeginClientReauthentication(
                    () -> channelStartTimeNanos != 0 ? channelStartTimeNanos : currentTimeNanos)) {
            Send send  = channel.write();
            ...
        }
	}
}

// 这里就是之前将 KafkaChannel 附加到 SelectionKey 的作用了，会根据 SelectionKey 获取出 KafkaChannel
private KafkaChannel channel(SelectionKey key) {
    return (KafkaChannel) key.attachment();
}
```

以上，算是标准的 Java NIO 使用姿势了，有啥事件进行相应的处理。对于我们前面发起的建立连接时注册的 `SelectionKey.OP_CONNECT` 事件，轮询到 `key.isConnectable()` ，这里就调用 `channel.finishConnect()` 完成连接的建立。

相信大家现在应该可以举一反三，对于读写操作不用继续分析。

### 总结

Kafka producer 网络部分的源码核心就是 Java NIO ，当我们把 NIO 理解透了，看起来应该就比较简单了。




