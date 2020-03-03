title: Tomcat NIO 基本架构
categories: Tomcat
date: 2015-12-07 20:53:05
tags: Tomcat NIO
description: Tomcat NIO 基本架构
---

## 前言

上几篇文章梳理了下Tomcat容器的基本架构和请求处理流程，关于请求处理还有一个重要的组件——Connector组件。Tomcat8中的Connector组件默认是使用NIO来处理请求，这两天大概地研究了一下NIO在Tomcat中的使用，借此机会熟悉一下NIO编程。

## Tomcat NIO基本架构

这篇文章打算先从整体概括一下Tomcat NIO基本架构，然后再具体分析。先看一下基本的架构图：

![Tomcat NIO](/image/tomcatnio-strcuture.png)

<!-- more -->

从上图我们可以看出，在Tomcat中利用NIO来处理请求主要包括三个部分：

- **Acceptor线程侦听连接**
- **Poller线程侦听请求**
- **Executor线程池数据处理**

### Acceptor

Acceptor线程用于侦听Socket链接。Acceptor是单线程模式的，会调用serverSock.accept()阻塞的方式等待请求的到来，以获得一个SocketChannel。当获取到一个SocketChannel之后，会将其包装到`org.apache.tomcat.util.net.NioChannel`对象中。然后利用轮询调度的方式获取一个Poller（关于Poller内容在后面会有解析），再将SocketChannel的包装对象NioChannel注册到该Poller中。该注册过程实际上就是将SocketChannel注册到Selector（每一个Poller拥有一个Selector）中，这就是NIO的知识了。但是在实现过程中并不是直接注册的，而是将NioChannel对象作为一个PollerEvent添加到Poller的PollerEvent队列中，然后Poller会从队列中取出来再将SocketChannel注册到Selector中。这就是典型的生产者-消费者模式，Acceptor生产连接事件包装为PollerEvent添加到队列中，Poller将事件从队列中取出执行。

我们看一下这一下过程的伪代码：

```

Acceptor.run(){
	SocketChannel socket = serverSock.accept();
	NioChannel channel = new NioChannel(socket, bufhandler);
	getPoller0().register(channel);
}

Poller.register(final NioChannel socket){
 	socket.setPoller(this);
    KeyAttachment ka = new KeyAttachment(socket);
    ka.setPoller(this);
	PollerEvent r = new PollerEvent(socket,ka,OP_REGISTER);
	addEvent(r);
}

PollerEvent.run(){
	socket.getIOChannel().register(socket.getPoller().getSelector(), SelectionKey.OP_READ, key);
}

```

### Poller

Poller线程用于侦听请求。Tomcat8中会根据java虚拟机可用的处理器数创建最多两条Poller线程。Poller维护一个Selector，在Acceptor中获取的SocketChannel就是注册到该Selector中。Poller线程的run方法除了上面谈到的将PollerEvent取出来运行注册感兴趣的channel事情外，还会调用selector.selectNow()或者selector.select(selectorTimeout)方法检查是否有感兴趣的事件发生。当有可以读写的channel，则会将该socket交给Executor线程池中的一条线程去处理。

这一过程的伪代码如下：

```
class Poller{
	private Selector selector;
    private final SynchronizedQueue<PollerEvent> events =
                new SynchronizedQueue<>();
    run(){
    	events();
    	keyCount = selector.selectNow();
    	Iterator<SelectionKey> iterator =
        keyCount > 0 ? selector.selectedKeys().iterator() : null;
        
        while (iterator != null && iterator.hasNext()) {
            SelectionKey sk = iterator.next();
            KeyAttachment attachment = (KeyAttachment)sk.attachment();
            
            attachment.access();
            iterator.remove();
            processKey(sk, attachment);
        }
	}

	protected boolean processKey(SelectionKey sk, KeyAttachment attachment) {
		if (sk.isReadable()) {
            processSocket(attachment, SocketStatus.OPEN_READ, true)
        }
        if (sk.isWritable()) {
            processSocket(attachment, SocketStatus.OPEN_WRITE, true)
        }
	}
}

NioEndPoint.processSocket(KeyAttachment attachment, SocketStatus status, boolean dispatch){
	SocketProcessor sc = new SocketProcessor(attachment, status);
    Executor executor = getExecutor();
    executor.execute(sc);
    }
}
```

### Executor

Executor线程池负责数据处理和业务调用。Poller接收到的请求socket会将其封装到SocketProcessor来处理，SocketProcessor从Http11ConnectionHandler中取出Http11NioProcessor对象，从Http11NioProcessor中调用CoyoteAdapter的逻辑。CoyoteAdapter会将org.apache.coyote.Request转成org.apache.catalina.connector.Request，接下来就是将请求交给各级Container直接Servlet处理了。

## 请求在Connector中的流转

上面的描述在表达上可能有点欠缺，不保证严格的准确性。为了更好地理解这一流转过程，我们看一下请求流转图：

![Connector请求流转](/image/tomcatnio-dispath.png)

## 总结

看完这篇文章，应该对下面两个问题有所了解：

- **理解Connector中请求处理的三个阶段：连接，接收，处理**
- **理解Connector内部的请求流转过程**
