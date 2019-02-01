---
title: RabbitMQ-Three-PublishSubscribe
date: 2017-03-13 08:55:00
categories: AMQP
tags: RabbitMQ 
description: RabbitMQ教程
---

## 发布/订阅

在上一篇教程中我们创建了一个工作队列。工作队列背后的假设是每个任务被交付给正好一个工作者进程。在今天的教程中我们将做一些完全不同的事情——将一个消息交付给多个消费者。这个模式成为**发布/订阅**。

为了演示这个模式，我们将创建一个简单的日志系统。它将包含两部分程序——第一部分将会发出日志信息而第二部分将会接收并打印出来。

在我们的日志系统中，接收程序的每个运行副本都将获得消息。这样的话我们就可以运行一个接收者直接记录日志到硬盘，与此同时我们将运行另一个接收者并在屏幕上查看日志。

本质上，发布的日志消息将被广播到所有接收者。

## Exchanges

在之前的教程中我们发送消息的队列和从队列接收消息。现在是时候在Rabbit中引入完整的消息模型。

先让我们来快速复习一下之前教程中出现的模型：

- **producer**是发送消息的用户程序。
- **queue**是保存消息的缓冲区。
- **consumer**是接收消息的用户程序。

RabbitMQ的消息传递模型的核心思想是，生产者从不会将任何消息直接发送到队列。实际上，生产者经常甚至不知道消息是否将被传递到任何队列。

相反，生产者只能向exchange发送消息。exchange是非常简单的东西，一边从生产者接收消息，另一边将消息推送到队列。exchange必须确切知道对接收到的消息做什么，是将它添加到一个特定的队列？还是添加到多个队列？或者是应该丢弃。这个规则的定义是由exchange的类型来决定的。

![exchange](https://raw.githubusercontent.com/rason/rason.github.io/master/image/exchanges.png)

有几种可用的exchange：`direct, topic, headers , fanout`。今天我们将关注最后一个——`fanout`。先让我们创建一个这个类型的exchange，将其命名为`logs`。

```
channel.exchangeDeclare("logs", "fanout");
```

`fanout`类型的exchange非常简单。从它的名字你应该可以大概猜到，它只是将它接收的所有消息广播到它知道的所有队列。这正是我们的日志系统所需要的。

<!-- more -->

### **列出exchanges的类型**

你可以在RabbitMQ服务器的终端中使用`rabbitmqctl`命令来列出exchanges的类型：

```
rason@rason-ThinkPad-T430 ~ $ sudo rabbitmqctl list_exchanges
Listing exchanges ...
amq.match	headers
amq.headers	headers
amq.direct	direct
amq.rabbitmq.trace	topic
spring-boot-exchange	topic
amq.rabbitmq.log	topic
amq.fanout	fanout
amq.topic	topic
	direct

```

在这个列表中有一些`amq.*`类型的exchanges和默认的(未命名)exchange。

### 未命名的exchange

在之前的教程中我们根本不知道exchanges的存在，但是我们依然能发送消息到队列中。那是因为我们同时设置空字符串使用了一个默认的exchange。

回想一下我们之前是如何发布消息的：

```
channel.basicPublish("", "hello", null, message.getBytes());
```

第一个参数就是exchange的名字。空字符串表示默认或未命名的exchange：消息路由到由routingKey（第二个参数）指定的名称的队列，如果存在的话。

现在，我们发布到我们命名的exchange中：

```
channel.basicPublish( "logs", "", null, message.getBytes());
```

## 临时队列

你可能记得之前的教程中，我们使用具有指定名称的队列（还记得hello和task_queue吗？）。命名队列对我们至关重要——我们需要将工作者进程指向同一个队列。当您想在生产者和消费者之间共享队列时，给队列命名很重要。

但是我们的日志系统不是这样的。我们想要收到所有的消息，而不是它们的一部分。我们也只对当前流动的消息而不是旧的消息感兴趣。为了解决这个问题，我们需要做两件事情。

首先，每当我们连接到Rabbit，我们需要一个新的、空的队列。为了实现这样的功能我们使用随机的名字来创建队列，或者更好的方式——让服务端为我们选择一个随机队列名字。

其次，一旦我们断开消费者连接，队列应该被自动删除。

在RabbitMQ的Java客户端，当我们不向queueDeclare() 方法传递参数时，那么将会创建一个非持久的、独立的自动删除队列。

```
String queueName = channel.queueDeclare().getQueue();
```

此时，queueName包含一个随机队列名称。比如看起来像这样的：`amq.gen-JzTY20BRgKO-HjmUJj0wLg`。

## Bindings

![Bindings](https://raw.githubusercontent.com/rason/rason.github.io/master/image/bindings.png)

我们已经创建了一个fanout类型的exchange和一个队列。现在我们需要告诉exchange发送消息到我们的队列。exchange和queue之间的关系成为绑定。

```
channel.queueBind(queueName, "logs", "");
```

现在开始`logs` exchange就会添加信息到我们的队列中。

> **列出Bindings**，你可以使用`rabbitmqctl list_bindings`命令列出正在使用的Bindings。

## 将它们结合到一起

![Pub/Sub](https://raw.githubusercontent.com/rason/rason.github.io/master/image/python-three-overall.png)

生产者程序发出日志消息，跟之前的教程没有太大的不同。最重要的改变就是将原来发送到未命名的exchange改为发送到我们的`logs` exchange。当发送消息时我们需要提供`routingKey`，但是它的值会被`fanout`类型的exchange忽略。下面就是我们的`EmitLog.java`程序代码：

```
package com.rason.rabbitmq.pubsub.hello;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

/**
 * Created by rason on 3/8/17.
 */
public class EmitLog {

    private static final String EXCHANGE_NAME = "logs";

    public static void main(String[] args) throws java.io.IOException,java.util.concurrent.TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

        String message = "Hello World!";
        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");

        channel.close();
        connection.close();
    }
}

```

正如你所见，在建立连接之后我们声明了一个exchange。这一步是必须的，因为禁止发送消息到一个不存在的exchange。

目前还没有queue绑定到exchange中，所以消息会丢失，但是对我们来说没什么问题，因为如果没有消费者监听消息的话可以直接丢弃掉。

下面是`ReceiveLogs.java` 代码

```
package com.rason.rabbitmq.pubsub.hello;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

/**
 * Created by rason on 3/8/17.
 */
public class ReceiveLogs {
    private static final String EXCHANGE_NAME = "logs";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
        String queueName = channel.queueDeclare().getQueue();
        channel.queueBind(queueName, EXCHANGE_NAME, "");

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                System.out.println(" [x] Received '" + message + "'");
            }
        };
        channel.basicConsume(queueName, true, consumer);
    }

}

```

当我们启动了两个`ReceiveLogs`程序时，然后通过`EmitLog`发送消息，你会发现两个程序都能接收到消息，这就是和我们前面教程中的工作队列不一样的地方了。为了方便我自己测试的时候没有一个输出到文件，一个输出到屏幕，只要两个都能接收到消息那就代表成功了。

另外，你可以使用`rabbitmqctl list_bindings`命令查看一下是否有两个队列绑定到了`logs` exchange。

```
rason@rason-ThinkPad-T430 ~ $ sudo rabbitmqctl list_bindings
Listing bindings ...
logs	exchange	amq.gen-2nmqGK0HMoE3mBTG4FcvqA	queue		[]
logs	exchange	amq.gen-dS77gWKpqQyAhXvNSdpOuQ	queue		[]

```

结果很显然：来自exchange的日志数据将传送到由服务器分配名称的两个队列中。这就是我们期望看到的结果。

## 总结

今天我们学习了RabbitMQ的**发布/订阅**模式，下一节将学习如何监听部分消息。

