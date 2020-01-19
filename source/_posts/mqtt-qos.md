---
title: MQTT QoS
date: 2020-01-14 19:56:48
tags:
---

最近做了个物联网的项目，用到了 MQTT，里面有个 QoS 的概念理解起来可能会有一点困扰，今天花了点时间梳理了一下其中的逻辑，写篇文章记录一下。

### 什么是 QoS ?

QoS 的全称是 quality of service。描述的是消息发送方和消息接收方之间消息传递的保证级别。MQTT 中有 3 个 QoS 级别：

- At most once (0)
- At least once (1)
- Exactly once (2)

MQTT 发布消息 QoS 保证不是端到端的，是客户端与服务器之间的。订阅者收到 MQTT 消息的 QoS 级别，最终取决于发布消息的 QoS 和主题订阅的 QoS。

| 发布消息的QoS | 主题订阅的QoS | 接收消息的QoS |
| :-----:| :----: | :----: |
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 0 | 2 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |
| 1 | 2 | 1 |
| 2 | 0 | 0 |
| 2 | 1 | 1 |
| 2 | 2 | 2 |

### QoS 0 - at most once

最小 QoS 级别为 0 。此服务级别保证了 *尽力而为* 的投递，不保证一定投递成功。接收方不会回复确认消息，发送方也不存储和重新发送消息。

| 发送方 | 报文流向 | 接受方 |
| :-----:| :----: | :----: |
| PUBLISH QoS = 0, DUP = 0 |  |  |
|  | —> |  |
|  |  | 接收消息（可能不会收到）并处理 |

### QoS 1 - at least once

| 发送方 | 报文流向 | 接受方 |
| :-----:| :----: | :----: |
| 存储消息 |  |  |
| 发送 PUBLISH QoS1, DUP = 0，带有 Packetld | |  |
|  | —> |  |
|  |  |接收消息并处理  |
|  |  | 发送带有 Packetld 和 PUBACK 确认报文 |
|  | <--- |  |
| 丢弃消息 |  |  |

只要发送方没有收到 `PUBACK` 回复报文，消息还会被重新发送，所有 QoS 1 可能会出现消息重复的问题。

需要注意的是，接收方在接收 QoS 1 消息时并不会幂等处理，也就是说消息重发也认为是一条新的消息（至于 DUP 标志，协议里有说 “DUP = 1“ 时，也不能假设认为之前收到过该报文）。

### QoS 2 - exactly once

| 发送方 | 报文流向 | 接受方 |
| :-----:| :----: | :----: |
| 存储消息 |  |  |
| 发送 PUBLISH QoS2, DUP = 0，带有 PacketId ||  |
|  | —> |  |
|  |  |存储 PacketId 和消息  |
|  |  | 发布带有 Packetld 和 Reason Code 的 PUBREC 报文|
|  | <--- |  |
| 丢弃存储的消息，存储接收到的带有相同 PacketId 的 PUBREC 报文 |  |  |
| 发送 PUBREL 报文 |  |  |
|  | —> |  |
|  | | 处理消息，然后丢弃 PacketId |
|  | | 发送带有 PacketId 的 PUBCOMP 报文 |
|  | <--- |  |
| 丢弃存储的状态 | | |

QoS 2 应该是最难理解的了，涉及了四次交互，如下图所示

![QoS 2](https://raw.githubusercontent.com/rason/rason.github.io/master/image/qos2.png)

简单来说，QoS 2 有点像分布式事务中 **两阶段提交** 的解决方案，第一阶段的请求响应只是将消息保存起来的，但还不能作为正式的消息去处理，相当于一个 **半提交** 的状态，第二阶段就是确认消息提交，可以进行处理了。

QoS 2 保证只到达了一次是因为在这四次交互之中根据 PacketId 做了幂等处理，但是在处理消息之后 PacketId 会被丢弃，也就是说在发起新一轮的四次交互 PacketId 是可以重用的。至于 QoS 2 怎么实现幂等处理，大家找个 MQTT 客户端的源码看一下。

对于 QoS 2 如果还有疑问的可以继续看下下面两篇文章：

- [Understanding MQTT QOS Levels- Part 2](http://www.steves-internet-guide.com/understanding-mqtt-qos-2/)
- [MQTT协议中QoS2为什么要四次包交互？](https://www.zhihu.com/question/54000916)

当然，源码是最好的老师。







