title: 网络基础知识
categories: Network
date: 2016-04-13 09:47:48
tags: Network
description: 网络基础知识
---

## 前言

前段时间大概浏览了一下大型网站技术架构一书，在学习的过程中发现自己的基础知识还是不太扎实，所以最近想补充一下自身的基础知识。首先是打算从网络这方面入手，先从简单的《图解TCP/IP》开始，所以接下来的博客会记录一下该书的读书笔记。

## 协议的分层

OSI模型将通信协议进行了分层，目的是为了让比较复杂的网络协议更加简单化。每个分层都接收由它下一层提供的特定服务，并且为自己的上一层提供特定的服务。上下层之间进行交互时所遵循的约定叫做“接口”，同一层之间的交互所遵循的约定叫做“协议”。如下图所示：

![分层模型](/image/networkprotocol-level.png)

<!-- more -->

协议的分层跟软件开发中的模块化作用类似，通过分层将每层独立使用，即使系统中某些分层变化，也不会波及整个系统。因此，可以构造一个扩展性和灵活性都较强的系统。

此外，通过分层能够细分通信功能，更易于单独实现每个分层的协议，并界定每个分层的具体责任和义务。

## OSI模型

![OSI模型](/image/networkprotocol-level-func.png)

- **应用层**：为应用程序提供服务并规定应用程序中通信相关的细节。
- **表示层**：将设备固有的数据格式转换为网络标准传输格式，因为不同设备对同一比特流解析的结果可能会不同。
- **会话层**：复杂建立和断开通信连接，以及数据的分割等数据传输相关的管理。
- **传输层**：可靠传输，只在通信双方节点上进行处理，无需在路由器上处理。
- **网络层**：将数据传输到目标地址，负责寻址和路由选择。
- **数据链路层**：将0,1序列划分为具有意义的数据帧传送给对端（数据帧的生成与接收）。
- **物理层**：负责0,1比特流（0,1序列）与电压的高低，光的闪灭之间的互换。

## OSI模型通信处理

- 发送方从第7层至第1层由上至下按照顺序传输数据，而接收端则从第1层至第7层由下至上传输数据。
- 每个分层上，在处理上一层传过来的数据时可以附上当前分层的协议所必须的”首部“信息。
- 接收端对收到的数据进行数据”首部“与”内容“的分离，再转发给上一层，并最终将发送端的数据恢复为原状。

![OSI模型通信](/image/networkprotocol-connection.png)

## 传输方式的分类

按照不同的分类方法会有不同的分类。

- **面向有连接与面向无连接**
- **电路交换与分组交换**
- **根据接收端分离：单播、广播、多播、任播**

## 网络的构成要素

![网络设备及其作用](/image/networknetwork-device.png)

- **中继器**：工作在OSI模型第1层，对减弱的信息进行放大和发送的设备。
- **网桥**：工作在OSI模型第2层，根据数据帧的内容转发数据给相邻的其他网络，根据MAC地址处理。
- **路由器**：工作在OSI模型第3层，可以将分组报文发送给另一个目标路由器地址，根据IP地址处理
- **4~7层交互机**：以TCP等协议的传输层及其上面的应用层为基础，分析收发数据，并对其进行特定的处理。负载均衡器就是一中。
- **网关**：不仅转发数据还负责对数据进行转换，在两个不能进行直接通信的协议之间进行翻译，最终实现两者之间的通信。比如互联网邮件与手机邮件之间的转换服务。再比如代理服务器和防火墙。

## 总结

都是一些基础的概念，先对基础有大概的认识。重点内容是OSI的七层模型，文中没有展开的知识点需要自行搜索了解。
