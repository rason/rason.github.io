---
title: Curator Service Discovery
date: 2017-10-25 16:23:18
categories: Distributed
tags: Curator
description: Curator Service Discovery
---

## 前言

[Zookeeper](https://zookeeper.apache.org/)可以用于服务的注册与发现，而[Curator](https://curator.apache.org/index.html)则是用于操作Zookeeper的JAVA客户端。使用Curator来操作Zookeeper只需要导入`curator-x-discovery`依赖即可。

```
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-x-discovery</artifactId>
    <version>x.x.x</version>
</dependency>
```

## 什么是发现服务？

在SOA/分布式系统中，服务间需要发现对方。比如说一个web服务需要发现一个缓存服务等。DNS可以用来发现服务，但它没有足够的灵活性，来应对不断变化的服务。一个服务发现系统提供这样的机制：

- 服务注册
- 定位特定服务的一个实例
- 服务实例变化时进行通知

## Curator 服务发现

### ServiceInstance

在Curator中，一个服务的实例用ServiceInstance 类来表示。ServiceInstance 包含名字，id，地址，端口或者ssl端口，和一个可选的payload(用户自定义)。ServiceInstance 被序列化然后存储到Zookeeper 中，如下所示：

```
base path
       |_______ service A name
                    |__________ instance 1 id --> (serialized ServiceInstance)
                    |__________ instance 2 id --> (serialized ServiceInstance)
                    |__________ ...
       |_______ service B name
                    |__________ instance 1 id --> (serialized ServiceInstance)
                    |__________ instance 2 id --> (serialized ServiceInstance)
                    |__________ ...
       |_______ ...
```

<!-- more -->

### ServiceProvider

服务提供者类，可以按照指定的策略来获取某个特定名字的服务实例。策略即以什么样的方式来从一系列实例中选出一个。一般有三种策略：轮询，随机和固定（一直选择同一个）。

ServiceProvider 通常由ServiceProviderBuilder来创建。你可以通过ServiceDiscovery（下面会提到） 类来获得一个ServiceProviderBuilder。ServiceProviderBuilder允许你设置服务的名字和一些其他可选值。

ServiceProvider 必须通过调用`start()` 方法来启动。完成的时候需要调用`close()`方法。

### ServiceProvider

为了创建ServiceProvider，你需要一个ServiceProvider。 它可以通过ServiceDiscoveryBuilder 来创建。

你需要调用其`start()` 方法来启动，当不再需要使用时调用其`close()` 方法。

### 实例稳定性

如果一个特定的实例发生I/O错误等。你应该调用`ServiceProvider.noteError()`方法并将该实例作为参数传递进去。这样ServiceProvider 根据`DownInstancePolicy` 可能会将该实例标记为"down"。

## 低层次的API

### 注册/注销服务

通常，当我们将服务实例传到ServiceDiscovery 构造器中就会自动注册/注销。如果你需要手动做这件事情，你可以通过以下方法：

```
public void registerService(ServiceInstance<T> service)
                    throws Exception

public void unregisterService(ServiceInstance<T> service)
                      throws Exception                   
```

### 查询服务

你可以查询所有服务的名字，也可以查询特定名字的所有实例或者单个实例。

```
public Collection<String> queryForNames()
                              throws Exception

public Collection<ServiceInstance<T>> queryForInstances(String name)
                                            throws Exception                          

public ServiceInstance<T> queryForInstance(String name,
                                         String id)
                                 throws Exception                                                
```

### 服务缓存

上面的查询方法都是直接调用Zookeeper来进行查询。如果你查询得比较频繁，可以使用`ServiceCache`。它会将服务的实例列表缓存在内存中。可以使用观察者来保持服务实例列表的更新。

你可以通过`ServiceDiscovery.serviceCacheBuilder()` 来获得一个`ServiceCache`。 `ServiceCache` 对象必须通过调用`start()` 方法来启动，然后结束的时候需要调用`close()` 方法。你可以通过调用下面的方法来获取当前可以的服务实例列表：

```
public Collection<ServiceInstance<T>> getInstances()
```

`ServiceCache` 支持监听者模式当观察者更新了实例列表之后来获取通知：

```
/**
 * Listener for changes to a service cache
 */
public interface ServiceCacheListener extends ConnectionStateListener
{
    /**
     * Called when the cache has changed (instances added/deleted, etc.)
     */
    public void cacheChanged();
}
```
