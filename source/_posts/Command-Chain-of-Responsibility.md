title: 'Command&Chain of Responsibility'
categories: DesignPattern
date: 2016-01-14 14:27:11
tags: Command Chain of Responsibility
description: Command and Chain of Responsibility
---

## 前言

顺着前两篇文章，继续温习一下Tomcat源码中的命令模式和责任链模式。

# 命令模式（Command）

命令模式的主要作用就是封装命令，把发出命令的责任和执行命令的责任分开，达到解耦的效果。请求者与接收者之间没有直接引用关系，发送请求的对象只需要知道如何发送请求，而不必知道如何完成请求。命令模式一般包含一下角色：

- **Command: 抽象命令类**
- **ConcreteCommand: 具体命令类**
- **Invoker: 请求者**
- **Receiver: 接收者**
- **Client:客户类**

![Command模式结构](/image/designpatternCommand.jpg)

<!-- more -->

它们之间的调用关系大致如下图所示：

![Command模式时序图](/image/designpatternCommandSeq.jpg)

在Tomcat中，Connector和Container之间的调用就是典型的命令模式。

在[Tomcat如何找到正确的Servlet](http://rason.me/2015/12/01/How-do-Tomcat-find-the-specific-servlet/)这篇文章中曾经引用了一张请求处理时序图，现简要截图如下：

![Tomcat请求时序简图](/image/designpatterntomcatseq.png)

在这张图中，可以很显然地看出Connector与容器之间的信息交互通过了Http11Processor,这个Http11Processor就是具体的命令类。完整的关系如下：

- Client客户类，就是Tomcat的Server组件，创建请求者对象Connector。
- Connector作为请求者，通过ProtocolHandler（Http11Protocol实现了此接口）创建了具体命令Http11Processor并执行（Processor为抽象命令）。
- Http11Processor作为命令对象，交给接收者Container处理，

那么，这样做的意义是什么？**将一个请求封装为一个对象，从而使我们可用不同的请求对客户进行参数化；对请求排队或者记录请求日志，以及支持可撤销的操作。**命令可以以队列的方式进来，Container也可以以不同的方式来处理请求，如Http1.0协议和Http1.1的处理方式就不同。

命令模式在生活中的例子：**电视机遥控器**

电视机是请求的接收者，遥控器是请求的发送者，遥控器上有一些按钮，不同的按钮对应电视机的不同操作。抽象命令角色由一个命令接口来扮演，有三个具体的命令类实现了抽象命令接口，这三个具体命令类分别代表三种操作：打开电视机、关闭电视机和切换频道。显然，电视机遥控器就是一个典型的命令模式应用实例。

## 责任链模式（Chain of Responsibility）

责任链模式就是很多对象由每个对象对其下家的引用而连接起来形成一条链，请求在这条链上传递，直到链上的某个对象处理此请求，或者每个对象都可以处理此请求，并传递给下家，直到链上的每个对象都处理完。这样可以不影响客户端而能够在链上增加任意的处理节点。

责任链模式通常包含以下角色：

- **Handler(抽象处理者)**：处理请求的接口
- **ConcreteHandler（具体处理者）**：处理请求的具体类

Tomcat中的容器就是责任链模式，容器这方面的知识在Tomcat研究的时候已经比较熟悉了，在Tomcat请求时序图就完整地展示了责任链模式。

![Tomcat请求处理时序图](/image/tomcatrequest-process.png)

与标准的责任链模式不同的是，这里引入了Pipeline和Valve接口，扩展了这个链的功能，使得在链往下传的时候能够接收外界的干预。

## 总结

- **命令模式**：封装命令对象将调用者和接受者分离
- **责任链模式**：链式处理请求
