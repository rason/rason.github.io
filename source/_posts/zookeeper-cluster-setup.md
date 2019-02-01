---
title: ZooKeeper集群搭建
date: 2017-07-11 10:42:32
categories: BigData
tags: ZooKeeper
description: ZooKeeper集群搭建
---

## 前言

将Flume事件存储在Kafka群中可以实现系统的高可用，Kafka需要用到ZooKeeper，因此我们需要先将ZooKeeper集群搭建起来。

ZooKeeper是一个开源的分布式协调系统。暴露简单的接口可以让我们来实现一些高级的功能，比如服务同步，配置管理，组管理和命名服务等。

关于ZooKeeper的应用场景和原理本文就不展开讨论，网上可以搜到很多信息，本文只介绍ZooKeeper集群的部署。

## 单机模式

搭建单机模式的ZooKeeper服务相当简单：

1.首先[下载](http://zookeeper.apache.org/releases.html)一个稳定的版本，然后解压。

2.在ZooKeeper跟目录下创建`conf/zoo.cfg`文件。

```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
```

- **tickTime** ： 基本时间单元，以毫秒为单位。用于心跳检测，也就是每个tickTime就会发送一个心跳。
- **dataDir** ： 保存内存快照数据的位置。
- **clientPort** ： 监听客户端连接的端口。

3.启动ZooKeeper

```
bin/zkServer.sh start
```

我们可以在根目录`zookeeper.out`文件中看到启动日志。

## 管理ZooKeeper存储

现在，我们可以简单地测试一下ZooKeeper的存储。

### 连接到ZooKeeper

```
$ bin/zkCli.sh -server 127.0.0.1:2181
```

控制台可以看到一些输出信息

```
Connecting to 127.0.0.1:2181
2017-07-10 15:54:33,803 - INFO  [main:Environment@97] - Client environment:zookeeper.version=3.3.6-1366786, built on 07/29/2012 06:22 GMT
...
Welcome to ZooKeeper!
JLine support is enabled
```

接着输入`help`命令可以看到一些使用介绍：

```
ZooKeeper -server host:port cmd args
	stat path [watch]
	set path data [version]
	ls path [watch]
	delquota [-n|-b] path
	ls2 path [watch]
	setAcl path acl
	setquota -n|-b val path
	history 
	redo cmdno
	printwatches on|off
	delete path [version]
	sync path
	listquota path
	get path [watch]
	create [-s] [-e] path data acl
	addauth scheme auth
	quit 
	getAcl path
	close 
	connect host:port
```

<!-- more -->

我们可以先看一下根目录下有什么：

```
[zk: 127.0.0.1:2181(CONNECTED) 1] ls /
[zookeeper]
```

下一步，我们创建一个新的znode:

```
[zk: 127.0.0.1:2181(CONNECTED) 4] create /zk_test my_data
Created /zk_test
```

再次查看根目录：

```
[zk: 127.0.0.1:2181(CONNECTED) 5] ls /
[zookeeper, zk_test]
```

获取一下`zk_test`的数据：

```
[zk: 127.0.0.1:2181(CONNECTED) 6] get /zk_test
my_data
cZxid = 0x2
ctime = Mon Jul 10 16:00:16 CST 2017
mZxid = 0x2
mtime = Mon Jul 10 16:00:16 CST 2017
pZxid = 0x2
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 7
numChildren = 0
```

修改`zk_test`的数据：

```
[zk: 127.0.0.1:2181(CONNECTED) 7] set /zk_test junk
cZxid = 0x2
ctime = Mon Jul 10 16:00:16 CST 2017
mZxid = 0x3
mtime = Mon Jul 10 16:01:52 CST 2017
pZxid = 0x2
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 4
numChildren = 0
[zk: 127.0.0.1:2181(CONNECTED) 8] get /zk_test
junk
cZxid = 0x2
ctime = Mon Jul 10 16:00:16 CST 2017
mZxid = 0x3
mtime = Mon Jul 10 16:01:52 CST 2017
pZxid = 0x2
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 4
numChildren = 0
```

最后，删除该节点：

```
[zk: 127.0.0.1:2181(CONNECTED) 9] delete /zk_test
[zk: 127.0.0.1:2181(CONNECTED) 10] ls /
[zookeeper]
```

## 集群模式

单机模式运行的ZooKeeper便于评估，开发和测试。 但在生产中，您应该以集群模式运行ZooKeeper。在集群模式下，群组中的所有服务器都具有相同配置文件的副本。

1.集群配置文件`conf/zoo.cfg`跟单机配置类似，只是有点点不同：

```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=5
syncLimit=2
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
```

- **initLimit** : 表示群组中的ZooKeeper服务器初始化时连接到Leader最大限度时间，即N个心跳时间内还没连接上Leader则表示连接失败。

- **syncLimit** : 表示和Leader发送消息，请求和应答时间长度。

- **server.X** ： 表示组成ZooKeeper服务的机器列表，注意这里的X需要在`myid`文件中指明。比如机器2中，`/var/lib/zookeeper/myid`的内容为2，表明自身是哪台机器。

- **2888:3888** ： 第一个端口是集群成员互相交流的端口，比如集群成员需要商定更新的顺序。第二个端口用于Leader选举。

2.配置Host

上面的配置文件使用了Host的方法类表明主机，因此我们还需要修改`/etc/hosts`文件来进行主机映射：

```
10.16.1.25 zoo1
10.16.1.24 zoo2
10.16.2.28 zoo3

```

3.将安装文件复制到集群的其他机器

```
$ scp -r zookeeper-3.3.6/ root@10.16.1.24:/home/laixs/zookeeper-3.3.6/
$ scp -r zookeeper-3.3.6/ root@10.16.2.28:/home/laixs/zookeeper-3.3.6/

```

注意其它机器也要修改`/etc/hosts`文件。

4.设置myid

```
在10.16.1.25机器中输入命令： echo "1" > /var/lib/zookeeper/myid
在10.16.1.24机器中输入命令： echo "2" > /var/lib/zookeeper/myid
在10.16.2.28机器中输入命令： echo "3" > /var/lib/zookeeper/myid
```

5.启动集群

分别在集群中的各台机器上执行：

```
$ bin/zkServer.sh start
```

6.验证

分别在集群中的各台机器上执行：

```
$ bin/zkServer.sh status

```

三台机器中，有两台输出`Mode: follower`，一台输出`Mode: leader`，这样就代表集群启动成功了。

如果失败，请查看启动日志。

## 总结

ZooKeeper集群的配置还是比较简单的，按照文档一步步操作下来应该就没什么问题，哪步出现了问题看看日志应该就能解决。
