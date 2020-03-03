title: ClassLoader工作机制（二）
categories: Java
date: 2016-02-29 15:07:16
tags: ClassLoader
description: ClassLoader工作机制
---

## 前言

在[ClassLoader工作机制（一）](http://rason.me/2016/02/25/ClassLoader/)中梳理了一下JVM中ClassLoader的工作机制，那么在一些开源框架中是怎么根据这些机制定义自己的ClassLoader呢？今天我们来看一些Tomcat中的ClassLoader。

## 概述

在JVM中，我们知道类加载器遵循委托机制。通常，一个类加载器被用来加载一个class或者资源，它会先请求父加载器来进行加载，如果没有的话才会在自己的仓库中查找。注意：这个模式在Tomcat中的WebappClassLoader中有一点不同，但是主要原则还是相同的。

当Tomcat启动，会创建一系列class loader并会组织成以下的父子关系：

![Tomcat中加载器父子关系](/image/web-Tomcat-ClassLoader.png)

<!-- more -->

## 类加载器定义

### Bootstrap

这里的Bootstrap只是个统称，所指的就是[ClassLoader工作机制（一）](http://rason.me/2016/02/25/ClassLoader/)中谈到的**Bootstrap ClassLoader**和**ExtClassLoader**。加载JVM自身工作需要的class和系统扩展路径（$JAVA_HOME/jre/lib/ext）中class。

### System

系统（也称为）类加载器，也就是上一篇文章提到的**AppClassLoader**。一般来说，它负责加载来自命令`java`中的`-classpath`，或者`java.class.path`系统属性或者`CLASSPAHT`操作系统属性所指定的jar类包和类路径。通过这个类加载器加载的所有类，都对Tomcat自身的类，以及所有web应用的类可见。但是，标准的Tomcat启动脚本（$CATALINA_HOME/bin/catalina.bat），完全无视默认的CLASSPATH环境变量，而是加载了以下三个jar 。

- **$CATALINA_HOME/bin/bootstrap.jar **——包含初始化Tomcat server的main方法和一些依赖的class。
- **$CATALINA_BASE/bin/tomcat-juli.jar 或者 $CATALINA_HOME/bin/tomcat-juli.jar** ——包含一些日志的class，增强java.util.logging API。
- **$CATALINA_HOME/bin/commons-daemon.jar**—— Apache Commons Daemon工程的一些class。

### Common

这个类加载器包含一些附加的class，这些class对Tomcat的内部class和所有web应用都是可见的。

通常，应用中的class不应该展现在这里。这个类加载器的搜索路径在`$CATALINA_BASE/conf/catalina.properties`文件中的`common.loader`中定义。默认设置的搜索路径顺序为：

- **$CATALINA_BASE/lib中未打包的classes和resources**
- **$CATALINA_BASE/lib中的jar文件**
- **$CATALINA_HOME/lib中未打包的classes和resources**
- **$CATALINA_HOME/lib中的jar文件**

默认地，会包含以下的文件：

- annotations-api.jar
- catalina.jar 
- catalina-ant.jar 
- catalina-ha.jar
- catalina-storeconfig.jar 
- catalina-tribes.jar
- ecj-*.jar
- el-api.jar 
- jasper.jar 
- jasper-el.jar 
- jsp-api.jar 
- servlet-api.jar
- tomcat-api.jar 
- tomcat-coyote.jar 
- tomcat-dbcp.jar
- tomcat-i18n-**.jar 
- tomcat-jdbc.jar
- tomcat-util.jar
- tomcat-websocket.jar 
- websocket-api.jar

这里大家可能会有个疑问，CATALINA_HOME和CATALINA_BASE究竟是什么关系？说白了，CATALINA_HOME就是Tomcat的安装目录，内容是每个Tomcat实例共享的；而CATALINA_BASE就是Tomcat一个实例的目录，当只有一个Tomcat实例时，这另个路径指向的是同一个地方。

### WebappX

部署在一个Tomcat实例中的每个应用都会创建一个WebappX类加载器，加载所有WEB-INF/classes里的类，以及WEB-INF/lib里的jar 。

从web应用的角度来看，class和resources的加载会按顺序查找以下仓库：

- Bootstrap classes
- /WEB-INF/classes 
- /WEB-INF/lib/*.jar 
- System class loader classes 
- Common class loader classes 

如果WebappX类加载器被配置为`<Loader delegate="true"/>`，顺序会变成这样：

- Bootstrap classes
- System class loader classes 
- Common class loader classes 
- /WEB-INF/classes 
- /WEB-INF/lib/*.jar 

## 自定义类加载器的作用

从上面的分析可以看出，Tomcat中自定义了System加载器和Common类加载器的搜索路径，另外还自定义了WebappX类加载器。这样做的目的是什么？

1. **隔离各个webapp中的class和lib。**每个webapp都有独立的WebappX类加载器。
2. **安全性问题。**指定Tomcat自身类库的搜索路径，避免恶意或无意的破坏。
3. **热部署问题。**修改文件可以不用重启就能自动装载类库。

## Tomcat启动时类加载器工作流程

所有的应用都有一个或多个入口，查看Tomcat启动脚本可知，Tomcat的启动类是`org.apache.catalina.startup.Bootstrap`。当在启动脚本中运行这个`Bootstrap`启动类时，会经历以下过程：

1. 当执行java命令时，JVM会先使用bootstrap classloader载入并初始化一个Launcher。Launcher会根据系统和命令设定初始化好class loader结构，JVM就用它来获得extension classloader和system classloader。system classloader从上文提到的路径中找到`org.apache.catalina.startup.Bootstrap`执行其Main方法。

2. `org.apache.catalina.startup.Bootstrap`的Main方法中会调用其它方法创建Common classloader，用来加载"org.apache.catalina.startup.Catalina"等相关class和Jar文件，然后启动Tomcat server。

3. Tomcat server在启动StandardContext时会创建WebappX类加载器，用于加载应用中的class和Jar文件。

以上只是一个大致的流程，具体的流程需自行查看Tomcat源码。

## 总结

两篇关于ClassLoader的文章，第一篇是ClassLoader的基本知识，第二篇温习并分析了以下Tomcat中ClassLoader的应用。我们重点需要掌握：

- **ClassLoader的作用，指定搜索路径加载class文件，因此在初始化一个ClassLoader实例时一般都需要指定URL**
- **ClassLoader的分类，即有哪几种类加载器，Bootstrap，Ext，System及应用中自定义的类加载器**
- **ClassLoader的委托加载机制，先是父加载器加载，如果没有父加载器就使用Bootstrap加载器加载，然后才是自身加载器来加载**
