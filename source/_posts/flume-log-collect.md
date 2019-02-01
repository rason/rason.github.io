---
title: 使用Flume将日志收集到HDFS
date: 2017-07-06 10:28:10
categories: BigData
tags: Flume
description: Flume HDFS
---

## 前言

本着从日志分析这个实际案例来学习Hadoop相关大数据技术，搜索了相关的日志分析方案，决定采用下图的方式：

![Hadoop log analysis](https://raw.githubusercontent.com/rason/rason.github.io/master/image/Log-Analysis-in-Hadoop.jpg)

那么第一步当然就是将日志收集到HDFS中，前面两篇博客已经简单地入门了Flume和Hadoop的部署，那么就在此基础上进行日志收集工作。

## Nginx 日志格式配置

本文以Nginx为例，采集其Access Log，因此为了后续工作的进行，先配置Nginx日志记录格式，如下所示：

定义格式：

```
log_format  main  '$host\t'
                      '$remote_addr\t'
                      '$remote_user\t'
                      '$time_local\t'
                      '$request_method\t'
                      '$request_uri\t'
                      '$server_protocol\t'
                      '$status\t'
                      '$body_bytes_sent\t'
                      '$http_referer\t'
                      '$http_user_agent\t'
                      '$request_time\t'
                      '$http_cookie\t'
                      '$sent_http_set_cookie\t'
                      '$upstream_addr\t'
                      '$upstream_cache_status\t'
                      '$upstream_response_time';

```

使用格式：

```
access_log  logs/xxx.access.log main;
```

## Flume配置

在Nginx机器上部署Flume，修改Flume按照目录下的`conf/example.conf`文件如下：

```

# Name the components on this agent
a1.sources = r1 
a1.sinks = k1 
a1.channels = c1
         
# Describe/configure the source
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /usr/local/nginx/logs/www.qhee.net.access.log

# Describe the sink 
a1.sinks.k1.type = hdfs
a1.sinks.k1.hdfs.fileType = DataStream
a1.sinks.k1.hdfs.path = hdfs://10.16.1.24:8020/flume/qhee/net

# Use a channel which buffers events in memory
a1.channels.c1.type = memory 
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
    
# Bind the source and sink to the channel
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1

```

数据源使用`exec`类型来获取最新的访问日志，然后将其保存到HDFS中,保存路径是上篇文章中部署的HDFS。然后启动Flume:

```
$ nohup bin/flume-ng agent --conf conf --conf-file conf/example.conf --name a1 &
```

<!-- more -->

## 启动Flume可能存在的问题

**1.** hadoop 依赖问题

```
Failed to start agent because dependencies were not found in classpath. Error follows.
java.lang.NoClassDefFoundError: org/apache/hadoop/io/SequenceFile$CompressionType
```

解决办法：将`/hadoop/share/hadoop/common/*.jar` 和`/hadoop/share/hadoop/common/lib/*.jar` 拷贝到flume安装目录的lib下。

**2.** hdfs 依赖问题

```
java.io.IOException: No FileSystem for scheme: hdfs
```

解决办法：将`/hadoop/share/hadoop/hdfs/hadoop-hdfs-client-x.x.x.jar` 拷贝到flume的安装目录的lib下。

**3.** 没有权限写 hdfs

```
org.apache.hadoop.security.AccessControlException: Permission denied: user=xxx, access=WRITE,
```

解决办法：使用管理账户开启hdfs写权限或者配置conf/hdfs-site.xml关闭权限控制

```
<property>
  <name>dfs.permissions</name>
  <value>false</value>
</property>
```

## 查看HDFS收集结果

启动Flume如果没有问题，那么在Hadoop的控制台页面应该能看到收集的日志，如下图所示：

![Collected Log](https://raw.githubusercontent.com/rason/rason.github.io/master/image/flume-log.png)

## 总结

本文比较简单，都是一些配置工作，先配置Nginx的日志记录格式，然后配置Flume将日志收集到HDFS中。接下来将会继续学习使用Pig将日志结构化，然后将其存储到Hive数据仓库中。
