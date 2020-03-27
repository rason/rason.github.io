---
title: 容易被误会的 kafka auto commit
date: 2020-03-27 15:27:00
tags:
---

与 kafka auto commit 两个配置：

- `enable.auto.commit`：是否开启自动提交
- `auto.commit.interval.ms`：自动提交时间间隔

假设 `enable.auto.commit` 设置为 true，`auto.commit.interval.ms` 设置为 3000，试想一下会不会出现这样的问题：

`poll` 方法返回了 500 条数据，需要 5 秒钟才能处理完，假设在第 4 秒的时候应用挂了，offset 是不是在第 3 秒的时候已经被自动提交了，从而导致第 4 秒之后的数据“丢失”了？

正确答案是：不会的！虽然 `auto.commit.interval.ms` 设置为 3000，但是检查时间间隔是否过了 3 秒是由 `poll` 方法去触发的，所以只要在记录还没处理完之前我们没有主动去调用 `poll` 方法，就算时间间隔到了，也不会去自动提交。

### 自动提交是在哪里执行的

kafka consumer offset 的提交是有 `org.apache.kafka.clients.consumer.internals.ConsumerCoordinator` 来完成的，真正执行提交的有两个方法：

- 同步提交：`org.apache.kafka.clients.consumer.internals.ConsumerCoordinator#maybeAutoCommitOffsetsSync`
- 异步提交：`org.apache.kafka.clients.consumer.internals.ConsumerCoordinator#maybeAutoCommitOffsetsAsync`

#### 同步提交

```
private void maybeAutoCommitOffsetsSync(Timer timer) {
    if (autoCommitEnabled) {
        Map<TopicPartition, OffsetAndMetadata> allConsumedOffsets = subscriptions.allConsumed();
        try {
            log.debug("Sending synchronous auto-commit of offsets {}", allConsumedOffsets);
            if (!commitOffsetsSync(allConsumedOffsets, timer))
                log.debug("Auto-commit of offsets {} timed out before completion", allConsumedOffsets);
        } catch (WakeupException | InterruptException e) {
            log.debug("Auto-commit of offsets {} was interrupted before completion", allConsumedOffsets);
            // rethrow wakeups since they are triggered by the user
            throw e;
        } catch (Exception e) {
            // consistent with async auto-commit failures, we do not propagate the exception
            log.warn("Synchronous auto-commit of offsets {} failed: {}", allConsumedOffsets, e.getMessage());
        }
    }
}
```

调用这个方法，当我们开启了自动提交，就会触发一个同步提交。那么哪里会调用这个方法？

- 加入一个消费者组之前：`org.apache.kafka.clients.consumer.internals.ConsumerCoordinator#onJoinPrepare`
- 关闭一个消费者之前：`org.apache.kafka.clients.consumer.internals.ConsumerCoordinator#close`

这两个触发点都跟我们要讨论的 `auto.commit.interval.ms` 问题无关，所以这里就不展开了。

#### 异步提交

```
public void maybeAutoCommitOffsetsAsync(long now) {
    if (autoCommitEnabled) {
        nextAutoCommitTimer.update(now);
        if (nextAutoCommitTimer.isExpired()) {
            nextAutoCommitTimer.reset(autoCommitIntervalMs);
            doAutoCommitOffsetsAsync();
        }
    }
}
```

当 nextAutoCommitTimer 到期了就会执行 `doAutoCommitOffsetsAsync()` 方法进行异步提交，这个到期时间间隔就是 `auto.commit.interval.ms` 设置的间隔，所以我们只要跟踪 `maybeAutoCommitOffsetsAsync` 方法的调用方就知道什么时候会检查是否已经到期，从而进行自动异步提交。

通过 IDEA 快捷键查看，也有两个地方调用：

- 手动分配分区时：`org.apache.kafka.clients.consumer.KafkaConsumer#assign`
- 拉取数据时：`org.apache.kafka.clients.consumer.KafkaConsumer#poll(java.time.Duration)`

手动分配分区时调用是确保消费者之前分配的老分区 offset 的提交，也和 `auto.commit.interval.ms` 无关。所以，无论同步提交还是异步提交，跟 `auto.commit.interval.ms` 有关的只剩下 `org.apache.kafka.clients.consumer.KafkaConsumer#poll(java.time.Duration)` 方法了，只有这个方法在正常情况下会被多次调用的。

这就验证了文章开头的问题，只要我们没有去调用 `poll` 方法，就算时间间隔到了，也无法触发自动提交。


