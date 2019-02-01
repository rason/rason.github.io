title: Tomcat 和 Servlet 中ServletContext，ServletConfig之间的关系
categories: Tomcat
date: 2015-11-26 14:36:23
tags: ServletContext ServletConfig
description: Tomcat and Servlet ServletContext ServletConfig
---

## 前言

刚从事一两年Java Web开发的同学们应该很多人都不是很理解ServletContext和ServletConfig究竟是干什么用的，更别说在Tomcat中他们是如何工作的。其实我也不是很理解哈哈，战斗力不足五的渣渣，只好一起来看看源码学习一下咯。

## javax.servlet.ServletConfig

### 作用

**被用作在Servlet初始化时，Servlet容器传递信息给Servlet。**Servlet容器大家应该懂了吧，不懂的话就要看下我的上篇文章[Tomcat与Servlet之间的接口](http://rason.me/2015/11/25/Interface-between-Servlet-and-Tomcat/)了。在Tomcat中，**Context容器才是真正运行Servlet的Servlet容器**。

那么它究竟传递了什么信息给Servlet？

### 方法

```
public interface ServletConfig {

    /** 
     * 返回Servlet实例的名字
     */
    public String getServletName();

    /**
     * 返回ServletContext,被用于调用者与它的Servlet容器之间的交互
     */
    public ServletContext getServletContext();

    /**
   	 * 返回初始化参数
     */
    public String getInitParameter(String name);

    /**
     * 返回初始化参数名字枚举
     */
    public Enumeration<String> getInitParameterNames();
}
```

<!-- more -->

嗯，能传递的信息一目了然了。那么这些信息在Tomcat中是什么时候被设置到ServletConfig中的呢？让我们用逆向思维来找一下，Servlet.getServletConfig()方法能获取到ServletConfig。那么ServletConfig肯定是在Servlet初始化的时候设置到Servlet的，沿着这条思路，就不难想到上篇文章[Tomcat与Servlet之间的接口](http://rason.me/2015/11/25/Interface-between-Servlet-and-Tomcat/)中谈到的Servlet是由StandardWrapper的loadServlet()方法初始化的，那么查看loadServlet()方法应该就能找到答案了。

### StandardWrapper类中部分代码

```
//StandardWrapper中的属性
protected final StandardWrapperFacade facade = new StandardWrapperFacade(this);

public synchronized Servlet loadServlet() throws ServletException {
	Servlet servlet;
    try {
        ...
        InstanceManager instanceManager = ((StandardContext)getParent()).getInstanceManager();
        try {
            servlet = (Servlet) instanceManager.newInstance(servletClass);
        }
    }    

    ...

    initServlet(servlet);
    ...
}

private synchronized void initServlet(Servlet servlet)
            throws ServletException {
    ...
    servlet.init(facade);
    ...
}

```

我们惊讶地发现，servlet在init的时候将StandardWrapper的Facade类作为参数传递过去，并且在GenericServlet中将facade赋值给ServletConfig了。这时我们才惊喜的发现，原来**StandardWrapper实现了ServletConfig接口，而StandardWrapper暴露的信息过多，所以就将StandardWrapperFacade作为ServletConfig传递给了Servlet了**，这回终于真相大白。

注：Facade是一种设计模式，这里就不展开了。

## ServletContext

### 作用

**定义了一系列Servlet和Servlet容器沟通的方法。**比如获取一个文件的MIME类型，分发请求或者写日志文件。

**每一个Java虚拟机中的每一个Web应用都只有一个context。**

**ServletContext由ServletConfig.getServletContext()方法获取**

### 方法

ServletContext接口的方法比较罗，在此就不罗列了，需自行查看。

### 初始化

同样，我们沿用上面的逆向思维来找一下ServletContext的初始化代码。

首先，ServletContext是由ServletConfig.getServletContext()方法获取，而上面说到StandardWrapperFacade就是ServletConfig的实现，所以看看StandardWrapperFacade的getServletContext()方法：

```
  @Override
    public ServletContext getServletContext() {
        if (context == null) {
            context = config.getServletContext();
            if ((context != null) && (context instanceof ApplicationContext))
                context = ((ApplicationContext) context).getFacade();
        }
        return (context);
    }
```

可以看出，StandardWrapperFacade是一种Facade设计模式，它的实际调用用是StandardWrapper的getServletContext()方法,如果该方法能获取到context，并且该context是ApplicationContext的实例的话，则会返回ApplicationContext的Facade类。因此我们可以得出结论：

- **ServletContext在Tomcat中是以ApplicationContext对象存在的，ApplicationContext实现了ServletContext接口**
- **Tomcat中的ApplicationContext（ServletContext）和StandardWrapper（ServletConfig）都是以Facade的方式传递给Servlet，因此Servlet只能获取到对应的部分信息（Facade设计模式的作用）**

然后，我们再继续看一下StandardWrapper的getServletContext()方法：

```
    @Override
    public ServletContext getServletContext() {

        if (parent == null)
            return (null);
        else if (!(parent instanceof Context))
            return (null);
        else
            return (((Context) parent).getServletContext());

    }
```

还记得Wrapper的parent（即父容器）是谁吗？忘记了就要复习一下[Tomcat的Container和Lifecycle](http://rason.me/2015/11/23/Tomcat-s-Container-Lifecycle/)这篇文章了。Context就是Wrapper的父容器，在Tomcat中的标准实现是StandardContext。所以上述的方法会调用StandardContext的getServletContext()方法，让我们来看一下：

```
    @Override
    public ServletContext getServletContext() {

        if (context == null) {
            context = new ApplicationContext(this);
            if (altDDName != null)
                context.setAttribute(Globals.ALT_DD_ATTR,altDDName);
        }
        return (context.getFacade());

    }
```

至此，我们终于发现了ApplicationContext的最初实例化的地方了，也正如我们上面所说的，它会返回一个ApplicationContext的Facade实例。这里虽然是将ApplicationContext实例化了，但是相应的属性是在什么时候设置进去的呢？这时又回到了容器初始化的内容了，就是上篇文章[Tomcat与Servlet之间的接口](http://rason.me/2015/11/25/Interface-between-Servlet-and-Tomcat/)中谈到的ContextConfig的configureContext(webXml)方法了，这里就不再叙述了。

最后，我们终于知道了ServletContext在Tomcat中的实现类是ApplicationContext，而Servlet能获取到的是ApplicationContext的Facade实例。

## 总结

- **ServletConfig在Tomcat中的实现类是StandardWrapper**
- **ServletContext在Tomcat中的实现类是ApplicationContext**
- **Servlet能获取到的ServletConfig和ServletContext是StandardWrapper和ApplicationContext的Facade类**
- **Facade是一种设计模式，作用是只开放部分的内容给使用者，详细请搜索Facade设计模式**
