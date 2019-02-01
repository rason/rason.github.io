title: Tomcat是怎么找到特定的Servlet来处理请求的
categories: Tomcat
date: 2015-12-01 14:22:00
tags: Tomcat find Servlet
description: How do tomcat find the specific servlet
---

## 前言

前面几篇文章讲解了Tomcat的架构，启动流程等内容，Tomcat整体有了个大概的认识。可是，我们还不是很清楚Tomcat接收到的请求是如何到达特定的Servlet，只是在[Tomcat架构](http://rason.me/2015/11/19/Tomcat-Architecture/)这篇文章中略略地提到了Tomcat请求的处理流程。现在，我们再来看一下Tomcat是如何接受处理请求并找到特定的Servlet。

## Tomcat请求处理流程图

![Tomcat请求处理流程图](https://raw.githubusercontent.com/rason/rason.github.io/master/image/tomcatrequest-process.png)

上图是Tomcat8请求处理的完整流程图，如看不清晰请在新窗口打开。

<!-- more -->

整个过程的脉络已经非常清晰，具体的源码细节比较复杂，有兴趣的同学可以自己去看一下。这里只大概地描述一下request和response形态的转化过程：

Tomcat接收到请求首先会创建org.apache.coyote.Request和org.apache.coyote.Response,这两个类是Tomcat内部使用的描述一次请求和响应的信息类，作用就是在服务器接收到请求后，经过简单解析将这个请求快速分配后续线程去处理。接下去当一个用户线程来处理这个请求时又创建org.apache.catalina.connector.Request和org.apache.catalina.connector.Response对象。这两个对象一直贯穿整个Servlet容器直到要传给的Servlet，传给Servlet的是Request和Response的门面类RequestFacade和ResponseFacade。如下图所示：

![Request和Response的转变过程](https://raw.githubusercontent.com/rason/rason.github.io/master/image/tomcatrequest-change.png)

## Tomcat如何找到特定的Servlet

我们知道一个请求URL包含了这些信息：*http://hostname:port/contextpath/servletpath*，那么只要将这些信息封装起来（MappingData），并设置在请求（org.apache.catalina.connector.Request）中。这样的话，按照上面的流程图，request已经包含了各层路由信息，在每一步中就可以找到相应的容器或者Servlet来处理这个请求了。而Tomcat正是基于这样的思路来实现的。

那么，现在的问题是。根据这个请求URL，怎么知道服务器中有没有相应的host,context和warpper？也就是说Tomcat服务器中是用什么东西来保存Container容器中所有子容器信息的呢？

### org.apache.catalina.mapper.Mapper

在Tomcat8中，这些信息是由org.apache.catalina.mapper.Mapper这个类来保存的。Mapper保存了所有的容器信息，你一定很奇怪它究竟是怎么做到的。要搞清楚这一点，就要看一下org.apache.catalina.mapper.MapperListener类。MapperListener持有一个Mapper属性，并且实现了容器监听者和生命周期监听者接口，实现监听者接口的作用稍后再说。我们先来看下MapperListener的startInternal()方法：

```
    @Override
    public void startInternal() throws LifecycleException {

        setState(LifecycleState.STARTING);

        findDefaultHost();

        Engine engine = (Engine) service.getContainer();
        addListeners(engine);

        Container[] conHosts = engine.findChildren();
        for (Container conHost : conHosts) {
            Host host = (Host) conHost;
            if (!LifecycleState.NEW.equals(host.getState())) {
                // Registering the host will register the context and wrappers
                registerHost(host);
            }
        }
    }

    private void addListeners(Container container) {
        container.addContainerListener(this);
        container.addLifecycleListener(this);
        for (Container child : container.findChildren()) {
            addListeners(child);
        }
    }
```

上面的代码有两处重点：

- **addListeners方法会递归调用，MapperListener将作为所有容器监听者**
- **registerHost(host)方法同时会注册context和wrappers**

我们看一下registerHost(host)做了什么工作：

```
    private void registerHost(Host host) {

        String[] aliases = host.findAliases();
        mapper.addHost(host.getName(), aliases, host);

        for (Container container : host.findChildren()) {
            if (container.getState().isAvailable()) {
                registerContext((Context) container);
            }
        }
        if(log.isDebugEnabled()) {
            log.debug(sm.getString("mapperListener.registerHost",
                    host.getName(), domain, service));
        }
    }
```

我们可以看到，这里所做的工作就是将host信息保存在mapper中。同理，registerContext就是把context信息保存在mapper中，registerContext方法里面还会将warpper信息保存到mapper中。也就是说，**MapperListener.startInternal()方法将所有Container容器信息保存到了mapper中**。

那么，现在初始化把所有容器都添加进去了，如果容器变化了将会怎么样？这就是上面所说的监听器的作用，容器变化了，MapperListener作为监听者会被通知，然后就会更新mapper。如下图所示：

![请求路由图](https://raw.githubusercontent.com/rason/rason.github.io/master/image/tomcatrequest-routes.png)

现在，我们知道了请求如何到达正确的Wrapper容器，但是请求到达最终的Servlet还要完成一些步骤，必须要执行Filter链，以及要通知你在web.xml中定义的listener。

## 总结

通过这篇文章，我们需要知道以下几点：

- **Tomcat的请求处理流程**
- **Tomcat请求处理流程中Request和Response的转化**
- **Tomcat是如何根据URL找到相应的Servlet**
- **Tomcat是用什么来保存容器的信息，以便在配置URL时找到相应的容器**
