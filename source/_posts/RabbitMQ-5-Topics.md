---
title: RabbitMQ-Five-Topics
date: 2017-03-21 08:37:13
categories: AMQP
tags: RabbitMQ 
description: RabbitMQ教程
---

## Topics

在上一篇教程中，我们改善了一下日志系统。替换了`fanout` 类型的exchange 为`direct`类型，使得我们可以选择性地接收不同级别的日志。

虽然`direct` 类型的exchange 改善了我们的系统，但它依然存在局限性——它不能基于多个标准进行路由。

在我们的日志系统中，我们可能不仅需要根据日志的级别，而且还需要根据日志的发出点来进行订阅。你可能知道这个概念来自于UNIX 系统工具[syslog](https://en.wikipedia.org/wiki/Syslog)，它的日志路由就是基于级别(info/warn/cirt...)和来源(auth/corn/kern...)。

这样的话就会灵活很多，比如我们想监听来自于`cron`的严重错误日志和来自于`kern`的所有日志。

为了实现这样的日志系统，我们需要学习更加复杂的`topic` 类型的exchange。

## Topic exchange

发送到`topic` 类型exchange 的消息不能是任意的routing_key——它必须是由点分隔的单词列表。这些单词可以随便取，但通常它们带有一些消息的链接特征。比如说："stock.usd.nyse", "nyse.vmw", "quick.orange.rabbit"。当然不是你想要多长就有多长，路由键的最大长度是255个字节。

`binding key`的格式和`routing key`格式是一样的。`topic` 类型exchange 背后的逻辑类似于`direct` 类型exchange  - 使用特定`routing key`发送的消息将被递送到用匹配`binding key`绑定的所有队列。但是有两个重要的特殊情况用于`binding key`：

- **\*** 号可以替换一个单词。
- **\#** 号可以替换0个或者多个单词。

下面的例子可以很简单地解析这两种用法：

![Topic](https://raw.githubusercontent.com/rason/rason.github.io/master/image/python-five.png)

在这个例子中，我们准备发送描述动物的信息。这些信息将会通过一个带有三个单词（两个点）的路由键发送。第一个单词描述的是速度，第二个单词描述的是颜色，第三个单词描述的是种类： `<speed>.<colour>.<species>`。

我们创建三个`binding`：Q1的`binding key`是`*.orange.*`，Q2 的是 `*.*.rabbit` 和 `lazy.#`。

这些`bindings`可以描述为：

- Q1对所有橙色的动物感兴趣。
- Q2对多所有的兔子和所有速度缓慢的动物感兴趣。

如果一条消息通过"quick.orange.rabbit"的`routing key` 发送的话两个队列都会收到消息。"lazy.orange.elephant"的消息两个队列也能收到。

如果是"quick.orange.fox"那就只有第一个队列收到了，如果是"lazy.brown.fox"则只有第二个队列收到。

如果是"lazy.pink.rabbit" 只会被发到第二个队列一次，并不会发送两次，虽然它两个规则都匹配。

而"quick.brown.fox"不能匹配到任何地方则会被丢弃。

如果我们不遵循自己约定的规则通过一个或者四个单词的`routing key`来发送消息会怎样？比如 "orange" 或者 "quick.orange.male.rabbit"。因为这些消息不能匹配到任何`binding`，所以会丢失。

另外，假如是"lazy.orange.male.rabbit"，虽然有四个单词，但是它能被匹配，所以会发送到第二个队列。

<!-- more -->

## Topic exchange

Topic exchange 功能是十分强大的，它可以配置表现出其它类型exchange 的特性。

如果一个队列的`binding key`是“\#”——那么它将接收到所有消息，不管`routing key`是什么——这就好像`fanout` 类型的exchange。

如果`binding key` 不使用通配符“\*”和“\#”，那么`topic` 类型的exchange 就和`direct`的一样了。


## 将它们结合到一起

我们准备使用`topic` exchange来改造日志系统。假设我们的`routing keys`由两个单词组成：”来源.严重性“。

代码跟之前的差不多，只需要进行简单的修改。

下面是`EmitLogTopic.java`的代码：

```
package com.rason.rabbitmq.topic.hello;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

/**
 * Created by rason on 3/10/17.
 */
public class EmitLogTopic {
    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] args) throws java.io.IOException,java.util.concurrent.TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, "topic");

        String routingKey = getRoutingKey(args);
        String message = getMessage(args);

        channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes());
        System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");

        channel.close();
        connection.close();
    }

    private static String getRoutingKey(String[] args) {
        if (args.length < 1)
            return "auth.info";
        return args[0];
    }

    private static String getMessage(String[] args) {
        if (args.length < 2)
            return "Hello World!";
        return args[1];
    }
}

```

下面是`ReceiveLogsTopic.java`的代码：

```
package com.rason.rabbitmq.topic.hello;

import com.rabbitmq.client.*;

import java.io.IOException;

/**
 * Created by rason on 3/10/17.
 */
public class ReceiveLogsTopic {
    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "topic");
        String queueName = channel.queueDeclare().getQueue();

        if (args.length < 1){
            System.err.println("Usage: ReceiveLogsTopic [binding_key]...");
            System.exit(1);
        }
        for(String bindingKey : args){
            channel.queueBind(queueName, EXCHANGE_NAME, bindingKey);
        }

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                System.out.println(" [x] Received '" + envelope.getRoutingKey() + "':'" + message + "'");
            }
        };
        channel.basicConsume(queueName, true, consumer);
    }
}

```

## 测试

我分别以”auth.info“,"auth.\*","\#"三个`binding key`启动三个接收进程，如下：

```
rason@rason-ThinkPad-T430 ~ $ sudo rabbitmqctl list_bindings
Listing bindings ...
topic_logs	exchange	amq.gen-9UdsYf_QobTb6EfhzqTGrQ	queue	#	[]
topic_logs	exchange	amq.gen-Uou5vEJWPs0aODWM5mN7uw	queue	auth.*	[]
topic_logs	exchange	amq.gen-Yo9WdpEFi-7XtnMiGgs11Q	queue	auth.info	[]

```

当我发送`routing key`为"auth.info"的信息，三个队列都能接收到。

当我发送`routing key`为”auth.critical“的信息时，只有`binding key`为"auth.\*"和"\#"的队列能接收到。

当我发送`routing key`为”kern.critical“的信息时，只有`binding key`为"\#"的队列能接收到。

其它情况不逐一演示了。

## 总结

今天我们学习了`topic` 类型的exchange，使用通配符的方式进行绑定可以很灵活地适用到各种情况。下一篇教程中我们将学习如何利用消息往返来实现RPC。
