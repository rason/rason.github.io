---
title: Redis 哨兵模式（一）
date: 2020-01-06 19:17:12
tags:
---

上一篇文章研究了一下 Redis 的主从复制，如果再进一步思考，假设发生了主从切换的情况，客户端是怎么感知到，从而连接到主服务器进行读写操作呢？

今天我们要研究的 Sentinel(哨兵) 模式就是 Redis 高可用的一种方案：由一个或多个 Sentinel 实例组成的 Sentinel 系统，可以监控任意多个主服务器，以及这些主服务器下的所有从服务器。当监控到主服务器下线之后，可以自动将某个从服务器升级为新的主服务器。

由于篇幅原因，今天先来了解一下 Sentinel 的一些数据结构和消息通信的基本知识，有了今天的基础知识之后，下一篇文章再来了解如何检测服务器下线情况和故障转移。

### 什么是 Sentinel

Redis Sentinel 其实就是一个比较特殊的 Redis 服务器，可以通过 `redis-sentinel /path/to/your/sentinel.conf` 或者 `redis-server /path/to/your/sentinel.conf --sentinel` 来启动。

当一个 Sentinel 启动时，它需要执行以下步骤：

1. 初始化服务器
2. 将普通 Redis 服务器使用的代码替换成 Sentinel 代码
3. 初始化 Sentinel 状态
4. 根据给定的配置文件，初始化 Sentinel 的监控主服务器列表
5. 创建与主服务器的网络连接

从上面的步骤我们可以得到的信息：Sentinel 就是一个普通的 Redis 服务器替换了一些达到 Sentinel 功能的代码，根据配置文件初始化并监控主服务器。

既然要监控 Redis 服务器的情况，那么 Sentinel 肯定有数据结构来保存相关的信息，然后才根据相关信息来做决策。所以，我们要先了解 Sentinel 的数据结构。

### Sentinel State

Sentinel 用一个叫做 sentinelState 的数据结构保存了服务器中所有和 Sentinel 功能有关的状态（服务器的一般状态仍然由 redisServer 数据结构来保存，因为 Sentinel 本质也是一个 Redis 服务器）

```
struct sentinelState {
	// 当前纪元，用于实现故障转移
	uint64_t current_epoch;

	// 保存了所以被这个 sentinel 监控的主服务器
	// 字典的 key 是主服务器的名称，在 sentinel.conf 配置文件中指定
	// 字典的值是一个指向 sentinelRedisInstance 结构的指针
	dict *masters;
	int tilt;
	int running_scripts;
	mstime_t tilt_start_time;
	mstime_t previous_time;
	list *scripts_queue;
}
```

本次我们的主题只需要关注有注释的两个属性，其中重点就是 masters 属性，这里记录了 Sentinel 系统所监控的主服务器。这个时候大家可能就会有疑问了，Sentinel 不是还监控主服务下面的所有从服务器吗，为什么 sentinelState 这个数据结构没有 slaves 属性？这是因为 slaves 被保存到 sentinelRedisInstance 结构了。

### sentinelRedisInstance

上面提到 Sentinel 监控的 masters 信息被保存在 sentinelRedisInstance 这个数据结构中。其实不仅是 master , slave 和 Sentinel 系统中的其他 Sentinel 节点信息也是用这个数据结构来保存，只是不同的角色所用到的属性不太一样。现在我们来看下这个数据结构的几个关键属性。

```
struct sentinelRedisInstance {
	// 标识值，记录了实例的类型，以及该实例的当前状态
	int flags;

	// 实例的名字
	// 主服务器的名字由用户在配置文件中指定
	// 从服务器以及 Sentinel 的名字由 Sentinel 自动设置，格式为 IP:PORT
	char *name;

	// 实例运行ID
	char *runid;

	// 配置纪元，用于实现故障转移
	uint64_t config_epoch;

	// 实例的地址，这个数据结构保存的是实例的 IP 和 PORT
	sentinelAddr *addr;

	// 实例多少毫秒没响应被认为主观下线，由 SENTINEL down-after-milliseconds 指定
	mstime_t down_after_period;

	// 多少个实例认为主观下线之后变为客观下线，即 SENTINEL monitor <master-name> <ip> <port> <quorum> 中的 quorum 参数
	int quorum;

	// 在执行故障转移时，可以同时对新的主服务器进行同步的从服务器数量
	int parallel_syncs;

	// 故障转移超时时间
	mstime_t failover_timeout;

	// 这个属性只有 master 才有，保存的是该 master 下属的所有 slave。key 是 ip:port，value 是 sentinelRedisInstance 指针
	dict *slaves;

	// 这个属性只有 master 才有，保存的是监控该 master的所有 Sentinel。key 是 ip:port，value 是 sentinelRedisInstance 指针
	dict *sentinels;
}
```

### 示例

核心的两个数据结构我们已经有所了解，为了让大家更有体感，我们来画一下具有三个 Sentinel 和一主两备的 Redis 实例的结构图。

假设我们 sentinel.conf 配置文件中配置的 master 名字为 mymaster。

![Sentinel deployment](https://raw.githubusercontent.com/rason/rason.github.io/master/image/sentinel-deploy.png)

上面是部署结构图，三个 Sentinel 和一主两备的 Redis 实例，下面是数据结构。

![Sentinel data struct](https://raw.githubusercontent.com/rason/rason.github.io/master/image/sentinel-datastruct.png)

### 获取主服务器信息

看完上面的示例，大家可能会有个疑问，在初始化的时候，我们配置文件只配置了 master 的相关信息，并没有配置 slave 的信息，Sentinel 是怎么获取到 slave 的信息并保存在数据结构中的呢？

回想一下主从复制的部署过程，从服务会通过 SLAVEOF 命令来复制主服务器，建立了双方之间的联系，此时主服务器是有 slave 服务器的相关信息。所以，在 Sentinel 初始化的时候并不需要那么麻烦把 slave 服务器也配置到 sentinel.conf 配置文件中。

Sentinel 初始化的最后一步是创建到主服务器的网络连接，包含命令连接和订阅连接：

- 命令连接：用于向主服务发送命令，并接收命令回复
- 订阅连接：用于订阅主服务器的 `__sentinel__:hello` 频道，这个作用后面会介绍

Slave 服务器的信息就是通过命令连接来发现的，Sentinel 默认会以十秒一次的频率，通过向主服务发送 INFO 命令，主要可以获得以下两方面的信息：

- 主服务器本身的信息，如 run_id 和 role
- 所有从服务器信息，包含 ip 和 port

根据这些返回的信息，就可以更新上面的数据结构中的内容了。

### 获取从服务器信息

从主服务器信息中获取到所有从服务器的 ip 和 port 之后，Sentinel 也会创建到从所有从服务器的命令连接和订阅连接，也就是说 Sentinel 对所有监控的 Redis 服务器（不管主从）都会建立命令连接和订阅连接。

![Sentinel network](https://raw.githubusercontent.com/rason/rason.github.io/master/image/sentinel-link.png)

创建命令连接之后，也是十秒一次的频率向从服务器发送 INFO 命令获取相关信息：

- 从服务器 run_id, role, slave_priority, slave_repl_offset
- 主服务器 master_host, master_port
- 主从服务器连接状态 master_link_status

获取到这些信息之后更新从服务器的 sentinelRedisInstance。

### 向主服务器和从服务器发送消息

现在，我们已经知道 Sentinel 怎么发现 slave 服务器的了，但是现在另外一个疑问来了，Sentinel 是怎么发现其他 Sentinel 的呢？Sentinel 之间是需要互相通信的，这样才能进行故障转移等工作，所以 Sentinel 之间必须能够互相发现。

答案就是通过上面所说的订阅链接，从订阅的 `__sentinel__:hello` 频道接收到信息，这些信息包含了其他 Sentinel 的信息。那么 `__sentinel__:hello` 频道里面的信息是谁发送的呢？必然需要有客户端向 `__sentinel__:hello` 频道发送了消息，Sentinel 才能收到消息。

既然是通过订阅连接发现其他 Sentinel ，只有 Sentinel 本身知道自己的信息，所以转了一圈发送信息到 `__sentinel__:hello` 频道的也是 Sentinel。简单来说就是 Sentinel 通过 PUBLISH 命令向所有被监控的主服务器和从服务器的 `__sentinel__:hello` 频道发送消息，告诉别的 Sentinel 自身的相关信息。

在默认情况下，Sentinel 会以两秒一次的频率，通过命令连接向所有被监控的主服务器和从服务器发送以下格式的命令：

```
PUBLISH `__sentinel__:hello` "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"
```

其中 s_ 开头的参数是 Sentinel 本身的信息，而 m_ 开头的参数是主服务器的信息。

以我们上面的例子来说，当 Sentinel1， Sentinel2， Sentinel3， 都向他们所监控的所有主服务器和从服务器发送了 PUBLISH 命令之后，三个 Sentinel 都会通过订阅连接获取到各自发送的消息，包含自己发送的那一条。当一个 Sentinel 从 `__sentinel__:hello` 频道收到一条消息时，会对这条消息进行分析：

- 如果 Sentinel runid 跟自己的 runid 相同，证明是自己发送的，则不作处理
- 如果不同，证明是其他 Sentinel 发送过来的，然后就更新数据结构

### 创建连向其他 Sentinel 的命令连接

通过 `__sentinel__:hello` 频道发现了其他的 Sentinel ，那么为了和其他 Sentinel 通信，需要建立与其他 Sentinel 的命令连接。最终监控同一个主服务的多个 Sentinel 将形成互相连接的网络：

![Sentinel network](https://raw.githubusercontent.com/rason/rason.github.io/master/image/sentinel-network.png)

通过命令连接相连的各个 Sentinel 可以通过向其他 Sentinel 发送命令请求来进行信息交换，下一篇文章要介绍的检测主观下线、客观下线就是通过信息交换来实现的。

这里要注意的是，Sentinel 在连接主服务器和从服务器时，会同时创建命令连接和订阅连接，但是在连接其他 Sentinel 时，却只会创建命令连接，因为他们已经互相发现了，没必要用订阅连接。

### 小结

今天的文章由于篇幅关系其实还没有讲到核心功能，检测服务器状态和故障转移，但是有了这些基础之后就很简单了，现在做一个小结：

- Sentinel 是一个特殊的 Redis 服务器，通过配置文件获取要监控的 master 信息
- Sentinel 通过向主服务发送 INFO 命令获取到 slave 信息
- Sentinel 通过订阅 `__sentinel__:hello` 频道获取其他 Sentinel 信息
- Sentinel 会向主服务和所属的从服务器建立命令连接和订阅连接，每十秒发送 INFO 命令，每两秒发送 PUBLISH 命令
- Sentinel 之间只会建立命令连接
