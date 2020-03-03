---
title: Redis 哨兵模式（二）
date: 2020-01-10 16:50:38
tags:
---

在上一篇文章中学习了 Sentinel 基本的数据结构和消息通信，现在我们来看一下监控和故障转移。

### 检测主观下线

众所周知，检测服务是否可用的常用方式就是通过发起一个请求，等待服务的响应，如果服务在某个规定的时间点内都没有响应，那么我们就会认为服务挂掉了。

Sentinel 检测主观下线也是采用类似的方式，在默认的情况下，Sentinel 会以每秒一次的频率向所有与它创建了命令连接的实例（包括主服务器，从服务器和其他 Sentinel）发送 PING 命令，并通过响应信息来判断实例是否在线。

- 有效响应：实例返回 +PONG, -LOADING, -MASTERDOWN 三种其中一种
- 无效响应：返回以上三种之外的的其他响应或者是没有响应

Sentinel 配置文件中的 `down-after-milliseconds` 属性用于判断实例进入主观下线所需要的时间：即在 `down-after-milliseconds` 配置的毫秒数内，连续向 Sentinel 返回无效响应，则认为实例进入主观下线状态，此时会更新数据结构中的 flags 属性，打开 `SRI_S_DOWN` 标识。


比如，主服务器在 `down-after-milliseconds` 时间范围内没有返回有效响应，则数据结构会更新如下：

![Sentinel network](/image/sri_s_down.png)

**注意：**

- `down-after-milliseconds` 不仅用于判断主服务器主观下线，还用于从服务器和其他 Sentinel
- 每个 Sentinel 配置的 `down-after-milliseconds` 时间有可能不同，如果配置不同，会出现 Sentinel1 认为已经主观下线，而 Sentinel2 不这样认为

### 检测客观下线

当 Sentinel 将一个主服务器判断为下线之后，为了确认该主服务是否真的下线，它会向同样监视这一主服务器的其他 Sentinel 进行询问，看它们是否也认为主服务已经进入了下线状态（可以是主观下线或者客观下线）。当 Sentinel 从其他 Sentinel 那里接受到足够数量的已下线判断，Sentinel 就会将主服务器判断为客观下线，并对其执行故障转移操作。

#### 发送 Sentinel is-master-down-by-addr 命令

Sentinel 通过使用：

```
Sentinel is-master-down-by-addr <ip> <port> <current_epoch> <runid>
```

命令询问其他 Sentinel 是否判断主服务已经下线，各参数意义如下：

- ip: 被 Sentinel 判断为主观下线的主服务器 IP 地址
- port: 被 Sentinel 判断为主观下线的主服务器 PORT 地址
- current_epoch: Sentinel 当前的配置纪元，用于选举 Leader Sentinel
- run_id: 可以是 * 或者 Sentinel 的 run_id：* 代表仅用于检测主服务器的客观下线状态，而 run_id 则用于选举 Leader Sentinel

#### 接收 Sentinel is-master-down-by-addr 命令

当一个 Sentinel 接收到另一个 Sentinel 的 `is-master-down-by-addr` 命令之后，会根据参数来检测主服务是否已经下线，然后返回一条包含以下三个参数的回复：

- down_status: 1 代表主服务已下线，0 代表未下线
- leader_runid: 可以是 * 或者该响应 Sentinel 判断出的局部 Leader Sentinel run_id
- leader_epoch: 仅在 leader_runid 不为 * 时有效，leader_runid 为 * 时总是 0

#### 接收 Sentinel is-master-down-by-addr 命令的回复

Sentinel 根据回复统计其他 Sentinel 判断主服务已下线的数量，当这一数量达到配置指定的判断客观下线所需的数量时，Sentinel 就会将主服务 sentinelRedisInstance 数据结构中的 flags 属性 SRI_O_DOWN 标识打开，表示主服务已经进入客观下线状态。

![Sentinel network](/image/sri_o_down.png)


**注意：** 不同的 Sentinel 配置判断客观下线的数量可能存在不同

### 选举 Leader Sentinel

当一个主服务器被判断为客观下线时，监视这个下线主服务的各个 Sentinel 会进行协商，选举出一个 Leader Sentinel，并由 Leader Sentinel 对下线的主服务器执行故障转移。

选举规则和方法：

- 所有在线的 Sentinel 都有被选举为 Leader 的资格
- 每次选举之后，不论选举是否成功，所有 Sentinel 的 epoch 都会自增一次
- 在一个配置纪元里面，所有 Sentinel 都有一次将某个 Sentinel 设置为局部 Leader Sentinel 的机会，并且设置之后再这个纪元之内就不能改变
- 每个发现主服务进入客观下线的 Sentinel 都会要求其他 Sentinel 将自己设置为局部 Leader Sentinel
- 当 Sentinel 向其他 Sentinel 发送 `is-master-down-by-addr` 命令中参数设置了自己的 run_id 就是让其他 Sentinel 选举自己为 Leader
- Sentinel 设置局部 Leader 的规则是先到先得，后到的会被拒绝
- Sentinel 会根据 `is-master-down-by-addr` 命令的回复取出 epoch 看是否跟自己的一致，如果一致再取出 leader_runid 看是否跟自己一直，如果一致则表示选举了自己为局部 Leader
- 如果某个 Sentinel 有超过半数的 Sentinel 设置自己为局部 Leader，那么这个 Sentinel 成为真正的 Leader
- 如果在规定的时间内没有选举出一个 Leader ，那么在一段时间之后再次选举，知道选举出一个 Leader 为止

有熟悉的读者可能已经看出，这就是 Raft 选举算法。

### 故障转移

选举出 Leader 之后，Leader 将会对已下线的主服务器执行故障转移操作，该操作包含三个步骤：

1. 挑选一个从服务器作为新的主服务器
2. 让其他从服务器改为复制新的主服务器
3. 将已下线的主服务设置为新的主服务器的从服务器

#### 选出新的主服务器

故障转移的第一步就是挑选出一个状态良好，数据完整的从服务器，然后向其发送 SLAVEOF no one 命令，将这个从服务器转换为主服务器。

Leader 会将已下线主服务器的所有从服务器保存到一个列表里面，然后按照以下规则，一项项地对列表进行过滤：

1. 删除列表中处于下线或者断线状态的从服务器
2. 删除列表中最近五秒没有回复过 Leader Sentinel 的 INFO 命令的从服务器
3. 删除所有与已下线主服务器连接断开超过 down-after-milliseconds * 10 毫秒的从服务器

之后，Leader 将根据从服务器的优先级，对列表中剩余的从服务器进行排序，选出优先级最高的从服务器。如果有相同最高优先级的，则选出偏移量最大的。如果偏移量也相同，则根据 run_id 排序选出最小的从服务器。

在向挑选出的从服务器发送 `SLAVEOF no one` 命令之后，Leader Sentinel 会以每秒一次的频率（平时是每十秒一次）向被升级的从服务发送 INFO 命令，并观察回复中的 role 信息。当被升级服务器的 role 从原来的 slave 变为 master 时，Leader 就知道被选中的从服务器已经升级为主服务器了。

#### 修改从服务器的复制目标

选举出新的主服务器之后，下一步就是向其他从服务器发送 SLAVEOF 命令，让它们复制新的主服务器。

#### 将旧的主服务器变为从服务器

因为旧的服务器已经下线，所以这个设置是保存在旧的主服务器 redisSentinelInstance 数据结构中。当旧的主服务器重新上线时，Sentinel 就会向它发送 SLAVEOF 命令，让它成为新的主服务器的从服务器。

### 小结

- Sentinel 通过每秒一次的 PING 命令来判断服务器是否主观下线
- 对于主观下线的主服务器，Sentinel 通过 `is-master-down-by-addr` 命令询问其他 Sentinel 是否也判断为主观下线
- 当有足够多的 Sentinel 判断主服务为主观下线则会设置为客观下线
- 当主服务器客观下线之后，Sentinel 会通过 Raft 算法选举出一个 Leader Sentinel 进行故障转移
- 故障转移三个步骤：选举新主，其他从服务器复制新主，旧服务器设置为新主的 Slave


