---
title: RabbitMQ-SIX-RPC
date: 2017-03-23 09:00:03
categories: AMQP
tags: RabbitMQ 
description: RabbitMQ教程
---

## Remote procedure call (RPC)

在RabbitMQ第二个教程中，我们学习了如何使用工作队列在多个工作线程间分配耗时的任务。

但是如果我们需要运行一个在远程计算机上的函数并等待结果呢？这种模式通常称为远程过程调用或RPC。

在今天这篇教程中我们将使用RabbitMQ来创建一个RPC系统：一个客户端和一个可扩展的RPC服务器。我们将要创建一个虚拟的RPC服务，返回斐波纳契数列计算值。

## 客户端接口

为了说明如何使用RPC服务，我们将创建一个简单的客户端类。它将暴露一个名为call的方法，它发送一个RPC请求并阻塞，直到接收到响应：

```
FibonacciRpcClient fibonacciRpc = new FibonacciRpcClient();   
String result = fibonacciRpc.call("4");
System.out.println( "fib(4) is " + result);
```

## RPC需要注意的地方

虽然RPC在计算中是一个很常见的模式，但它经常被批评。当程序员不知道函数调用是本地还是缓慢的RPC时，就会出现问题。这样的混乱会导致不可预测的系统，并增加不必要的调试复杂性。误用RPC不但没有简化系统，可能导致像意大利面条一样无法维护的代码。

牢记这一点，请考虑以下建议:

- 确保很明显哪个函数调用是本地的，哪个是远程的。
- 记录系统。使组件之间的依赖关系清晰。
- 处理错误情况。当RPC服务器长时间停机时，客户端应该如何处理？

当有疑问时避免使用RPC。如果可以，您应该使用异步流水线，而不是类似RPC的阻塞，结果被异步推送到下一个计算阶段。

## 回调队列

实际上通过RabbitMQ来实现RPC十分简单。一个客户端发送请求信息和一个服务端返回响应信息。为了接收响应信息我们需要在请求信息中包含一个‘callback’ 队列。我们可以使用默认队列（在Java 客户端中是唯一的名字）。

```
callbackQueueName = channel.queueDeclare().getQueue();

BasicProperties props = new BasicProperties
                            .Builder()
                            .replyTo(callbackQueueName)
                            .build();

channel.basicPublish("", "rpc_queue", props, message.getBytes());

// ... then code to read a response message from the callback_queue ...
```

## Message properties

AMQP协议中定义了消息的一系列属性。除了下面的属性之外，大部分都很少使用。

- **persistent**：标记消息为持久(true)或者暂存(false)，教程二中有提及。
- **content_type**：用于描述编码的mime-type。例如对于经常使用的JSON编码，一个好的做法是将此属性设置为：application/json。
- **reply_to**：通常用来命名一个回调队列。
- **correlation_id**：用于将RPC响应与请求相关联。

## Correlation Id

在上面提供的方法中，我们建议为每个RPC请求创建一个回调队列。这是非常低效的，但幸运的是有一个更好的方法——让我们为每个客户端创建一个回调队列。

但是这样会引起新的问题，当队列里有了返回消息，我们怎么知道这个消息是对应哪个请求？这就是`correlation_id`属性的用途了。我们为每个请求设置唯一的`correlation_id`值。然后当响应消息返回时我们就可以查看这个属性值，这样就能知道对应哪个请求了。如果查看到的是未知的`correlation_id`值，我们就可以忽略这个响应消息——因为它不属于我们的请求。

你可能会问，为什么我们应该忽略回调队列中的未知消息，而不是失败的错误。这是由于服务器端出现竞争条件的可能性。虽然不太可能，但是RPC服务器可能在发送我们响应之后但在发送请求的确认消息之前死亡。如果发生这种情况，重新启动的RPC服务器将再次处理该请求。这就是为什么在客户端上我们必须优雅地处理重复的响应，并且RPC应该是幂等的。

<!-- more -->

## 概括

![RPC](/image/python-six.png)

我们的RPC工作流程：

- 当客户端启动时，它会创建一个匿名、唯一的回调队列。

- 对于一个RPC请求，客户端发送的消息包含两个属性：`reply_to`，设置回调队列；`correlation_id`，设置请求的唯一值。

- 请求被发送到rpc_queue队列中。

- RPC服务器等待队列中的请求。当有一个请求发生，它就会完成这个任务然后发送响应消息到`reply_to`属性设置的队列中。

- 客户端等待回调队列中的数据。当有响应消息到来，它会检查`correlation_id`属性。如果它匹配来自请求的值，则它将响应返回给应用程序。

## 将它们结合到一起

Fibonacci任务：

```
private static int fib(int n) {
    if (n == 0) return 0;
    if (n == 1) return 1;
    return fib(n-1) + fib(n-2);
}
```

声明我们的fibonacci函数。假设只有有效的正整数输入。（不要指望这个工作的大数字，这可能是最慢的递归实现）

下面是`RPCServer.java `代码：

```
package com.rason.rabbitmq.rpc.hello;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

/**
 * Created by rason on 3/15/17.
 */
public class RPCServer {

    private static final String RPC_QUEUE_NAME = "rpc_queue";

    public static void main(String[] argv) {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = null;
        try {
            connection      = factory.newConnection();
            final Channel channel = connection.createChannel();

            channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);

            channel.basicQos(1);

            System.out.println(" [x] Awaiting RPC requests");

            Consumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                    AMQP.BasicProperties replyProps = new AMQP.BasicProperties
                            .Builder()
                            .correlationId(properties.getCorrelationId())
                            .build();

                    String response = "";

                    try {
                        String message = new String(body,"UTF-8");
                        int n = Integer.parseInt(message);

                        System.out.println(" [.] fib(" + message + ")");
                        response += fib(n);
                    }
                    catch (RuntimeException e){
                        System.out.println(" [.] " + e.toString());
                    }
                    finally {
                        channel.basicPublish( "", properties.getReplyTo(), replyProps, response.getBytes("UTF-8"));

                        channel.basicAck(envelope.getDeliveryTag(), false);
                    }
                }
            };

            channel.basicConsume(RPC_QUEUE_NAME, false, consumer);

            //...
        } catch (TimeoutException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private static int fib(int n) {
        if (n == 0) return 0;
        if (n == 1) return 1;
        return fib(n-1) + fib(n-2);
    }
}

```

服务端的代码直接明了：

- 首先建立connection, channel 和声明queue。

- 我们可能会运行多个服务端进程。为了均摊负载设置了` prefetchCount`的值为1。

- 使用`basicConsume`方法访问队列，我们提供了一个consumer来完成任务并返回响应信息。

下面是` RPCClient.java`代码：

```
package com.rason.rabbitmq.rpc.hello;

import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.UUID;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeoutException;

/**
 * Created by rason on 3/15/17.
 */
public class RPCClient {
    private Connection connection;
    private Channel channel;
    private String requestQueueName = "rpc_queue";
    private String replyQueueName;

    public RPCClient() throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        connection = factory.newConnection();
        channel = connection.createChannel();

        replyQueueName = channel.queueDeclare().getQueue();
    }

    public String call(String message) throws IOException, InterruptedException {
        final String corrId = UUID.randomUUID().toString();

        AMQP.BasicProperties props = new AMQP.BasicProperties
                .Builder()
                .correlationId(corrId)
                .replyTo(replyQueueName)
                .build();

        channel.basicPublish("", requestQueueName, props, message.getBytes("UTF-8"));

        final BlockingQueue<String> response = new ArrayBlockingQueue<String>(1);

        channel.basicConsume(replyQueueName, true, new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
                if (properties.getCorrelationId().equals(corrId)) {
                    response.offer(new String(body, "UTF-8"));
                }
            }
        });

        return response.take();
    }

    public void close() throws IOException {
        connection.close();
    }

    public static void main(String[] args) throws IOException, InterruptedException, TimeoutException {
        RPCClient fibonacciRpc = new RPCClient();

        System.out.println(" [x] Requesting fib(30)");
        String response = fibonacciRpc.call("30");
        System.out.println(" [.] Got '" + response + "'");

        fibonacciRpc.close();
    }
}

```

客户端代码稍微有点复杂：

- 首先建立connection和channel，然后声明一个独立的回调队列。

- 我们订阅回调队列，以便我们可以接收RPC响应。

- 我们的call方法发起真正的RPC请求。

- 在这里我们首先生成一个唯一的`correlationId`，我们的DefaultConsumer会使用这个值来获得相应的返回结果。

- 下一步，我们发布请求消息，配置两个属性： replyTo 和 correlationId。

- 这时我们就可以等待相应的返回信息。

- 由于我们的消费者处理响应消息是在一个单独的线程中进行的，我们需要在响应到达之前挂起主线程。使用`BlockingQueue`是一种可用的解决方式。这里我们创建了容量为1的ArrayBlockingQueue，因为我们只需等待一个响应。

- `handleDelivery`方法处理很简单，对于每个消费的响应消息，它检查correlationId是否是我们正在寻找的。如果是，则将响应信息放到BlockingQueue中。

- 同时主线程正在等待响应从BlockingQueue中获取它。

- 最后，我们将响应返回给用户。

## 测试

先运行服务端程序，然后再运行客户端程序可以看到输出结果。

## 总结

这里介绍的设计不是RPC服务的唯一可能的实现，但它有一些重要的优势：

- 如果RPC服务器太慢，您可以通过运行另一个进行扩展。尝试在新控制台中运行第二个RPCServer。

- 在客户端，RPC只需要发送和接收一个消息。不需要像queueDeclare这样的同步调用。因此，RPC客户端对于单个RPC请求只需要一个网络往返。

我们的代码仍然相当简单，没有尝试解决更复杂（但重要）的问题，比如：

- 如果没有服务器正在运行，客户端应该如何反应？

- 客户端是否有某种RPC请求超时策略？

- 如果服务器发生故障并引发异常，是否应将其转发到客户端？

- 在处理消息之前防止无效的输入（例如检查边界，类型）。


