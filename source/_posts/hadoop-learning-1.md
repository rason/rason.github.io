---
title: Hadoop初体验
date: 2017-06-29 14:01:48
categories: BigData
tags: Hadoop
description: Hadoop hdfs mapreduce
---

## 背景

上篇文章学习了Flume框架，可以用于日志的收集，收集之后的日志需要有一个地方来存储。HDFS就是其中的一种方式，所以，顺着路线我需要学习一下HDFS。

HDFS只是Hadoop的一部分，解决分布式文件存储问题；另外重要的一部分就是MapReduce，解决分布式计算的问题。

今天这篇文章介绍如何设置和配置单节点Hadoop安装，以便你可以使用Hadoop MapReduce和Hadoop分布式文件系统（HDFS）快速执行简单的操作。

## 准备工作

- 系统：windows或者linux都可以，我的机器是linux
- JDK7以上
- ssh 
- rsync
- [hadoop](http://www.apache.org/dyn/closer.cgi/hadoop/common/)

## 运行Hadoop

解压下载的Hadoop发行版。修改`etc/hadoop/hadoop-env.sh`文件，添加JAVA_HOME路径：

```
 # set to the root of your Java installation
 export JAVA_HOME=/usr/java/latest
```

然后运行一下命令：

```
 $ bin/hadoop
```

会展示hadoop脚本的使用文档。

## 运行模式

现在，你可以使用三种支持的模式之一启动Hadoop集群：

- 单机模式
- 伪分布式
- 完全分布式

今天我们先学习前两种，后一种将在以后学习。

## 单机模式

默认情况下，Hadoop作为单一的Java进程以非分布式的模式运行。在Debug的时候很有用。

下面的例子将复制配置文件目录的内容作为输入，查找并显示匹配给定正则表达式的条目。输出写入到指定的 output 目录。

```
$ mkdir input
$ cp etc/hadoop/*.xml input
$ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.8.0.jar grep input output 'dfs[a-z.]+'
$ cat output/*

```

第三行命令是一个MapReduce 作业，查找匹配的内容输出。

<!-- more -->

## 伪分布式

Hadoop也可以以伪分布式模式运行在单节点上，其中每个Hadoop守护程序在单独的Java进程中运行。

### 配置

*etc/hadoop/core-site.xml:*

```
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
</configuration>
```

*etc/hadoop/hdfs-site.xml:*

```
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
```

### ssh免密配置

先检查一下ssh能否免密连接到localhost:

```
$ ssh localhost
```

如果不行，执行以下命令：

```
$ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
$ chmod 0600 ~/.ssh/authorized_keys
```

### 执行

下面的指令将在本地运行一个MapReduce作业。

1.格式化文件系统：

```
$ bin/hdfs namenode -format
```

2.启动NameNode和DataNode守护进程：

```
$ sbin/start-dfs.sh
```

Hadoop守护进程日志输入到`$HADOOP_LOG_DIR`目录（默认是 `$HADOOP_HOME/logs`）。

3.浏览NameNode的web界面，默认地址是：

```
http://localhost:50070/
```

4.创建运行MapReduce 作业所需要的HDFS目录

```
$ bin/hdfs dfs -mkdir /user
$ bin/hdfs dfs -mkdir /user/<username>
```

注意用户名不要搞错。

5.复制输入文件到分布式文件系统中：

```
$ bin/hdfs dfs -put etc/hadoop input
```

6.运行Hadoop发行版提供的一些MapReduce 例子：

```
$ bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.8.0.jar grep input output 'dfs[a-z.]+'
```

7.检验一下输出结果：将分布式文件系统中的output文件夹拷贝到本地进行检验：

```
$ bin/hdfs dfs -get output output
$ cat output/*
```

或者，你也可以直接在分布式文件系统中查看：

```
$ bin/hdfs dfs -cat output/*
```

8.作业完成之后，停止守护进程：

```
$ sbin/stop-dfs.sh
```

## 在单节点上运行YARN框架

你也可以通过设置几个参数并运行ResourceManager守护程序和NodeManager守护程序，在伪分布式模式下运行YARN上的MapReduce作业。

需要先执行上面的前四个步骤，然后执行下面指令：

1.配置参数

*etc/hadoop/mapred-site.xml*

```
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

*etc/hadoop/yarn-site.xml*

```
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

2.启动ResourceManager和NodeManager守护进程：

```
$ sbin/start-yarn.sh
```

3.浏览ResourceManager的web界面，默认地址是：

```
http://localhost:8088/
```

4.运行一个MapReduce作业

你可以选择再次执行上面例子的作业。

5.完成之后，停止守护进行：

```
$ sbin/stop-yarn.sh
```

## 可能出现的问题

按照教程跑第二第三个例子，如果多次执行了`$ bin/hdfs namenode -format`命令对namenode进行格式化，可能会导致datanode的clusterID跟namenode的clusterID不对应，然后datanode启动失败。

在启动日志中可以看到`java.io.IOException: Incompatible clusterIDs`的异常信息，解决方法很简单，只要让datanode的clusterID跟namenode的clusterID对应起来就行了。可以选择大家同时格式化掉从新来，也可以选择修改datanode的clusterID跟namenode的一致就行了。

## 总结

初次接触Hadoop，对HDFS和MapReduce有了基础的了解，并能够在本机上运行简单的MapReduce作业。


