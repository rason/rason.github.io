---
title: RabbitMQ-Two-WorkQueues
date: 2017-03-09 10:32:52
categories: AMQP 
tags: RabbitMQ
description: RabbitMQ教程
---

## Work Queues

![工作队列](https://raw.githubusercontent.com/rason/rason.github.io/master/image/python-two.png)

上一篇教程中我们编写程序从一个命名的queue中发送和接收消息。在这篇教程中，我们将创建一个工作队列，用于在多个工作者之间分配耗时的任务。

工作队列（也称为任务队列）背后的主要思想是避免立即执行资源密集型任务，并且不得不等待它完成。相反，我们安排任务以后完成。我们将任务封装成一个消息然后将其发送到队列中。在后台运行的工作进程将弹出队列中任务并最终执行作业。当你运行很多工作进程时，任务将在它们之间共享。

这个概念在Web应用程序中尤其有用，因为在短期HTTP请求窗口中无法处理复杂任务。

## 准备

在前一篇教程我们发送一个“Hello World!”消息。现在我们将发送字符串代表复杂的任务。我们不需要一个真正工作的任务，比如图片的缩放或者是PDF文件的渲染，我们可以通过`Thread.sleep()`功能来模拟耗时任务。一个点代表工作一秒。比如，一个任务`Hello...`代表将耗时3秒。

我们将稍微修改一下之前的`Send.java`，允许从命令行中发送任意消息。这个程序将任务调度到我们的工作队列，所以将其命名为`NewTask.java`。

```
package com.rason.rabbitmq.workqueues.hello;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

import java.util.concurrent.TimeoutException;

/**
 * Created by rason on 3/6/17.
 */
public class NewTask {

    private static final String QUEUE_NAME = "hello";

    public static void main(String[] args) throws java.io.IOException,TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME, false, false, false, null);

        String message = getMessage(args);
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");

        channel.close();
        connection.close();
    }

    private static String getMessage(String[] strings) {
        if (strings.length < 1)
            return "Hello World!";
        return joinStrings(strings, " ");
    }

    private static String joinStrings(String[] strings, String delimiter) {
        int length = strings.length;
        if (length == 0) return "";
        StringBuilder words = new StringBuilder(strings[0]);
        for (int i = 1; i < length; i++) {
            words.append(delimiter).append(strings[i]);
        }
        return words.toString();
    }
}

```

<!-- more -->

我们的`Recv.java`也需要进行修改：它需要为消息正文中的每个点伪造一秒的工作。它将处理提交的消息并执行任务，所以我们将其称为`Worker.java`。

```
package com.rason.rabbitmq.workqueues.hello;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

/**
 * Created by rason on 3/6/17.
 */
public class Worker {

    private final static String QUEUE_NAME = "hello";

    public static void main(String[] args) throws IOException,TimeoutException {
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
                try {
                    doWork(message);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    System.out.println(" [x] Done");
                }
            }
        };
        boolean autoAck = true; // acknowledgment
        channel.basicConsume(QUEUE_NAME, autoAck, consumer);
    }

    private static void doWork(String task) throws InterruptedException {
        for (char ch: task.toCharArray()) {
            if (ch == '.') Thread.sleep(1000);
        }
    }
}

```

## 轮询调度

使用任务队列的优点之一是能够轻松地并行工作。如果我们积压了很多工作，我们只需要增加更多的工作者进程即可，这样的话扩展起来就很轻松了。

首先，我们同时运行两个工作者实例。他们都会从队列中获取消息，但是究竟会怎样呢？ 让我们来看看。

启动了工作者实例之后，我们运行任务实例，分别发送五个不同的任务，如下：

```
[x] Sent '1 message.................'
[x] Sent '2 message.................'
[x] Sent '3 message.................'
[x] Sent '4 message.................'
[x] Sent '5 message.................'
```

然后观察两个工作者实例的输出。

工作者实例1:

```
 [x] Received '1 message.................'
 [x] Done
 [x] Received '3 message.................'
 [x] Done
 [x] Received '5 message.................'
 [x] Done

```

工作者实例2:

```
 [x] Received '2 message.................'
 [x] Done
 [x] Received '4 message.................'
 [x] Done

```

默认情况下，RabbitMQ将按顺序发送每个消息到下一个消费者。平均每个消费者将获得相同数量的消息。这种分发消息的方式称为轮询调度。我们可以试下3个或者更多的工作者实例观察情况。

## 消息确认

执行任务可能需要几秒钟。你可能想知道，如果一个消费者开始一个长时间的任务，如果它只有部分完成会怎么样。我们当前的代码，一旦RabbitMQ将消息传送给消费者之后会立刻从内存中删除。在这种情况下，如果你杀掉一个工作者进程将会丢失它正在处理的消息。我们还将丢失所有发送给此特定工作者但尚未处理的消息。

但是我们并不想丢失任何任务。如果一个工作者进程挂掉了，我们希望任务能够传送到另外的工作者线程中。

为了确保消息永远不会丢失，RabbitMQ支持消息确认机制。从消费者发送回ack（nowledgement）以告诉RabbitMQ已经接收到特定消息，处理并且RabbitMQ可以自由删除它。

如果一个消费者挂掉了（比如channel关闭，connection关闭或者TCP连接丢失），并没有发送ack，RabbitMQ会理解为消息没有被完全处理完成并将其重新入队。如果有另外的消费者在线，则会马上将消息重新发送到另外的消费者。通过这样的方式，你可以确保消息不会被丢失，即使是工作者进程突然挂掉了。

处理消息没有超时的概念，当消费者死亡时，RabbitMQ将重新传递消息。即使处理消息的时间很长，很长都没有问题，反而很好，因为这正是消息队列的作用。

默认情况下，消息确认机制会自动开启。在前面的代码中我们通过设置`autoAck=true`关闭了确认机制。现在我们将其设为false，一旦我们完成了任务之后，发送一个正确的确认答复。

```
        channel.basicQos(1); // accept only one unack-ed message at a time (see below)

        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                String message = new String(body, "UTF-8");

                System.out.println(" [x] Received '" + message + "'");
                try {
                    doWork(message);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    System.out.println(" [x] Done");
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }
            }
        };
        boolean autoAck = false; // acknowledgment
        channel.basicConsume(QUEUE_NAME, autoAck, consumer);
```

使用这段代码我们就可以确保消息不会丢失，即使我们中途把工作者进程杀掉。工作者进程挂掉后，所有未确认的消息将很快被重新传递到另外的工作者进程。

> **忘记进行消息确认**，这是一个很常见的错误，但后果会非常严重。RabbitMQ会吃掉越来越多的内存，因为它无法释放任何还没有确认的消息。为了debug这类型的错误，你可以使用`rabbitmqctl`来打印出未被确认的消息域。

```
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
Listing queues ...
hello	0	1
```

## 消息持久化

我们已经学会了当消费者挂掉后如何确保消息不会丢失。但是如果RabbitMQ服务端挂掉的话，消息依然会丢失。

当RabbitMQ服务端退出或者突然挂掉，它会忘记所有的队列和消息，除非我们告诉它不要忘记。我们需要做两件事情来确保消息不会丢失：将队列和消息都标记为持久化。

首先，我们需要确保RabbitMQ不要丢失队列。为了达到这样的效果，我们需要声明它为持久化：

```
boolean durable = true;
channel.queueDeclare("hello", durable, false, false, null);
```

虽然上面的代码完全正确，但是在我们目前的设置中不会生效。因为我们已经声明了一个名为`hello`的非持久化队列。RabbitMQ不会让你使用不同的参数重定义一个已经存在的队列。我们有一个快速的解决方案——使用别的名字定义一个新的队列，比如task_queue：

```
boolean durable = true;
channel.queueDeclare("task_queue", durable, false, false, null);
```

`queueDeclare`方法需要生产者和消费者两个地方都要同时改动才能生效。

现在我们已经确保了RabbitMQ重启时队列不会丢失，现在我们需要将消息也标记为持久化——通过设置`MessageProperties`的`PERSISTENT_TEXT_PLAIN`值。

```
import com.rabbitmq.client.MessageProperties;

channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

> 消息持久化注意事项：将消息标记为持久化也不能完全保证所有消息不会丢失。虽然我们已经告诉了RabbitMQ将消息保存到硬盘，但是当RabbitMQ已经接受消息并且还没有保存它时，仍然有一个短暂时间窗口。另外，RabbitMQ也不是马上将消息同步到硬盘，只是先将其保存到缓存中。所以这里的持久化不是特别的严格，但是对于一般的任务来说已经足够了。如果你需要一个强持久化的方案，你可以使用[ publisher confirms](https://www.rabbitmq.com/confirms.html)。

## 均衡调度

你可能已经注意到目前的调度方式还不完全是我们想要的。比如有两个工作者进程的情况下，当偶数消息的任务比较繁重，奇数消息的任务比较简单，这样的话一个工作者会持续繁忙而另一个工作者几乎不做任何工作。好吧，RabbitMQ不知道任何事情，仍然会均匀发送消息。

这是因为RabbitMQ只是在消息进入队列时分派消息，它不查看消费者的未确认消息的数量，它只是盲目地向第n个消费者分派第n个消息。

![prefetch-count](https://raw.githubusercontent.com/rason/rason.github.io/master/image/prefetch-count.png)

为了解决这个问题我们使用`basicQos`方法将`prefetchCount`的值设为1。这告诉RabbitMQ不要一次给一个工作者进程一个以上的消息。或者，换句话说，不要向工作者进程分派新消息，直到它处理并确认了前一个消息。取而代之，它会将其分派给下一个仍然不忙的工作者。

```
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

> **注意队列大小**：如果所有工作者进程都很繁忙，你的队列可能很快就会被塞满。你需要留意一下这种情况的发生，然后添加更多的工作者进程或者采用其它策略。

## 总结

我们学会了使用消息确认和prefetchCount来设置消息队列，还有消息持久化的选项设置。更多关于Channel和方法和MessageProperties的信息，可以参考[在线文档](http://www.rabbitmq.com/releases/rabbitmq-java-client/current-javadoc/)。
