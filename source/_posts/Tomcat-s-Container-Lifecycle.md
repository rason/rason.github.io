title: "Tomcat的Container和Lifecycle"
categories: Tomcat
date: 2015-11-23 15:49:27
tags: Container Lifecycle
description: Tomcat Container Lifecycle
---

上篇文章[Tomcat架构](http://rason.me/2015/11/19/Tomcat-Architecture/)中梳理了一下Tomcat的基本架构，现在来介绍一下Tomcat里面的两个接口`Container`和`Lifecycle`，为深入了解Tomcat的工作原理打下基础。

## Container（容器）

**一个Container对象可以接收来自客户端的请求，并基于这些请求返回响应信息。Container可以通过实现Pipeline接口支持管道阀门，按运行时配置的顺序处理请求。**

在Tomcat中，Container（容器）会以以下的概念存在：

- **Engine**
代表一个完整的servlet引擎，可以包含一个或者多个子容器（Host,Context或者自定义的组）

- **Host**
代表一个包含一系列Context的虚拟主机

- **Context**
代表一个单一的ServletContext，可以包含一个或者多个Wrapper

- **Wrapper**
代表一个servlet定义，servlet在tomcat中是以Wrapper形式出现

<!-- more -->

以上的内容一定很熟悉，因为[Tomcat架构](http://rason.me/2015/11/19/Tomcat-Architecture/)文章就介绍过，原来他们都实现了同一个接口，我们来看看Container接口里面都有些什么方法：

![Container接口方法](https://raw.githubusercontent.com/rason/rason.github.io/master/image/container.png)

方法比较多，但从名字上我们大概就知道用途，不多做解析。我们可以看到有一个`addChild(Container child)`添加子容器的方法，从[Tomcat架构](http://rason.me/2015/11/19/Tomcat-Architecture/)文章中的构架图可以看出，Tomcat中的Container是一层接一层的。那么，Tomcat是怎么将这些容器组件有效地管理起来呢？

## Lifecycle（生命周期）

Container接口扩展于Lifecycle接口，因此可以知道Tomcat正是通过Lifecycle来管理上面那些容器组件的。我们知道Servlet是有init，service，destroy这些生命周期的，Tomcat中的容器（Container）生命周期和Servlet的生命周期类似。

先看一下Lifecycle的结构和方法：

![Lifecycle数据结构和方法](https://raw.githubusercontent.com/rason/rason.github.io/master/image/lifecycle.png)

可以看出Lifecycle主要有**init，start，stop，destroy**四个过程。而对应的生命周期状态却有以下几种：

```
public enum LifecycleState {
    NEW(false, null),
    INITIALIZING(false, Lifecycle.BEFORE_INIT_EVENT),
    INITIALIZED(false, Lifecycle.AFTER_INIT_EVENT),
    STARTING_PREP(false, Lifecycle.BEFORE_START_EVENT),
    STARTING(true, Lifecycle.START_EVENT),
    STARTED(true, Lifecycle.AFTER_START_EVENT),
    STOPPING_PREP(true, Lifecycle.BEFORE_STOP_EVENT),
    STOPPING(false, Lifecycle.STOP_EVENT),
    STOPPED(false, Lifecycle.AFTER_STOP_EVENT),
    DESTROYING(false, Lifecycle.BEFORE_DESTROY_EVENT),
    DESTROYED(false, Lifecycle.AFTER_DESTROY_EVENT),
    FAILED(false, null),
    MUST_STOP(true, null),
    MUST_DESTROY(false, null);
}
```

- **当容器（Container）组件的状态为STARTING_PREP, STARTING 或者STARTED时，调用生命周期的start()方法是没有作用的。**
- **当容器（Container）组件的状态为NEW时，调用生命周期的start()方法时会首先调用init()方法。**

另外我们还注意到Lifecycle接口中还有`addLifecycleListener(LifecycleListener listener)`方法。很显然这是一个**监听器**模式，监听事件的常量也可以在接口中看到，当生命周期变化时可以调用`fireLifecycleEvent`方法通知相应的监听器。

## Tomcat的启动流程

**Tomcat的启动是基于观察者模式设计的，所有的容器会继承Lifecycle接口，它管理着容器的整个生命周期，所有容器的修改和状态改变都会由它去通知已注册的观察者。**Tomcat的启动时序图如图：

![Tomcat启动时序图](https://raw.githubusercontent.com/rason/rason.github.io/master/image/tomcat_start.png)

有了Container和Lifecycle的基础之后，理解上图应该不是很困难，容器之间是层层关联起来的。

## 总结

- **1.我们需要知道Engine，Host，Context，Wrapper都实现了Container接口。**
- **2.Container接口扩展了Lifecycle接口，用于管理容器的生命周期。**
- **3.Tomcat的启动流程就是一系列容器初始化和调用的过程。**
