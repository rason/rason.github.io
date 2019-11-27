---
title: Spring Kafka AckMode
date: 2019-06-20 17:36:49
tags:
---

## enable-auto-commit

1. 当 enable-auto-commit 为 true 时, 会根据 auto-commit-interval 配置的时间间隔去自动 commit, 就算 record 被消费异常也会自动 commit.
2. 当 enable-auto-commit 为 false 时, 需要根据 listener 的 ack-mode 来确定确认模式.

## ack-mode

#### record

在 listener 处理每条消息之后提交, 即处理一条提交一条.

#### batch

在下一次 poll 之前提交已经处理完的记录.

#### time

按照时间间隔来提交, 单位为毫秒.

#### count

累积到 count 数目时提交.

#### count_time

到了时间或者到了累积的数目时提交.

#### manual

手工提交, 需要在业务代码中调用 Acknowledgment.acknowledge() 提交, 调用之后，就是跟 batch 同理处理。

#### manual_immediate

调用 Acknowledgment.acknowledge() 之后立马提交.

## 注意

如果你想业务代码发生异常时进行某些处理再提交, 你可能需要将 enable-auto-commit 设置为 false, 然后 ack-mode 设置为手工提交两种模式之一, 其他情况, 都会按照一定方式自动提交.
