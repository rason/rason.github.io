title: 博客一周年
categories: Thought
date: 2016-08-22 10:36:15
tags: Thought
description: 博客一周年
---

## 前言

不知不觉，这个博客已经坚持写了一周年。说起来也是惭愧，毕业三年了才开始写技术博客，虽然大学的时候在博客园写过一段时间，但还是无疾而终。真正让我下定决心来写博客的原因是稀里糊涂写了三年的代码，逐渐觉得自己的基础实在是太差，很多东西都不知其所以然。很庆幸，这次终于算是坚持下来了，可以写一周年总结了。

## 博客之初

这一年的博客，大部分是阅读笔记和总结，中部分的源码学习和技能点学习，小部分的日常技术总结等。时间线基本上是倒着来的，一开始写的是杂七杂八的技术总结，然后再是源码的学习，最后才是读书笔记。这其实也是很多事物的一般发展过程，先是瞎折腾，再是重点学习，最后是系统学习。那么，就按时间线的先后来简单总结一下吧。

### 博客的搭建

为了简单起见，我并没有购买云服务器搭建自己的博客，而是选择托管在`GitHub Pages`上。`GitHub Pages`推荐使用`Jekyll`来写博客，记得是在windows系统上安装使用`Jekyll`比较麻烦，最后选择了`Hexo`。经过一年的使用`GitHub Pages`+`Hexo`的方案还不错，搭建完之后写博客就是几条命令的事情，那感觉叫一个酸爽。

### Git的学习

既然博客托管在`GitHub Pages`上，那么顺其自然地花了一些时间在`Git`的学习上。说起来也是惭愧，在托管博客的时候才发现我的`GitHub`账号在大三（2011）的时候就已经注册了，然而大学和工作的前三年都没有使用`Git`进行版本管理，所以也没有深入地了解和学习`Git`。直到博客的搭建和近几年来`Git`的火热程度，我决定系统性地学习一下`Git`。

- [Git远程仓库](http://rason.me/2015/08/25/Git-remote/)一文总结了远程仓库相关命令的使用，`git remote`、`git fetch`、`git branch`、`git checkout`、`git push`、`git pull`。

- [Git分支](http://rason.me/2015/09/02/Git-branch/)一文总结了Git分支的简单使用，如何在现有分支新建一个新需求特性分支，如何在需求开发过程中回到master中修复一个bug，如何将分支合并到master中。

- [Git内部原理](http://rason.me/2015/09/08/Git-theory/)一文简单地总结了Git的内部原理，Git实际上就是一个键值对数据库，每一次提交都对应一个Git提交对象。

<!-- more -->

## 源码学习

在学习了Git之后，期间还写了一些简单的日常技术总结。然后就进入关键技能点的突破学习，由于工作三年基本上都是做Web相关开发，所以就打算沿着Web开发这条线深入学习。

做Java Web开发的，无非就是利用Servlet做文章，所以深入学习Servlet成了我第一个学习重点。于是，我选择了Tomcat源码的学习，主要是想搞清楚，一个请求是如何通过Tomcat达到Servlet的。

### SpringMVC初了解

在研究Tomcat源码之前，先查了一下资料学习一下SpringMVC是如何处理请求的。

- SpringMVC通过一个中央DispatcherServlet将请求进行分发，泛化了参数和返回值的含义，并且引入Annotation来完成请求和响应的映射关系

- SpringMVC的运行体系是由DispatcherServlet、组件和容器这三者共同构成的。在这个运行体系中，DispatcherServlet是逻辑处理的调度中心，组件则是被调度的操作对象。而容器在这里所起到的作用，是协助DispatcherServlet更好地对组件进行管理。

### Tomcat源码学习

在大概了解SpringMVC工作原理之后正式进入Tomcat源码的学习。

- [Tomcat架构](http://rason.me/2015/11/19/Tomcat-Architecture/)一文简单熟悉了Tomcat的基本架构：Server、Service、Engine、Host、Context和Connector。

- [Tomcat的Container和Lifecycle](http://rason.me/2015/11/23/Tomcat-s-Container-Lifecycle/)一文总结了容器和生命周期的基本知识，Engine、Host、Context、Wrapper都实现了Container接口，Container接口扩展于Lifecycle接口，Tomcat的启动流程就是一系列容器初始化和调用的过程。

- [Tomcat与Servlet之间的接口](http://rason.me/2015/11/25/Interface-between-Servlet-and-Tomcat/)一文中总结了Wrapper相关知识，知道了Wrapper是Servlet与Tomcat之间的接口与纽带，Servlet在Tomcat中就是以Wrapper的形式存在的；并且知道了Context的监听器类ContextConfig完成了Web应用的配置解析和相应初始化工作。

- [Tomcat 和 Servlet 中ServletContext，ServletConfig之间的关系](http://rason.me/2015/11/26/The-ServletContext-and-ServletConfig-relations-between-Tomcat-and-Servlet/)一文中的总结：ServletConfig在Tomcat中的实现类是StandardWrapper，ServletContext在Tomcat中的实现类是ApplicationContext，Servlet能获取到的ServletConfig和ServletContext是StandardWrapper和ApplicationContext的Facade类。

- [Tomcat是怎么找到特定的Servlet来处理请求的](http://rason.me/2015/12/01/How-do-Tomcat-find-the-specific-servlet/)一文中总结了Tomcat中请求处理的流程，Http11Processor接收Socket请求，然后通过Connector将请求交给Tomcat Container处理，最后经过ApplicationFilterChain的一系列Filter达到相应的Servlet。Tomcat通过Mapper和MappingData来实现容器的路由。

### Spring源码学习

在对Tomcat源码有了大概的了解之后，转而学习Spring的源码，因为Spring是当前应用比较广泛而且功能十分强大的框架。

- [初识Spring基本结构](http://rason.me/2015/12/28/Spring-Structure/)一文简单地总结了Spring的三大核心组件：Bean，Context和Core。Context就是一个Bean关系的集合，这个集合又叫做IOC容器，Core组件就是发现，建立和维护每个Bean之间的关系所需的一系列工具。

- [IOC容器如何工作](http://rason.me/2015/12/31/How-do-IOC-work/)一文简单地总结了IOC容器的初始化过程，读取资源、解析资源和完成Bean定义的载入。

- [Bean的创建](http://rason.me/2016/01/05/Bean-Create/)一文简单地总结了Bean的创建过程和IOC的扩展点的概念，其中有一个特殊的FactoryBean，它是生产Bean的Bean；在[AOP](http://rason.me/2016/01/08/AOP/)一文中也谈到了AOP就是利用FactoryBean这个扩展点和动态代理来实现的。

- 总的来说，对于Spring源码的学习其实不够深入，只是大概了解一下IOC的加载过程和AOP的大致实现原理。

## 设计模式的学习

在Tomcat和Spring源码学习的过程中发现不少地方都有设计模式的使用，简单地总结了一下。

- **代理模式和策略模式**：AOP就是使用动态代理模式，所谓代理就是代理对象持有实际对象的应用，并添加了一些额外的操作；策略模式就是完成一件事件有多种方法，每种方法就是一种策略，比如Spring实现动态代理就有JDK动态代理和CGLIB动态代理两种方式。

- **门面模式（Facade）和观察者模式（Observer）**：Facade模式通俗地讲就是开放部分数据，将不必要的数据隐藏起来。在Tomcat中，有很多地方都运用了Facade设计模式。比如Request和Response的封装，StandWrapper到ServletConfig的封装，ApplicationContext到ServletContext的封装等。观察者模式也就发布-订阅模式，也就是事件监听机制，通常在发生某个事情前后会触发一些操作；比如Tomcat中的容器类都实现了Lifecycle接口。

- **命令模式和责任链模式**：命令模式的主要作用就是封装命令，把发出命令的责任和执行命令的责任分开，达到解耦的效果。请求者与接收者之间没有直接引用关系，发送请求的对象只需要知道如何发送请求，而不必知道如何完成请求。在Tomcat中，Connector和Container之间的调用就是典型的命令模式，生活中的电视遥控器也是一种命令模式。责任链模式就是很多对象由每个对象对其下家的引用而连接起来形成一条链，请求在这条链上传递，直到链上的某个对象处理此请求，或者每个对象都可以处理此请求，并传递给下家，直到链上的每个对象都处理完。Tomcat中的容器就是责任链模式。

- **适配器和装饰器模式**：适配器就是把一个类的接口边换成客户端所能接收的另一种接口，从而使两个接口不匹配而无法一起工作的两个类能购在一起工作。。InputStreamReader和OutputStreamWriter的作用就是将InputStream和OutputStream适配到Reader和Writer。装饰器模式，顾名思义，就是将某个类重新装扮一下，使得它比原来更“漂亮”，或者功能上更强大，这就是装饰器模式所要达到的目的。在Java I/O类库中有很多不同的功能组合情况，这些不同的功能组合都是使用装饰器模式实现的。

## 书籍阅读

博文之中比重最大的当属阅读笔记了，由于比重过大就不像源码学习那样列举总结了。其实前面也不应该用列举的方式来写一周年总结，只是想着借此机会去回顾之前的博文，然后就写成这样了，果然写作水平渣渣的。

这一年，主要阅读了四本书籍：

- 《深入分析Java Web技术内幕》
- 《大型网站技术架构》
- 《图解TCP/IP》
- 《算法》

那就分别回顾一下这四本书吧。

### 《深入分析Java Web技术内幕》

这本书是淘宝的许令波写的，我只想说，做Java Web开发的人必须要去读一下这本书。它的特点就是全面，对Java Web开发会有一个总体的认识。对于工作了一两年的人来说尤其适合，它让对你所做的工作有更深入更全面的认知。

这本书第一章，首先从Web请求开始，讲述从用户输入网站按下回车那一刻起，到请求达到服务器这一过程中涉及的知识。包括浏览器、HTTP协议解析、DNS域名解析和CDN工作机制等知识。这令我们对Web请求过程有了整体的把握和认识。然后第二章讲述了Java中很重要的知识点I/O问题，进而在第三章中讲述了中文编码问题，因为有I/O的地方就会有编码问题。

四、五、六、七章讲述了编译原理、class文件结构、类加载器和JVM方面的知识。这几章书我认真阅读的只有类加载器那张，这对于我们解决ClassNotFoundException这类异常特别有用。其它几章书并没有认真去阅读，因为有点深奥，为了节省时间就知识大致地浏览了一下。

第八、九章分别讲述了Servlet的工作原理和Session、Cookie方面的知识。毫无疑问，Servlet是Java Web开发最基本的知识，因此我们需要重点去学习。Session和Cookie也是必须要懂的，书中还简单地给出了分布式Session的解决方案。

之后的几章书就是框架的深入学习，有Tomcat、Jetty、Spring、SpringMVC、Ibatis等常用框架的介绍学习。前面对源码的研究就是以这本书为出发点去学习的，建议大家结合书籍和源码一起学习，这样的学习效果更佳。

总的来说，这是一本覆盖比较全面的书，对Java Web开发会有更深入的认识。

### 《大型网站技术架构》

这本书的特点就是泛泛地讲述了大型网站架构的演变过程，给出了大型网站架构的建设和思考方向。如果自己是零架构知识，这本书将会是一本很好的入门书籍。

第一章概述了大型网站架构的演变、模式和要素；第二章分别从几个要素出发讨论讲述了网站的高性能、高可用、伸缩性、可扩展和安全架构；第三章给出了一些案例设计；第四章讲述了架构师的自我修养。

这本书并不需要花费太多的时间，但对架构的认识还是有很大的作用。

### 《图解TCP/IP》

阅读这本书的出发点是自身网络方面的知识比较薄弱，事实上阅读完之后也不见得有多大的提升。只是说温习了一下TCP/IP七层协议和各层所涉及的一些知识，对基础的巩固还是有点作用的。

这本书涉及的基础知识包括：网络基础、TCP/IP基础、数据链路、IP协议、TCP和UDP、路由协议、应用协议和网络安全。基本上都是概念的认识和介绍，感觉就是要考自己的记忆力了。

怎么说呢，网络方面的知识虽然不是开发中的重点，但还是需要大致地掌握，基础知识还是要去温习一下的。

### 《算法》

这本书是我耗时最长的一本书了，前前后后看了两个多月，结果也只是将前三章看完了而已。但这确实是需要细读的一本书，强烈推荐。

第一章讲述了一些基础编程模型、数据抽象和基础的数据结构包、队列和栈，算是本书的热身章节。

第二章就是排序，分别讲解了两种初级排序（选择和插入，希尔）、归并排序、快速排序和优先队列中的堆排序。阅读完这一章书，就能熟悉这几种最常用的排序算法了。

第三章讲述的是查找，重点讲述符号表的几种实现方式：基础实现方式（无序链表和有序数组）、二叉查找树、平衡查找树（2-3树和红黑树）和散列表（拉链法和线性探测）。学完本章不但会了解到符号表的几种实现方法，还知道它们性能之间的差别，并且进一步熟悉链表、数组和树这几种数据结构。

剩余的几章书还没有时间去学习。

## 总结

洋洋洒洒地写了一堆文字，然而总结的思路似乎不太清晰。总的来说，坚持写博客这一年，之前遇到的很多问题能够知其所以然了，更重要的一点是现在遇到问题会去刨根问底了。接下来，再接再厉吧！
