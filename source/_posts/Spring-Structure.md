title: 初识Spring基本结构
categories: Spring
date: 2015-12-28 13:15:59
tags: Spring基本结构
description: Spring基本结构
---

## 前言

前段实际研究了一下Tomcat的源码，对Tomcat的整体架构有了大概的了解。接下来准备研究一下另一个热门框架Spring的源码，这两天大致看了一下Spring的基本结构，现简单的记录分享一下以便加深记忆。

## Spring架构

Spring的组件有十来个，但核心的组件只有几个，下面是Spring框架的总体架构图：

![Spring总体架构](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springspring-structure.png)

由上图可以看出，Spring的核心组件只有三个：Bean，Context和Core组件。这三个组件是Spring框架的基础，没有它们不可能有AOP，WEB等上层特性功能。因此要了解Spring的整体架构，就要首先了解这三个组件存在的意义。

<!-- more -->

## Spring设计理念

Spring解决了什么问题？众所周知，Spring可以让对象的依赖关系转而用配置文件来配置，也就是举世闻名的依赖注入机制。既然谈到了用配置文件来配置对象的依赖关系，那么我们就可以用类比Tomcat源码的方法来学习Spring源码了。

在Tomcat的一系列文章中，我们知道Context组件读取解析web.xml中配置的Servlet，管理着Servlet的生命周期，Context就是Servlet的容器。Spring的设计理念也类似，IOC容器读取解析spring的配置文件，管理着配置文件中bean的注入关系。这种设计策略完全类似Java OOP的设计理念：

- 构建一个数据结构
- 根据数据结构设计它的生存环境
- 数据结构在生存环境中不停地运动，并设计一系列与环境或者其它个体完成信息交换的基础内容

这种设计理念在大多数框架中都类似，因此我们都可以类比地学习。Spring框架中的核心组件正是对应着这种设计理念，数据结构就是Bean，Bean的生存环境就是Context，Core就是完成信息交换的基础内容。所以，Context就是一个Bean关系的集合，这个集合又叫做IOC容器，Core组件就是发现，建立和维护每个Bean之间的关系所需的一系列工具。

它们之间的关系可以用下图表示：

![核心组件关系](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springspring-relation.png)

## Bean组件

Bean组件在org.springframework.beans包下，这个包下的类主要解决了三个问题：Bean的定义，Bean的创建和Bean的解析。对于Spring使用来说我们只关系Bean的创建，其它两个由问题由容器来解决，对我们来说是透明的。

Spring中Bean的创建是典型的工厂模式，它的顶层接口是BeanFactory。BeanFactory的实现类会拥有许多的Bean定义，每一个都会由一个字符串名字来区分。根据Bean的定义，BeanFactory会返回一个独立的 Bean实例或者一个共享的Bean实例（单例模式），具体返回哪种类型会由Bean工厂的配置决定。

BeanFactory的设计目的就是作为应用组件的注册中心和集中管理，Spring的依赖注入功能就是用BeanFactory接口和它的子接口来实现。通常BeanFactory会通过配置源来加载Bean的定义，比如XML文件，然后用org.springframework.beans包下的类来组装成Bean。因此BeanFactory就是一个最基本的IOC容器，前面我们谈到Context是一个Bean关系的集合，也是一个IOC容器，因此可以猜测到Context直接或者间接实现了BeanFactory接口。下图是BeanFactory的继承层次关系：
![BeanFactory继承层次关系](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springspring-beanfactory.png)

BeanFactory有三个子接口：ListableBeanFactory，HierarchicalBeanFactory和AutowireCapableBeanFactory。而它的默认实现类是DefaultListableBeanFactory，它实现了所有的三个接口。为什么会有那么多个接口？因为每个接口都有它的使用场合，主要是区分在spring内部对象的传输和转化过程中，对对象的数据访问所做的限制。例如ListableBeanFactory接口表示这些Bean是可列表的，而HierarchicalBeanFactory表示这些Bean是有继承关系的，也就是每个Bean可能有父Bean。这几个接口共同定义了Bean的集合，Bean之间的关系和Bean的行为。

Bean的定义由BeanDefinition来描述。Bean的定义完整地描述了在Spring的配置文件中你定义的`<bean/>`节点中所有的信息，包括各种子节点。当Spring成功地解析你定义的一个`<bean/>`节点后，在Spring的内部它就被转化成BeanDefinition对象，以后所有的操作都是对这个对象进行的。

Bean的解析过程比较复杂，功能被分得很细。Bean的解析主要就是对Spring配置文件的解析，这个解析过程主要通过以下的类来完成：

![Bean的解析类](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springspring-xmlreader.png)

## Context组件

Context组件在org.springframework.context包下，前面已经大概讲解了一下Context的作用，实际上就是给Spring提供一个运行时的环境，用于保存各个对象的状态。

ApplicationContext是Context的顶级父类，是为应用提供配置的中心接口。在应用运行时，它是只读的，不过可能会被reload，如果它的实现类支持的话。

一个ApplicationContext提供以下功能：

- 访问应用组件的Bean工厂方法，继承自ListableBeanFactory。
- 加载文件资源的能力，继承自ResourceLoader。
- 向注册的监听者发布事件通知，继承自ApplicationEventPublisher。
- 解析messages，支持国际化的能力，继承自MessageSource。
- 继承父Context。

ApplicationContext的相关类结构如下图所示:

![Context相关类结构图](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springspring-context-relation.png)

由上图可知，ApplicationContext的子类主要包含两个方面：

- ConfigurableApplicationContext表示该Context是可修改的，它下面最常用的子类是AbstractRefreshableApplicationContext。
- WebApplicationContext顾名思义就是为了web准备的ApplicationContext，它可以直接访问到ServletContext，通常情况下这个接口用得比较少。

## Core组件

Core组件作为Spring的核心组件，其中包含了许多关键类，一个重要的组成部分就是定义了资源的访问方式。这种把资源都抽象成一个接口的方法值得在以后的设计中借鉴。Core组件主要功能还是涉及资源的定义，加载和解析，这里暂时不打算展开了，到时候研究Spring的启动过程再认真研究一下。

## 总结

以上内容主要是对Spring的设计理念有了初步的了解，同时也大概了解了一下三个核心组件的功能和作用，但实际上并没有做展开和深入的解析。本文内容只能达到了解轮廓的功能，可能很多东西都会似懂非懂，还需继续深入源码学习才能真正了解字里行间的意思。
