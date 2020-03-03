---
title: RabbitMQ-Four-Routing
date: 2017-03-15 10:44:04
categories: AMQP
tags: RabbitMQ 
description: RabbitMQ教程
---

## Routing

在上一篇教程中我们创建了一个简单的日志系统。我们可以广播日志信息到很多接收者。

在今天的教程中我们添加一个新特性——使它能够订阅部分的消息。例如，我们可以仅将关键错误消息定向到日志文件（为了节省硬盘空间），同时仍然能够在控制台上打印所有日志消息。

## Bindings

在之前的例子中我们已经创建了Bindings。你应该记得代码是这样的：

```
channel.queueBind(queueName, EXCHANGE_NAME, "");
```

binding表示的是exchange和queue之间的绑定关系。可以简单地理解为：队列对这个exchange的消息感兴趣。

Bindings可以使用额外的routingKey参数。为了避免与`basic_publish`参数混淆，我们将其称为`binding key`。下面是使用key来创建一个binding：

```
channel.queueBind(queueName, EXCHANGE_NAME, "black");
```

`binding key`的含义取决于exchange的类型。上一篇文章中的`fanout`类型会忽略它的值。

## Direct exchange

上篇教程的日志系统广播所有消息给所有消费者。我们希望将其扩展为允许根据消息的重要性过滤消息。例如我们可能想一个程序只接收关键错误日志信息并将其写到硬盘中，而警告信息和一般信息就不写了，节约硬盘空间。

我们之前使用的`fanout`类型的exchange不够灵活，因为它只是盲目地全部广播。

我们将使用`direct`类型的exchange来代替。`direct` 类型的exchange 背后的路由算法很简单——消息将发送到队列的`binding key`与发送消息时的`routing key`完全匹配的队列。

为了演示这个模式，请看下图：

![direct-exchange](/image/direct-exchange.png)

上图中，我们可以看到一个`direct` 类型的exchange X有两个队列绑定到它。第一个队列使用`binding key` orange 来绑定，第二个队列有两个bindings，一个是black一个是green。

这样的设置，如果一个消息通过`routing key` orange 发送将会被路由到第一个队列Q1，如果消息通过`routing key` black 或者 green都将被路由到队列Q2。其它的消息则会被丢弃。

<!-- more-->

## Multiple bindings

![direct-exchange-multiple](/image/direct-exchange-multiple.png)

用相同的`binding key` 绑定多个队列是完全合法的。在我们的例子中可以使用`binding key` black 添加多一个`binding` ，如上图所示。这样的话，`direct` 类型的exchange 可以像 `fanout` 类型的exchange 一样将消息广播到所有匹配的队列中。`routing key` 为black 的消息会被同时发送到Q1和Q2。

## 发送日志

我们将在日志系统中使用这个模式。我们将消息发送到`direct`类型的exchange 而不是`fanout`类型的exchange 。我们将使用日志的严重性作为`routing key`。通过这个方式接收程序可以选择它想接收的日志。先让我们来看下发送日志程序。

跟之前一样，我们先创建一个exchange：

```
channel.exchangeDeclare(EXCHANGE_NAME, "direct");
```

然后准备发送消息：

```
channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes());
```

为了简化，我们假设'severity'可以是'info'，'warning'，'error'。

## 订阅

接收消息跟之前的基本一样，除了一点之外——我们将为每个我们感兴趣的严重性消息创建一个新的`binding`。

```
String queueName = channel.queueDeclare().getQueue();

for(String severity : argv){    
  channel.queueBind(queueName, EXCHANGE_NAME, severity);
}
```

## 将它们结合到一起

![日志消息路由](/image/python-four.png)

下面是` EmitLogDirect.java `的代码：

```
package com.rason.rabbitmq.routing.hello;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

/**
 * Created by rason on 3/8/17.
 */
public class EmitLogDirect {
    private static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] args) throws java.io.IOException,java.util.concurrent.TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        channel.exchangeDeclare(EXCHANGE_NAME, "direct");

        String severity = getSeverity(args);
        String message = getMessage(args);

        channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");

        channel.close();
        connection.close();
    }

    private static String getSeverity(String[] args) {
        if (args.length < 1)
            return "info";
        return args[0];
    }

    private static String getMessage(String[] args) {
        if (args.length < 2)
            return "Hello World!";
        return args[1];
    }

}

```

下面是`ReceiveLogsDirect.java`的代码：

```
package com.rason.rabbitmq.routing.hello;

import com.rabbitmq.client.*;

import java.io.IOException;

/**
 * Created by rason on 3/8/17.
 */
public class ReceiveLogsDirect {
    private static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] args) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "direct");
        String queueName = channel.queueDeclare().getQueue();

        if (args.length < 1){
            System.err.println("Usage: ReceiveLogsDirect [info] [warning] [error]");
            System.exit(1);
        }
        for(String severity : args){
            channel.queueBind(queueName, EXCHANGE_NAME, severity);
        }

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

先使用`error`参数启动一个接收者进程ReceiveLogsDirect，再使用参数`info warning error`启动第二个接收者进程`ReceiveLogsDirect`。

假设我们使用路由键`info`或者`warning`来发送日志信息时，只有第二个进程接收到日志消息；假设我们使用`error`路由键发送信息时，两个进程都能接收到消息。

我们也可以使用`sudo rabbitmqctl list_bindings`命令查看绑定关系：

```
rason@rason-ThinkPad-T430 ~ $ sudo rabbitmqctl list_bindings
Listing bindings ...
direct_logs	exchange	amq.gen-7JhZWrkWhF66RGYCHlPtmA	queue	error	[]
direct_logs	exchange	amq.gen-Hyyd0gslITYZJukyPoCAuA	queue	error	[]
direct_logs	exchange	amq.gen-Hyyd0gslITYZJukyPoCAuA	queue	info	[]
direct_logs	exchange	amq.gen-Hyyd0gslITYZJukyPoCAuA	queue	warning[]

```

## 总结

今天我们学习了RabbitMQ的消息路由，下一篇文章将学习根据模式来监听消息。

