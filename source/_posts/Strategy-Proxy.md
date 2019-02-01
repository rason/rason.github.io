title: 'Strategy&Proxy'
categories: "DesignPattern"
date: 2016-01-11 09:57:20
tags: Strategy Proxy
description: Stategy&Proxy DesignPattern 
---

## 前言

在学习Tomcat和Spring源码的过程中，你会发现其中的代码十分优秀，很多地方都值得我们借鉴和学习。而设计模式更是应用得甚广，在这里简单地写一下Spring源码中用到的代理模式和策略模式。

## 代理模式

在[动态代理](http://rason.me/2015/10/22/Why-we-need-dynamic-proxy/)一文中，我们应该能够大概了解什么是代理模式。通俗一点来说，代理模式就是创建一个代理对象，这个代理对象拥有原对象的一个引用，调用代理对象的方法可以在原对象方法的基础上增加一下额外的操作。下图是代理模式的结构：

![代理模式结构](https://raw.githubusercontent.com/rason/rason.github.io/master/image/designpatternproxy.png)

- Subject：抽象主题，即代理对象的真实对象实现的接口。
- ProxySubject：代理类，除了实现抽象主题外，还必须持有所代理对象的引用。
- RealSubject：被代理的类，即目标对象。

<!-- more -->

在[AOP](http://rason.me/2016/01/08/AOP/)一文中也重点提到，AOP就是利用动态代理模式实现的。在Spring中代理对象除了实现目标对象接口外还实现了org.springframework.aop.SpringProxy和org.springframework.aop.framework.Adised接口。Spring中使用的代理模式结构图如下：

![Spring代理模式结构](https://raw.githubusercontent.com/rason/rason.github.io/master/image/designpatternspring-proxy.png)

$Proxy就是创建的代理对象，Subject是抽象主题，代理对象是通过InvocationHandler来持有对目标对象的引用。

## 策略模式

策略模式，就是完成某种事情有多种方法，每种方法都称之为一种策略。使用者可以根据不同的场合使用不同的策略。策略模式结构：

![策略模式](https://raw.githubusercontent.com/rason/rason.github.io/master/image/designpatternstrategy.png)

- Context：使用不同策略的环境。它可以根据自身条件选择不同的策略实现类来完成索要的操作。它持有一个策略实例的引用。创建具体策略对象的方法也可以由它来完成。
- Strategy：抽象策略，定义每个策略都要实现的策略方法。
- ConcreteStrategy：具体策略实现类，实现抽象策略中定义的策略方法。

在Spring中的代理方式有JDK动态代理和CGLIB代理，这两种代理方式使用了策略模式。

在ProxyFactoryBean获取代理对象的时候，会根据不同情况获取到JdkDynamicAopProxy或者ObjenesisCglibAopProxy，而这两个类都实现了AopProxy接口。在这里，ProxyFactoryBean就是不同策略的使用环境，AopProxy就是抽象策略，而JdkDynamicAopProxy和ObjenesisCglibAopProxy就是具体的两种不同策略。

## 总结

- **代理模式**：代理对象在原对象操作基础上增加额外操作。
- **策略模式**：根据情况使用不同的策略来完成某件事情。
