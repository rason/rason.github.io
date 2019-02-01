---
title: I/O Model
date: 2017-11-06 15:28:37
categories: OS
tags: IO
description: 五种IO模型
---

## 前言

上篇文章大致看了一下Thrift 的工作过程，说到底核心就是根据消息协议进行通信。脑补了一下如果自己来实现会怎么做，想着想着就想到网络通信和I/O这一块了。好吧，就先研究一下I/O模型，我管这种学习方法为**缘分学习法**。学习这东西感觉就是一个无底洞，越挖发现东西越多。

## 五种I/O模型

其实之前也有大概看过这方面的知识，什么同步异步，什么阻塞非阻塞，当时看得一团糟。今天再次梳理一下：

- 同步和异步是根据数据从内核空间到用户空间是否需要等待来区别的，等就是同步，不等就是异步。
- 阻塞和非阻塞是根据数据在就绪之前是立即返回还是等待，即发起I/O请求是否会被阻塞来区别的，等就是阻塞，不等就是非阻塞。

### 同步阻塞

![同步阻塞](https://raw.githubusercontent.com/rason/rason.github.io/master/image/bio.png)

上图中 `recvfrom` 方法等待数据就绪，也等待数据从内核空间拷贝到用户空间，因此是同步阻塞模型。

### 同步非阻塞

![同步非阻塞](https://raw.githubusercontent.com/rason/rason.github.io/master/image/nio.png)

上图中不断调用 `recvfrom` 看数据就绪了没有，会立即返回没有阻塞；当数据就绪后会等待从内核空间拷贝到用户空间，属于同步，因此这是同步非阻塞模型。通过对socket设置O_NONBLOCK可实现同步非阻塞。

### I/O复用

上面的模型虽然是非阻塞，但是还是会不断去询问数据就绪了没有，好像也没啥卵用。所以就出现了I/O复用，就是先向操作系统注册自己感兴趣的一大堆数据，然后统一问操作系统有哪几个是已经准备好的了。

![I/O复用](https://raw.githubusercontent.com/rason/rason.github.io/master/image/mulio.png)

上面的 `select` 方法其实也是阻塞的，但是他会返回批量已经准备好的数据，所以效率还是挺高的。 `recvfrom` 方法是同步的，需要等待系统进行数据复制，这个操作是纯CPU 操作，速度并不慢。

### 信号驱动

![信号驱动](https://raw.githubusercontent.com/rason/rason.github.io/master/image/sigio.png)

此模型利用信号的模型来通知用户进程数据是否就绪，但是Linux中信号队列是有限制的，如果超过这个数字问题就无法读取数据。

### 异步非阻塞

![异步非阻塞](https://raw.githubusercontent.com/rason/rason.github.io/master/image/aio.png)

这个模型不用等数据就绪，也不用等数据拷贝，直接准备好数据给你。数据异步非阻塞。
