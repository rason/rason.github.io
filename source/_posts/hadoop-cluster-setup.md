---
title: Hadoop集群部署
date: 2017-07-03 15:17:17
categories: BigData
tags: Hadoop
description: Hadoop Cluster Setup
---


## 前言

上一篇文章学习了Hadoop的单机部署和伪分布式部署，今天来学习一个集群式部署。本文不包含高级主题，比如安全和高可用。

通常，集群中的一个机器被指定为NameNode，另一台机器被指定为ResourceManager。 这些是Master。 其他服务（如Web App Proxy Server和MapReduce Job History server）通常在专用硬件或共享基础架构上运行，具体取决于负载。

集群中的其余机器充当DataNode和NodeManager。 这些是Slaves。


## 机器分配

为了简单起见，我用三台机器来演示集群部署。如下所示：

| 10.16.1.24 | 10.16.1.25 | 10.16.2.28 |
| ------| ------ | ------ |
| 主机名：QHEE-T-Crawler | 主机名：QHEE-T-Crawler2 | 主机名：QHEE-T-CrawlerDB |
| NameNode | ResourceManager | SecondaryNameNode |
| DateNode | DateNode | DateNode |
| NodeManager | NodeManager | NodeManager |
| MRHistroyServer |  |  |

在配置文件 `/etc/hosts` 中添加主机名映射

```
10.16.1.24 QHEE-T-Crawler
10.16.1.25 QHEE-T-Crawler2
10.16.2.28 QHEE-T-CrawlerDB

```

## Hadoop集群配置

1.为了管理方便，将集群中的组件都安装配置在相同的目录。

```
[root@QHEE-T-Crawler /home/laixs/hadoop-2.8.0]# 
```

我使用的hadoop是2.8.0版本，都解压到`/home/laixs`目录中。

2.配置JDK。

修改`etc/hadoop/hadoop-env.shZ`， `etc/hadoop/mapred-env.sh` 和 `etc/hadoop/yarn-env.sh`三个配置文件指定JAVA_HOME

```
export JAVA_HOME="/usr/java/jdk1.8.0_131"
```

3.配置相关服务组件XML。

- `etc/hadoop/core-site.xm`配置NameNode

```
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://10.16.1.24:8020</value>
  </property>
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/home/laixs/hadoop-2.8.0/data/tmp</value>
  </property>
</configuration>

```

- `etc/hadoop/slaves`文件配置DataNode

```
10.16.1.24 QHEE-T-Crawler
10.16.1.25 QHEE-T-Crawler2
10.16.2.28 QHEE-T-CrawlerDB
```

- `etc/hadoop/hdfs-site.xml`文件配置SecondaryNameNode

```
<configuration>
   <property>
       <name>dfs.namenode.secondary.http-address</name>
       <value>10.16.2.28:50090</value>
   </property>
</configuration>

```

- `etc/hadoop/yarn-site.xml`文件配置YARN

```
<configuration>
   <property>
           <name>yarn.resourcemanager.hostname</name>
           <value>10.16.1.25</value>
   </property>
   <property>
           <name>yarn.nodemanager.aux-services</name>
           <value>mapreduce_shuffle</value>
   </property>
   <property>
           <name>yarn.log.aggregation-enable</name>
           <value>true</value>
   </property>
   <property>
           <name>yarn.log-aggregation.retain-seconds</name>
           <value>18000</value>
   </property>
</configuration>

```

具体每项配置解析可以看[yarn-default.xml](https://hadoop.apache.org/docs/r2.4.1/hadoop-yarn/hadoop-yarn-common/yarn-default.xml)。

- `etc/hadoop/mapred-site.xml`文件配置MapReduce

```
<configuration>
   <property>
           <name>mapreduce.framework.name</name>
           <value>yarn</value>
   </property>
   <property>
           <name>mapreduce.jobhistory.address</name>
           <value>10.16.1.24:10020</value>
   </property>
   <property>
           <name>mapreduce.jobhistory.webapp.address</name>
           <value>10.16.1.24:19888</value>
   </property>
</configuration>

```

<!-- more -->

## 测试

先在一台机器上测试各个服务，没问题之后再将配置文件同步到其它节点。然后再测试其它节点启动有没有问题，启动任何服务如果有问题都应该根据日志报出的异常来进行解决。

以下是各项服务启动命令，按照文章开头的机器作用分配启动相关服务。

- 启动namenode

```
$ sbin/hadoop-daemon.sh start namenode
```

- 启动datanode

```
$ sbin/hadoop-daemon.sh start datanode
```

- 启动secondarynamenode

```
$ sbin/hadoop-daemon.sh start secondarynamenode

```

- 启动resourcemanager

```
$ sbin/yarn-daemon.sh start resourcemanager
```

- 启动nodemanager

```
$ sbin/yarn-daemon.sh start nodemanager

```

- 启动historyserver 

```
$ sbin/mr-jobhistory-daemon.sh start historyserver
```

- 使用`jps`命令查看启动的进程：

```
[root@QHEE-T-Crawler /home/laixs/hadoop-2.8.0/etc/hadoop]# jps
29077 NodeManager
2566 Jps
2551 NameNode
31448 JobHistoryServer
28443 DataNode
```

- 测试HDFS能否创建目录、上传文件、读取文件

```
$ bin/hdfs dfs -mkdir /test

$ bin/hdfs dfs -put /etc/hosts /test/

$ bin/hdfs dfs -text /test/hosts
```

## 同步配置文件

初步测试地一台机器没问题的话就可以把配置文件同步到另外两台机器，然后启动相关服务进行测试。

```
$ scp -r etc/ root@10.16.1.25:/home/laixs/hadoop-2.8.0/
$ scp -r etc/ root@10.16.2.28:/home/laixs/hadoop-2.8.0/
```

当所有节点的服务都启动起来之后，可以通过管理界面`http://10.16.1.24:50070`查看DataNode状态:

![DataNode](/image/datanode.png)

## 测试MapReduce

在NameNode节点上跑一个MapReduce作业，可以在`http://10.16.1.25:8088`查看进度：

```
$ bin/yarn jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.5.0.jar wordcount /test/ /test/out
```

![YARN](/image/yarn.png)

作业执行完之后，可以通过`$ bin/hdfs dfs -cat /test/out/*`命令查看输出结果。

## 可能存在的问题

在启动机器2的时候出现`java.net.UnknownHostException: QHEE-T-Crawler2: QHEE-T-Crawler2: Name or service not known
`异常。

这是因为我一开始没有配置`/etc/hosts`文件，所有导致Host解析失败。

全文完。





