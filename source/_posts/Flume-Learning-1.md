---
title: Flume初体验
date: 2017-06-27 14:24:48
categories: BigData
tags: Flume
description: Flume入门
---

## 概述

最近想学点大数据，大概浏览了一下所涉及的框架知识，就先从数据收集框架开始学习。

Apache Flume是一种分布式，可靠和可用的系统，用于高效收集，聚合和将大量日志数据，从许多不同的来源移动到集中式数据存储。

使用Apache Flume不仅限于日志数据聚合。 由于数据源是可定制的，因此可以使用Flume 来传输大量事件数据，包括但不限于网络流量数据，社交媒体生成数据，电子邮件消息以及几乎任何数据源。

## 数据流模型

**Flume事件**被定义为具有字节有效载荷和可选的一组字符串属性的数据流的单位。 Flume Agent是一个（JVM）进程，它承载事件从外部源传递到下一个目标（跳）的组件。

![Flume](/image/flume1.png)


Flume 消费由外部Source（如Web服务器）传递给它的事件。外部源将Flume 的事件以目标Flume Source识别的格式发送到Flume。

例如，可以使用Avro Flume Source 从Avro客户端或其他Flume Agent 发送的Avro事件。

类似的流程可以使用Thrift Flume Source 来定义，以接收来自Thrift Sink或Flume Thrift Rpc客户端的事件，或者使用从Flume thrift 协议生成的任何语言编写的Thrift客户端。

当Flume Source 接收到事件时，它将其存储到一个或多个Channel。该Channel是一个被动存储，保持事件，直到它被Flume Sink 消费。文件Channel 是一个例子 - 它由本地文件系统支持。Sink 从通道中删除事件，并将其放入外部存储库，如HDFS（通过Flume HDFS Sink）或将其转发到流中下一个Flume Agent（下一跳）的Flume Source。

<!-- more -->

## 起步

### 配置Agent

Flume Agent配置存储在本地配置文件中。 这是一个遵循Java属性文件格式的文本文件。 可以在同一配置文件中指定一个或多个Agent的配置。 配置文件包括Agent中每个Source，Sink 和Channel 的属性，以及它们如何连接在一起以形成数据流。

### 配置单独组件

数据流中的每个组件（Channel，Sink 或 Channel）具有名称、类型和特定类型的一系列属性。 例如，Avro Source 需要一个主机名（或IP地址）和一个端口号来接收数据。 内存Channel 可以具有最大队列大小（“容量”），并且HDFS Sink需要知道文件系统URI，创建文件的路径，文件轮换的频率（“hdfs.rollInterval”）等。组件的所有这些属性需要在托管Flume Agent的属性文件中设置。

### 将组件串联起来

Flume Agent 需要知道要加载的组件以及它们如何连接起来。通过在Agent配置文件中列出每个Source,Sink 和Channel的名字，然后指定每个Sink和 Source连接的Channel 来完成。

### 启动一个Agent

Agent通过Flume发行版bin目录下的flume-ng脚本来启动。需要指定Agent的名字、配置文件目录和配置文件。

```
$ bin/flume-ng agent -n $agent_name -c conf -f conf/flume-conf.properties.template
```

这样Agent就会启动运行在配置文件中配置的Source 和 Sink。

## 一个简单的例子

下面是一个单Flume节点部署配置文件的例子。此配置允许用户生成事件，并随后将其记录到控制台。

```
# example.conf: A single-node Flume configuration

# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# Describe the sink
a1.sinks.k1.type = logger

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

该配置文件定义了一个叫做a1的Agent。a1有一个Source 监听端口44444的数据，一个Channel 缓冲事件数据在内存中，和一个Sink 记录事件数据到控制台。配置文件命名各种组件，然后配置其类型和配置参数。 给定的配置文件可以定义几个命名的Agent; 当给定的Flume 进程启动时，会传递一个标志，告诉它哪个命名的Agent 程序可以显示。

给定这个配置文件，我们可以启动Flume，如下所示：

```
$ bin/flume-ng agent --conf conf --conf-file conf/example.conf --name a1 -Dflume.root.logger=INFO,console
```

请注意，在完全部署中，我们通常会再包含一个选项：`--conf = <conf-dir>`。 `<conf-dir>`目录将包含shell脚本flume-env.sh和潜在的log4j属性文件。 在这个例子中，我们传递一个Java选项来强制Flume登录到控制台，我们没有一个自定义的环境脚本。

从一个单独的终端，我们可以telnet端口44444并发送Flume一个事件：

```
$ telnet localhost 44444
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello world!
OK

```

在启动Flume 的终端中我们可以在日志信息中输出的事件。

```
2017-06-27 16:26:32,909 (lifecycleSupervisor-1-1) [INFO - org.apache.flume.source.NetcatSource.start(NetcatSource.java:155)] Source starting
2017-06-27 16:26:32,934 (lifecycleSupervisor-1-1) [INFO - org.apache.flume.source.NetcatSource.start(NetcatSource.java:169)] Created serverSocket:sun.nio.ch.ServerSocketChannelImpl[/127.0.0.1:44444]
2017-06-27 16:27:26,920 (SinkRunner-PollingRunner-DefaultSinkProcessor) [INFO - org.apache.flume.sink.LoggerSink.process(LoggerSink.java:95)] Event: { headers:{} body: 68 65 6C 6C 6F 20 77 6F 72 6C 64 21 0D          hello world!. }

```

恭喜——你已经成功地配置和部署了一个Flume Agent! 在接下来的文章中我们将更详细地介绍Agent 的配置。

