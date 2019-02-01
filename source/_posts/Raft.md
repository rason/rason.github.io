---
title: Raft
date: 2017-12-12 15:38:55
categories: Distributed
tags: Raft
description: 一致性协议Raft
---

## 前言

上篇文章说解决问题要**分而治之**，先把分片的问题解决了再考虑多个副本的**一致性问题**。那么什么是一致性问题？

> 因为同一数据存在多个副本，在并发的众多客户端读/写请求下，如何维护数据一致性视图非常重要，即在存储系统外部使用者看起来即使是多个副本数据，其表现也和单份数据一样。

简单来说，要使多个副本的数据一致，需要通过某种一致性协议来对副本进行更新。如果不按照某种一致性协议进行更新，数据就可能会出现不一致的情况。

比方说，两个副本A，B，有两个客户端分别对A，B进行不同的更新，这样两个副本的数据显然就不一致了。假如有某一种协议，多个客户端对副本的更新，只能在一个副本上更新，然后这个副本把更新操作同步到其它副本，成功之后再返回给客户端，这样就有可能实现多个副本的一致性。今天要学习的**Raft协议**就是类似这样的一种一致性协议。

当然，数据的一致性也可以分为强一致性和弱一致性，Raft协议是一种强一致性协议。

## Raft协议概述

Raft 是一种实现了分布式一致性的协议。假设我们有三个副本节点，来看一下Raft 协议如何工作：

### 节点状态

Raft 协议中的节点有三种状态：

- Follower
- Candidate
- Leader

在下面的图形用，分别用无边框，虚线边框，实线边框表示这三种状态。

### 领导选举

在开始阶段，所有节点都处于Follower 状态，如下图所示：

![初始阶段](https://raw.githubusercontent.com/rason/rason.github.io/master/image/raft-init.png)

如果Follower 没有收到Leader 发过来的信息，则可变成Candidate：

![成为Candidate](https://raw.githubusercontent.com/rason/rason.github.io/master/image/become-candidate.png)

然后Candidate 向其它节点发送投票自己为Leader 的请求，如果得到了大多数节点的投票响应，则成为Leader:

![成为Leader](https://raw.githubusercontent.com/rason/rason.github.io/master/image/become-leader.png)

这个过程称为**领导选举**，选出Leader 之后，系统的所有变更操作都需要通过Leader 节点。

<!-- more -->

### 日志复制

假设某个客户端(绿色圈圈)对某个数据进行更新：

![数据更新](https://raw.githubusercontent.com/rason/rason.github.io/master/image/update-data.png)

所有的变更请求先发送到Leader 节点，每个变更操作都会作为一条日志记录保存在Leader 节点上。

目前日志记录处于**未提交**状态，所以节点的数据目前还没有更新。

要提交日记记录，首先得先把日志记录复制到其它Follower 节点：

![日志复制](https://raw.githubusercontent.com/rason/rason.github.io/master/image/log-replica.png)

Leader 节点等待大多数节点已经完成日志复制的响应，然后才会把日志记录提交，Leader 节点的数据才会更新：

![Leader 更新](https://raw.githubusercontent.com/rason/rason.github.io/master/image/log-commited.png)

然后Leader 节点通知Follower 节点日志记录已经提交，Follower 节点也进行提交，数据达到一致性状态。

![Follower 更新](https://raw.githubusercontent.com/rason/rason.github.io/master/image/notify-commited.png)

这个过程就称为日志复制。

## 领导选举

Raft 协议中有两个超时设置用于领导选举：

- **选举超时**：每个Follower 都会等待一个超时时间，然后就可以成为 Candidate，这个超时时间就称为选举超时。选举超时时间一般随即分配为150ms到300ms之间。

![选举超时](https://raw.githubusercontent.com/rason/rason.github.io/master/image/election-timeout.png)

如上图所示，节点C 将会先到达超时时间，成为Candidate 然后开始一个新的选举任期(election term) :

![选举超时成为Candidate](https://raw.githubusercontent.com/rason/rason.github.io/master/image/timeout-candidate.png)

如上图所示，成为Candidate 的节点会将自己的term 加1，然后给自己投票，再发送请求投票信息到其他节点：

![发送投票请求](https://raw.githubusercontent.com/rason/rason.github.io/master/image/send-vote-request.png)

接收请求的节点在这个term 内如果还没有投票，则会将选票投给这个Candidate，然后重置自身的超时时间：

![投票](https://raw.githubusercontent.com/rason/rason.github.io/master/image/vote.png)

一旦Candidate 收到了大多数的投票，那么它将成为Leader 。

![成为Leader](https://raw.githubusercontent.com/rason/rason.github.io/master/image/vote-become-leader.png)

然后Leader 会发送`Append Entries` 消息给Follower，发送这个消息的时间间隔就是**心跳超时**(heartbeat timeout)。

- **心跳超时**： Leader 发送Append Entries信息的时间间隔。

![发送Append Entries 消息](https://raw.githubusercontent.com/rason/rason.github.io/master/image/append-entries-message.png)

Followers 将会响应每一条Append Entries 消息，然后重置自身的超时时间：

![响应Append Entries 消息](https://raw.githubusercontent.com/rason/rason.github.io/master/image/response-append-entries.png)

这个选举任期将会持续到一个Follower没有收到Leader 的心跳信息，然后成为Candidate。也就是说，Leader 会在任期期间不断发送心跳信息来重置其他的Follower的超时时间，让他们没有机会成为Candidate。只有当Leader 挂掉了，其他Follower 才有机会成为Candidate 进行新的选举。所以Raft 协议正是用**选举超时**和**心跳超时**

我们来看一下Leader 挂了，然后开始一个新的选举：

![Leader 挂了](https://raw.githubusercontent.com/rason/rason.github.io/master/image/leader-crash.png)

上图中，当Leader 挂掉之后，剩余的两个Follower 将会等待超时然后成为Candidate。由上面的超时时间可以看出，A 节点将会先超时，然后成为Candidate 将自己的term 加1，然后给自己投票，再发送请求投票信息到其他节点。当收到节点B 的投票请求之后成为新的Leader :

![新的Leader](https://raw.githubusercontent.com/rason/rason.github.io/master/image/new-leader.png)

需要得到大多数节点的投票才能成为Leader 保证了每个任期只有一个节点能成为Leader 。如果两个节点同时成为Candidate ，那么他们可能会得到一样多的投票，这种情况下将继续等待下一个超时，然后重新发起投票，直到选出一个Leader 为止。

## 日志复制

当选举出了新的Leader 之后，需要将系统中的所有变化复制到其他节点。这跟发送心跳信息的方式一样，也是通过发送`Append Entries`消息来完成这一动作，只是心跳信息中没有日志信息。我们来看一下这个过程：

首先，客户端发送更新操作到Leader 节点：

![变更操作](https://raw.githubusercontent.com/rason/rason.github.io/master/image/change-op.png)

变更会先被追加到Leader 节点的日志，然后在发送下一个心跳信息的时候将变更信息发送到Follower :

![发送变更信息](https://raw.githubusercontent.com/rason/rason.github.io/master/image/send-change.png)

当大多数Follower 确认这个信息，那么该变更日志将会被提交，然后Leader 将会响应给客户端：

![确认变更提交](https://raw.githubusercontent.com/rason/rason.github.io/master/image/entry-commit.png)

然后Leader 节点会在下一个心跳信息时通知Follower 节点日志记录已经提交，Follower 节点也进行提交，数据达到一致性状态。

### 网络分区

Raft 在面对网络分区的时候也能保持一致性。

![初始状态](https://raw.githubusercontent.com/rason/rason.github.io/master/image/partition-init.png)

假设A，B节点与C，D，E 节点形成了网络分区。C，D，E节点由于收不到原Leader B的心跳信息，那么他们之间将会选举出一个新的Leader ，但是这个Leader 的term 跟原来Leader B 的 term不一样，这时就会存在两个不同term 的Leader:

![两个Leader](https://raw.githubusercontent.com/rason/rason.github.io/master/image/two-leader.png)

现在，假设有两个客户端同时更新不同的Leader，客户端1将更新请求发送到Leader B 将某个值设置为3，Leader B 不能复制到大多数的节点（因为只有1个Follower节点），所以这个更新日志是处于未提交状态。

![客户端1更新](https://raw.githubusercontent.com/rason/rason.github.io/master/image/log-uncommited.png)

客户端2将某个值设置为8，因为它将会得到大多数节点的响应，所以这个更新操作能够成功：

![客户端2更新](https://raw.githubusercontent.com/rason/rason.github.io/master/image/client2-update.png)

现在，假设网络分区问题已经修复，这时节点B 会收到比他更高任期(term) 的信息，然后就会变成Follower：

![网络分区修复](https://raw.githubusercontent.com/rason/rason.github.io/master/image/parition-fix.png)

节点A和B 未提交的日志记录将会被回滚然后匹配新Leader 的日志：

![回滚匹配](https://raw.githubusercontent.com/rason/rason.github.io/master/image/roll-back.png)

现在，日志在集群中达成一致性。

## 总结

本文只是介绍了Raft 协议的大致流程，并没有关注很多细节问题。主要从领导选举和日志复制两个方面分析了Raft 协议的工作过程，领导选举重点是要了解两个超时时间，日志复制是Master-Slave模式，后续的日志一致性维护由领导者来完成。
