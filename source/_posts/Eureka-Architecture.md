---
title: Eureka Architecture
date: 2019-07-04 13:52:41
tags:
---

被誉为下一代微服务架构的 Service Mesh 已经越来越火了, 我还在写 Eureka 的文章, 是不是太 low 了. 那就先 low 一回, 下次再时尚一把.

#### 总体架构

![Eureka Architecture](/image/eureka_architecture.png)

上图是 Eureka 官方给出的架构图, 一共有三种角色:

- Eureka Server: Eureka 服务端, 核心就是保存服务的注册表信息, 如果是集群部署, 多个服务端之间会复制注册表信息, 达到最终的一致性.
- Application Service: 即 Service Provider , 作为 Eureka 的客户端, 启动时会注册到 Eureka 服务端, 对外提供服务.
- Application Client: 即 Server Consumer , 也是 Eureka 客户端, 启动时会用 Eureka 服务端获取所有的服务列表.

#### Eureka Server

对于 Eureka 服务端, 我们首先能想到的几个问题:

1. 服务提供者的注册过程是怎么样的, 用什么样的数据结构来保存服务提供者信息?
2. Eureka 服务端怎么知道服务提供者挂掉了, 挂掉了之后会怎么办?
3. 集群情况下, Eureka 服务端之间的注册表信息是怎么同步的?
4. 集群情况下, 一个 Eureka 服务端启动的时候, 是怎么获取到其他节点的注册表信息的?

##### 服务注册(Register)

Eureka 服务端是通过暴露 HTTP 接口来提供服务注册的; 另外, 更新服务状态也是这个接口, 比如服务下线, 会调用这个接口将状态更新为 DOWN.

```
com.netflix.eureka.resources.ApplicationResource#addInstance
```

注册的过程比较简单, 先是注册到本节点, 然后调用 replicateToPeers 向其它 Eureka Server 节点做状态同步（异步操作）

![Service Register](/image/service-register.png)

注册的服务信息保存在一个嵌套的 HashMap 中:

 ```
 ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
 ```

- 第一层 HashMap 的 key 是 appName，也就是应用名字
- 第二层 HashMap 的 key 是 instanceId ，也就是实例ID

为什么是这样的结构, 大家想一下应该就知道了, 因为一个应用会存在多个实例.

##### 服务续约(Renew)

Eureka 服务端会提供服务续约接口给服务提供者调用, 让它告诉 Eureka 服务端它还活着, 正常提供者服务呢, 不要把我给踢掉.

```
com.netflix.eureka.resources.InstanceResource#renewLease
```

接口的实现跟 Register 基本保持一致, 先更新自身节点状态, 然后同步到其他节点.

![Service Renew](/image/service-renew.png)

##### 服务下线(Cancel)

Eureka 服务端会提供服务下线接口给服务提供者调用, 让它告诉 Eureka 服务端它要下线了, 不对外提供服务了, 把我踢掉吧.

```
com.netflix.eureka.resources.InstanceResource#cancelLease
```

![Service Cancel](/image/service-cancel.png)

##### 服务获取

Eureka 服务端会提供接口让服务消费者调用, 用来获取注册在 Eureka 服务端上的服务.

为了提高性能，服务列表在Eureka Server会缓存一份，同时每30秒更新一次.

![Service Fetch](/image/fetch-service.png)

##### 服务剔除

Eureka 服务端会定时检查在一定时间内(默认是90秒)没有续约的服务, 默认是每60秒检查一次. 也就是如果有服务超过90秒没有向 Eureka 服务端发起续约请求的话，就会被当做失效服务剔除掉(如果没有开启自我保护模式的话). 自我保护模式这里不详细展开, 有兴趣的可以谷歌查一下.

失效时间可以通过 `eureka.instance.leaseExpirationDurationInSeconds` 进行配置，定期扫描时间可以通过 `eureka.server.evictionIntervalTimerInMs` 进行配置.

![Service Eviction](/image/service-eviction.png)

##### 节点间服务同步

上面的 Register、Renew、Cancel 接口都会先在本节点进行操作, 然后同步到其他节点. 同步的方式也很简单, 就是将服务提供者请求的方式一样转发到其他节点.

节点之间的状态是采用异步的方式同步的，所以不保证节点间的状态一定是一致的，不过基本能保证最终状态是一致的.

结合服务发现的场景，实际上也并不需要节点间的状态强一致。在一段时间内（比如30秒），节点A比节点B多一个服务实例或少一个服务实例，在业务上也是完全可以接受的（Service Consumer侧一般也会实现错误重试和负载均衡机制）。

##### 节点互相感知

Eureka 服务端之间需要同步注册表的前提是知道对方的存在, 那么他们之间是怎么互相感知的?

Eureka Server在启动后会调用 `EurekaClientConfig.getEurekaServerServiceUrls` 来获取所有的节点，并且会定期更新。定期更新频率可以通过 `eureka.server.peerEurekaNodesUpdateIntervalMs` 配置。

默认情况下是从 `eureka.client.serviceUrl` 配置项读取其他节点的地址, 如果希望灵活控制节点, 可以重写 `getEurekaServerServiceUrls` 方法从外部存储读取服务端地址列表.

![Peer Discovery](/image/peer-discovery.png)

##### 节点初始化

一个新的 Eureka 服务端节点启动时, 会从其他节点获取到所有的服务信息, 前提是 `register-with-eureka = true`, 当然这个配置项默认就是 true.

![Server Init](/image/server-init.png)

#### Service Provider

上面说的是 Eureka 服务端的工作机制, 接下来就是 Eureka 客户端, 首先看下服务提供者.

对于服务的提供者, 主要就是 Register、Renew、Cancel 三个操作.

##### 服务注册(Register)

将服务注册到 Eureka Server 上, 首先需要配置 `eureka.client.registerWithEureka=true`. 注册过程很简单, 就是在服务启动的时候会有一个定时任务调用 Eureka Server 提供的注册接口.

![Client Register](/image/client-register.png)

##### 服务续约(Renew)

服务提供者需要不断地告诉 Eureka Server 自己还活着, 现在这是一个定时任务不断地发送续约请求.有两个参数需要配置:
 
- `instance.leaseRenewalIntervalInSeconds` 续约的频率, 就是多久发送一次续约请求, 默认 30 秒.
- `instance.leaseExpirationDurationInSeconds` 服务时效时间, 就是多少秒内没有发送需求请求, Eureka Server 就认为服务提供者挂了.

![Client Renew](/image/client-renew.png)

##### 服务下线(Cancel)

在服务提供者 shut down 的时候，需要及时通知 Eureka Server 把自己剔除，从而避免客户端调用已经下线的服务。

![Client Cancel](/image/client-cancel.png)

##### 怎么发现 Eureka Server

服务提供者怎么知道 Eureka Server 的地址呢? 这个跟 Eureka Server 各个节点之间的发现是类似的, 默认也是从配置文件中读取, 灵活的话就是重写 `getEurekaServerServiceUrls` 方法.定期更新频率可以通过 `eureka.client.eurekaServiceUrlPollIntervalSeconds` 配置.

![Client Discovery Server](/image/client-discovery-server.png)

#### Server Consumer

服务消费者比较简单, 主要是怎么获取服务列表和更新服务列表.

##### 服务获取

Service Consumer 在启动时会从 Eureka Server 获取所有服务列表，并在本地 **缓存** . 需要注意的是，需要确保配置 `eureka.client.shouldFetchRegistry=true`.

![Client Fetch Service](/image/client-fetch-service.png)

##### 服务更新

由于在本地有一份缓存，所以需要定期更新，定期更新频率可以通过 `eureka.client.registryFetchIntervalSeconds` 配置.

![Client Update Service](/image/client-update-service.png)

##### 怎么发现 Eureka Server

服务消费者和服务提供者本质上都是 Eureka 的客户端, 所以逻辑是一样的.


#### 总结

整个流程, 有很多定时任务, 这里梳理一下:

- Eureka Server: 定时检查更新服务列表缓存, 默认 30 秒; 定时剔除失效服务, 默认 60 秒; 定时更新其他服务端节点地址, 默认 10 分钟.
- Service Provider: 定时续约, 默认 30 秒; 定时更新 Eureka Server 地址, 默认 5 分钟.
- Server Consumer: 定时更新服务列表缓存, 默认 30 秒; 定时更新 Eureka Server 地址, 默认 5 分钟.





