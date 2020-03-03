title: Tomcat架构
categories: Tomcat
date: 2015-11-19 13:31:31
tags: Tomcat架构
description: Tomcat架构 Tomcat Architecture
---

上两篇文章探讨了一下SpringMVC的架构，我们知道`DispatcherServlet`是它的核心入口，并且可以断言**任何MVC框架都是已servlet技术为基础发展起来的**。因此本来想是继续探讨一下servlet技术的工作原理，但是`servlet`技术又与servlet容器关系密切，所以只好先探讨一下servlet容器，以我们最熟悉的tomcat容器为例子。本文为tomcat8。

## server.xml

将项目部署到tomcat，我们必然需要配置`server.xml`。原来我们一直配置的这个文件，就已经把它的整体架构展示给我们了。只是很多时候我们只是谷歌了一下怎么配置，能跑起来就行了，并没有进行更深入的探究。

```
<?xml version='1.0' encoding='utf-8'?>
<Server port="8005" shutdown="SHUTDOWN">
  <Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    <Connector port="8009" protocol="AJP/1.3" redirectPort="8443" />
    <Engine name="Catalina" defaultHost="localhost">
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        <Context path="xxx" docBase="xxx"/>
      </Host>
    </Engine>
  </Service>
</Server>
```

由于篇幅关系上述配置文件有删减，可以看出配置文件主要由Server，Service，Connector，Engine，Host，Context组成，那么他们各自的作用是什么？

<!-- more -->

### Server

在tomcat的世界里，一个Server就代表整个容器。tomcat会提供Server的默认实现，开发者很少自己自定义一个Server。

### Service

Service是Server里面连结一个或者多个Connector到一个确切的Engine的一个中间件。与一样，tomcat会提供Server的默认实现，开发者很少自己自定义一个Service。

### Engine

Engine代表一个具体的Service的请求处理管道。由于一个Service可能有多个Connector，Engine接收并处理来自这些Connector的所有请求，并且处理响应信息返回到恰当的Connector，然后由Connector返回给客户端。

### Host

Host就是一个虚拟主机，一个Engine可能包含多个Host

### Connector
	
Connector是用于处理与客户端的通讯。在tomcat有多个可用的Connector，包括HTTP connector和 AJP connector。HTTP connector通常用于把tomcat当做一个单一的server时，而AJP connector通常用于将tomcat连接到一个web server（例如Apache HTTPD server）。

### Context

一个Context就代表一个 web application。一个Host可能包含多个Context，每一个都有唯一的路径。

## 架构图

根据以上分析，我们可以用一个图来表示上述关系：

![Tomcat架构](/image/tomcat.png)

## Tomcat启动流程

### 1.从命令行运行org.apache.catalina.startup.Bootstrap

- 初始化classloaders
- 通过反射初始化启动类org.apache.catalina.startup.Catalina
- Bootstrap.daemon.init()完成

### 2.处理命令行参数（开始，停止），假设为启动命令

- Catalina.setAwait(true);
- Catalina.load()	初始化目录和命名空间，通过Digester方式根据server.xml装配组件（上面提到的架构内容）
- Catalina.start()	开始一个新的server实例	        
- Tomcat receives a request on an HTTP port   接受http端口的请求
- Invocation of the servlet class   调用servlet处理请求

## Tomcat请求处理流程

- ThreadPoolExecutor里面有独立的线程接收请求。通过ServerSocket.accept()方法不断地等待请求的来临，当请求来临时线程就会唤醒。
- ThreadPoolExecutor会指定一个任务线程来处理请求。
- 调用Coyote Http11Processor的处理方法来处理请求。
- 通过Http11InputBuffer来将HTTP请求包装成原生的请求。这个原生的请求包含一些http信息，比如servername, port, scheme等。
- 请求包装完成之后，Http11Processor会调用CoyoteAdapter的service()方法装配一些信息（request, cookies, the context）。
- 转换完成之后，CoyoteAdapter会调用它的容器StandardEngine的invoke(request,response)方法。
- StandardEngine.invoke()方法会简单地调用容器的pipeline.invoke()。
- 默认情况下engine只有一个Valve（StandardEngineValve），这个Valve会调用Host pipeline的invoke方法；
- StandardHost默认会有两个Valve（StandardHostValve和ErrorReportValve）
- StandardHost的valve会获取Manager和session
- 调用Context pipeline的FormAuthenticator valve。然后StandardContextValve会调用context的监听器，再会调用StandardWrapperValve。
- 在调用StandardWrapperValve过程中，会调用Jasper，然后再会调用实际的servlet。

上述步骤一点儿也不像是人说的话，通俗地说就是**先把http请求信息包装一下，然后将请求通过一个管道传输，在传输过程中会经过很多个阀门，每个阀门都会进行一些处理**。

以上仅对tomcat的主要架构罗列了一下，细节问题需自行源码探讨。
