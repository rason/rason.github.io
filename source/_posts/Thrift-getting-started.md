---
title: Thrift getting started
date: 2017-10-26 16:36:01
categories: Distributed
tags: Thrift
description: Thrift getting started
---

## 前言

[Apache Thrift](https://thrift.apache.org/)是一个可伸缩的跨语言服务开发框架，重点就是 **RPC** 和 **跨语言**。

## 跨语言

那么，Thrift是如何实现跨语言的？因为其使用了与语言无关的语法来定义接口，然后利用自身的编译器生成相应的接口代码，也就是IDL（interface description language）。

## RPC

什么是RPC？简单来说就是像调用本地代码一样调用远程代码。也就是说，RPC框架封装了消息发送和接收这个过程，让我们看起来像是调用本地代码一样。其过程如下：

- 1.客户端调用客户端stub。这个调用是本地过程调用，跟普通的的方式一样将参数压栈。

- 2.客户端stub 将参数打包成消息然后发起一个系统调用来发送消息。参数打包的过程称为marshalling。

- 3.客户端本地操作系统将消息从客户端机器发送到服务端机器。

- 4.服务端机器操作系统将接收到的消息数据包传递给服务端Skeleton。

- 5.服务端Skeleton 将消息解包。这个解包的过程称为unmarshalling。

- 6.最后，服务端Skeleton 调用相应的服务。响应过程跟接收过程一样，只是反过来了而已。

## Thrift RPC

那么Thrift 是如何实现上述的RPC 过程？我们直接上代码学习。

### 先使用IDL来定义一个简单的接口

新建一个文本，文件名为hello.thrift，内容如下

```
namespace java com.rason.thrift.rpc

service HelloService {
    void hello(1:string name)
}
```

<!-- more -->

### 使用thrift 编译器来生成相应语言的代码

```
thrift --gen java hello.thrift
```

### 自动生成的代码

运行上述命令之后，会生成一个HelloService类，这个类比较长，只是简单贴一点代码：

```
@SuppressWarnings({"cast", "rawtypes", "serial", "unchecked", "unused"})
@javax.annotation.Generated(value = "Autogenerated by Thrift Compiler (0.10.0)", date = "2017-10-26")
public class HelloService {

  public interface Iface {

    public void hello(String name) throws org.apache.thrift.TException;

  }
  ...
  ...
  ...
}  
```

### 编写服务端的实现类

接口定义好了，当然是编写服务端的实现类：

```
package com.rason.thrift.rpc;

import org.apache.thrift.TException;

public class HelloServiceImpl implements HelloService.Iface {
    public void hello(String name) throws TException {
        System.out.println("hello " + name);
    }
}
```

### 编写服务端代码

现在，实现类已经写好了。很显然，我们现在需要启动一个服务，监听某个端口来接收消息，然后将接收到的参数传递给这个实现类的hello方法。听起来这个过程很难实现？没关系，thrift 框架已经将这个过程进行了抽象封装，我们只需要编写几行简单的代码即可：


```
package com.rason.thrift.rpc;

import org.apache.thrift.server.TServer;
import org.apache.thrift.server.TSimpleServer;
import org.apache.thrift.transport.TServerSocket;
import org.apache.thrift.transport.TServerTransport;
import org.apache.thrift.transport.TTransportException;

public class JavaServer {

    public static HelloService.Processor processor;

    public static void main(String[] args) throws TTransportException {

        // 实际干活的实现类
        HelloService.Iface helloService = new HelloServiceImpl();

        // 处理器，这是有thrift 自动生成的，我们只需要将真正要干活的实现类传进去即可
        processor = new HelloService.Processor(helloService);

        // 传输层抽象，即监听某个接口这一动作
        TServerTransport serverTransport = new TServerSocket(9090);

        // 服务端抽象，即会启动传输层抽象的监听动作，接收消息，然后将消息交给相应的处理器
        TServer server = new TSimpleServer(new TServer.Args(serverTransport).processor(processor));
        
        // 启动服务
        server.serve();
    }
}

```

### 编写客户端代码

客户端需要干啥工作？很显然，就是调用接口啊！不是说还要将消息打包然后系统调用发送消息嘛？这个过程thrift 早就抽象封装好了啊！要不然怎么能像调用本地代码一样调用远程代码？


```
package com.rason.thrift.rpc;

import org.apache.thrift.TException;
import org.apache.thrift.protocol.TBinaryProtocol;
import org.apache.thrift.protocol.TProtocol;
import org.apache.thrift.transport.TSocket;
import org.apache.thrift.transport.TTransport;

public class JavaClient {
    public static void main(String[] args) throws TException {

        // 连接到服务端的9090端口
        TTransport transport = new TSocket("localhost", 9090);
        transport.open();

        // 传输协议，也就是消息以什么样的格式来发送接收
        TProtocol protocol = new TBinaryProtocol(transport);

        // 获得访问接口的客户端，这个也是由thrift 生成的
        HelloService.Client client = new HelloService.Client(protocol);

        // 调用接口
        client.hello("rason");

        transport.close();

    }
}
```


你看，就是这么的简单，thrift 帮你生成了一个访问远程接口的客户端，你就跟调用本地代码一样进行远程调用。

