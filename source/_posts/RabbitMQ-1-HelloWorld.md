---
title: RabbitMQ-One-HelloWorld
date: 2017-03-07 11:00:56
categories: AMQP
tags: RabbitMQ
description: RabbitMQ教程
---

## 介绍

RabbitMQ是一个消息代理。主要想法很简单：接收然后转发消息。你可以将其想象成一个邮局：当你发送邮件到邮箱，你很确定邮递员最终会将邮件送到收件人手上。按照这样的比喻，RabbitMQ就相当于一个邮箱、邮局和邮递员。

RabbitMQ与邮局的主要不同点是它并不是使用纸质邮件，而是接收，存储和转发二进制消息数据块。

RabbitMQ和一般消息传递，会使用一些术语：

- **生产**：发送消息的程序称为生产者。我们通常会画成这样：

![生产者](https://raw.githubusercontent.com/rason/rason.github.io/master/image/producer.png)

- **队列**:相当于邮箱的名字。它存在于RabbitMQ中。虽然消息流经RabbitMQ和你的应用，但它们只能被保存在队列中。队列没有界限，你想要多少它就能存储多少消息，本质上是一个无限缓冲区。多个生产者可以发送消息到一个队列，多个消费者和从一个队列获取数据。一个队列我们可以画成这样：

![队列](https://raw.githubusercontent.com/rason/rason.github.io/master/image/queue.png)

- **消费**：消费跟接收具有类似的含义。一个消费者就是一个等待接受消息的程序。我们通常会画成这样：

![消费者](https://raw.githubusercontent.com/rason/rason.github.io/master/image/consumer.png)

<!-- more -->

## Hello World

在这部分教程，我们准备写两个程序：一个生产者发送一条消息和一个消费者接收消息并将其打印出来。使用Java API将会掩盖部分实现细节，聚焦于简单的事情。

如下图所示，“P”代表生产者，“C”代表消费者。中间的盒子代表一个队列——RabbitMQ代表消费者保存消息的缓存区。

![消息转发](https://raw.githubusercontent.com/rason/rason.github.io/master/image/python-one.png)

> 使用RabbitMQ我们需要导入其Java客户端库，自行搜索依赖版本添加到pom.xml文件中即可。

### 发送消息

![发送消息](https://raw.githubusercontent.com/rason/rason.github.io/master/image/sending.png)

我们将发送者命名为`Send`，接收者命名为`Recv`。发送者将会连接到RabbitMQ，发送一条消息，然后退出。

```
package com.rason.rabbitmq.helloworld.hello;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.util.concurrent.TimeoutException;

/**
 * Created by rason on 3/6/17.
 */
public class Send {

    private static final String QUEUE_NAME = "hello";

    public static void main(String[] args) throws java.io.IOException,java.util.concurrent.TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        String message = "Hello World!";
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");

        channel.close();
        connection.close();
    }
}

```

先声明了队列的名字，然后创建一个连接到服务端。这个连接是抽象的socket连接，为我们负责协议版本协商和认证等操作。这里我们使用`localhost`连接到本地机器的一个消息代理，这需要你在本地机器中安装了RabbitMQ服务端。如果你需要连接到其它机器的消息代理，修改`localhost`为相应的IP地址即可。

接下来我们创建了一个`channel`。为了发送消息，我们必须声明一个队列来接收发送的消息，然后将消息发送到队列中。

声明一个队列是幂等的，即它只会在不存在的情况下才会被创建。消息内容是一个字节数组，所以你可以对其进行任何形式的编码。

最后，我们关闭channel和connection。

## 接收消息

接收者的消息是由RabbitMQ推送过来的，所以不像发送者那样发布一条单一的消息，我们会让它一直运行着监听消息然后打印出来。

![接收消息](https://raw.githubusercontent.com/rason/rason.github.io/master/image/receiving.png)

```
package com.rason.rabbitmq.helloworld.hello;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

/**
 * Created by rason on 3/6/17.
 */
public class Recv {

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws java.io.IOException,java.util.concurrent.TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                System.out.println(" [x] Received '" + message + "'");
            }
        };

        channel.basicConsume(QUEUE_NAME, true, consumer);
    }

}

```

建立连接和发送者一样，打开一个connection和channel，然后声明一个准备消费的queue。注意这个queue就是接收发送者消息的queue。

我们在这里也声明了queue，因为我们可能会先启动接收者然后再启动发送者，我们需要确保队列的存在才能对其进行消费。

建立好connection和channel之后，我们就可以告诉服务端将queue中的消息传递过来了。由于服务端是异步推送消息，所以我们提供一个回调对象来缓冲着数据直到我们真正使用。这是`DefaultConsumer`类所做的工作。

## 测试

运行发送者，会看到以下输出

```
 [x] Sent 'Hello World!'
```

此时，我们可以看下RabbitMQ服务端的队列信息，在终端执行`sudo rabbitmqctl list_queues`命令，会看到以下信息

```
Listing queues ...
hello	1
```

然后，我们运行接收者，会看到以下输出

```
 [x] Received 'Hello World!'
```

再次执行`sudo rabbitmqctl list_queues`命令，发现消息已经被消费

```
Listing queues ...
hello	0
```

## 总结

本文演示了RabbitMQ最简单的应用，作为一个中介角色存储转发消息。
