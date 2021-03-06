title: 路由协议
categories: Network
date: 2016-05-05 14:34:50
tags: Network
description: 路由协议
---

## 前言

前后花了一个月左右时间终于把《图解TCP/IP》一书浏览了一遍，看的不是很深入，最后一篇笔记就记录一下路由协议方便的知识点。最后两章应用层协议和网络安全没什么实际的内容，就此略过了。等有空了还是得通读一下《详解TCP/IP》卷一、二、三，《图解TCP/IP》还是写的太基础了，学的不是很扎实。

## 路由控制

### IP地址与路由控制

互联网由路由器连接的网络组合而成的。为了能让数据包能够正确地达到目标主机，路由器必须在图中进行正确的转发，这种向“正确方向”转发数据所进行的处理就叫做**路由控制或路由**。

### 静态路由与动态路由

**路由器根据路由控制表转发数据包。**

那么，怎么制作和管理路由控制表则是路由的关键了，这里有两种方式：静态和动态。

**静态路由**是值事先设置好路由器和主机并将路由信息固定的一种方式。可想而知，这种方式只适合在小规模的网络中，大规模的网络用人工维护的话难度就太大了。

**动态路由**可以让路由器自动学习到其他路由器的网络，并且网络拓扑发生改变后自动更新路由表。网络管理员只需配置动态路由协议即可。

<!-- more -->

## 路由控制范围

随着IP网络的发展，想要对所有网络统一管理是不可能的事。因此，人们根据路由控制范围常使用**IGP(Interior Gateway Protocol)**和**EGP(Exterior Gateway Protocol)**两种类型的路由协议。

这就好比国家与国家之间的交流有对外政策，国家内部的治理有对内政策，这样管理起来逻辑就简单清晰很多了。

### 自治系统与路由协议

企业内部的管理方针，往往由企业内部自行决定。因此每个企业或组织机构对网络管理和运维的方法都不尽相同。

制定自己的路由策略，并以此为准在一个或多个网络群体中采用的小型单位叫做**自治系统（AS：Autonomous System）**或**路由选择域(Routing Domain)**。

![EGP与IGP](/image/EGP&IGP.png)

上图中，区域网络与ISP（互联网服务提供商）就是自治系统的典型例子。

自治系统内部动态路由采用的协议是域内路由协议，即IGP。而自治系统之间的路由控制采用的是域间路由协议，即EGP。

### IGP与EGP

如前面所述，路由协议大致分为两大类：

- **外部网关协议EGP**
- **内部网关协议IGP**

这就好比IP地址中网络地址与主机地址之间的关系，根据EGP在区域网络之间（或ISP之间）进行路由选择，也可以根据IGP在区域网络内部（或ISP内部）进行主机识别。

由此，路由协议被分为EGP和IGP两个层次。**没有EGP就不可能有世界上各个不同组织机构之间的通信，没有IGP机构内部也就不可能进行通信**。

IGP中还可以使用**RIP**（Routing Information Protocol，路由信息协议）、**RIP2**、**OSPF**（Open Shortest Path First，开放式最短路径优先）等众多协议。与之相对，EGP使用的是**BGP**（Border Gateway Protocol，边界网关协议）协议。

## 路由算法

路由控制有各种各样的算法，其中最具代表性的有两种，**距离向量（Distance-Vector）算法**和**链路状态（Link-State）算法**。

### 距离向量算法

距离向量算法是指**根据距离和方向决定目标网络或目标主机位置**的一种方法。

路由器之间可以互换目标网络的方向及其距离的相关信息，并以这些信息为基础制作路由控制表。当网络构造复杂时，需要**消耗一定时间**，也极易**发生路由循环**等问题。

### 链路状态算法

链路状态算法是**路由器在了解网络整体连接状态的基础上**生成路由控制表的一种方法。该方法中，每个路由器必须保持**同样的信息**才能进行正确的路由选择。

**距离向量算法**中每个路由器掌握的信息**都不相同**。通往每个网络所耗的距离也根据路由器的不同而不同。因此，该算法的一个缺点就是不太容易判断每个路由器上的信息是否正确。

而链路状态算法所有路由器保持相同的信息，即使网络结构变得复杂，也能保持正确的路由信息，进行稳定的路由选择。付出的代价就是**如何从网络代理获取路由信息表。**

### 主要的路由协议

路由协议分很多种，下表列出了主要的几种路由协议。

| 路由协议名 | 下一层协议 | 方式 | 适用范围 |循环检测 |
| :------: | :------: | :---: | :-------: | :-----: | 
| RIP  | UDP | 距离向量 | 域内    | 不可以 |
| RIP2 | UDP | 距离向量 | 域内    | 不可以 |
| OSPF | IP  | 链路状态 | 域内    | 可以   |
| BGP  | TCP | 路径向量 | 对外连接 | 可以  |

## RIP

RIP是距离向量型路由协议，**广泛用于LAN**。

### 广播路由控制信息

RIP将路由控制信息**定期（30秒一次）向全网广播**。如果没有收到路由控制信息，连接就会被断开。不过，这有可能是由于丢包导致的，因此RIP规定等待5次，如果等了6次（180秒）仍未收到路由信息，才会真正关闭连接。

### 根据距离向量确定路由

RIP基于距离向量算法决定路由。距离的单位为“跳”，跳数是值所经过路由器的个数。RIP希望**尽可能少通过路由器**将数据包装法到目标IP地址。

### RIP中路由变更时的处理

RIP的基本行为可归纳为如下两点：

- 将自己所知道的路由信息定期进行广播。
- 一旦认为网络被断开，数据将无法通过此路由器，其它路由器也就可以得知网络已经断开。

不过，在网络断开时接收到自己发出去的消息，这样会存在**无限计数**的问题，为了解决这个问题可以采取以下两种方法：

- 最长距离不超过16.
- 规定路由器不再把所收到的路由信息原路返回给发送端。也称作水平分割。

然而，这种方法对有些网络来说是无法解决的，比如在网络本身就有环路的情况下。为了尽可能解决这个问题，任务提出**毒性逆转（Poisoned Reverse）**和**触发更新（Triggered Update）**两种方法。

毒性逆转是指当网络中发生链路被断开的时候，不是不再发送这个消息，而是将这个无法通信的消息传播出去，即发送一个距离为16的消息。

触发更新是指当路由信息发生变化时，不等待30秒而是立即发送出去的一种方法。

## RIP2

RIP2就是RIP的第二版，是一种改良协议，具有一下特点：

- 使用多播
- 支持子网掩码
- 路由选择域
- 外部路由标志
- 身份验证密钥

## OSPF

OSPF协议是“开放式最短路径优先(Open Shortest Path First)”的缩写，属于链路状态路由协议。

OSPF提出了“区域（area）”的概念，每个区域中所有路由器维护着一个相同的链路状态数据库（LSDB）。区域又分为骨干区域（骨干区域的编号必须为0）和非骨干区域（非0编号区域），如果一个运行OSPF的网络只存在单一区域，则该区域可以是骨干区域或者非骨干区域。如果该网络存在多个区域，那么必须存在骨干区域，并且所有非骨干区域必须和骨干区域直接相连。如下图所示：

![AS与区域](/image/AS&AREA.png)

OSPF利用所维护的链路状态数据库，通过最短路径优先算法（SPF算法）计算得到路由表。OSPF的收敛速度较快。由于其特有的开放性以及良好的扩展性，目前OSPF协议在各种网络中广泛部署。

## BGP

BGP（Border Gateway Protocol），边界网关协议是连接不同组织机构（或者说连接不同自治系统）的一种协议。因此，它属于外部网关协议（EGP）。只有BGP、RIP和OSPF共同进行路由控制，才能够进行整个互联网的路由控制。如下图所示：

![BGP使用AS号管理网络信息](/image/BGP.png)

ISP、区域网络等会将每个网络域编配成一个个自治系统进行管理。它们为每个自治系统分配一个16比特的AS编号。BGP就是根据这个编号进行相应的路由控制。

## 总结

本文简单介绍了什么是路由控制、静态路由和动态路由概念，路由控制范围、路由算法、路由协议（内部网关协议和外部网关协议）等基本知识，只是达到了知道了这个概念的水平，并无深入知识。
