title: Tomcat与Servlet之间的接口
categories: Tomcat
date: 2015-11-25 15:41:22
tags: Tomcat Servlet Interface
description: The interfaces between servlet and tomcat
---

## 前言

在[Tomcat架构](http://rason.me/2015/11/19/Tomcat-Architecture/)和[Tomcat的Container和Lifecycle](http://rason.me/2015/11/23/Tomcat-s-Container-Lifecycle/)两篇文章中基础性地介绍了Tomcat作为一个Servlet容器的基本架构和启动流程，还不清楚Tomcat和Servlet两者究竟是什么样的关系，Servlet又是如何能在Tomcat中正常工作的。今天这篇文章打算介绍一下Servlet与Tomcat之间的联系。

## StandardWrapper

在[Tomcat的Container和Lifecycle](http://rason.me/2015/11/23/Tomcat-s-Container-Lifecycle/)文章中，我们有说到`Wrapper`代表一个servlet定义，servlet在tomcat中是以`Wrapper`形式出现，也就是说我们的Servlet在Tomcat中被包装成Wrapper了。因此，要理解Servlet与Tomcat之间的关系，按思路就应该从Wrapper开始分析，而StandardWrapper是Wrapper的标准实现，所以我们来一探StandardWrapper的究竟。

<!-- more -->

### StandardWrapper的初始化

我们知道StandardWrapper的父容器是StandardContext，因此StandardWrapper的初始化工作应该在StandardContext中寻找。先看一下在[Tomcat的Container和Lifecycle](http://rason.me/2015/11/23/Tomcat-s-Container-Lifecycle/)文章中画的Tomcat启动流程图：

![Tomcat启动流程图](https://raw.githubusercontent.com/rason/rason.github.io/master/image/tomcat_start.png)

上图中**步骤12**，`ContextConfig`实现了生命周期监听器接口`LifecycleListener`监听StandardContext的生命周期状态变化,在StandardContext状态变化时会触发ContextConfig.lifecycleEvent方法。在StandardContext初始化时会触发ContextConfig的init方法，该方法中完成了每个Web应用的配置解析。

那么StandardContext初始化完毕之后，会在哪里初始化我们需要分析的StandardWrapper对象呢？上图中已经有了答案，在StandardContext的startInternal方法中会完成Web应用的初始化工作，那么**将我们的Servlet包装成Wrapper的工作也将在这里完成**。在startInternal方法中有一行代码：

```
  fireLifecycleEvent(Lifecycle.CONFIGURE_START_EVENT, null);
```

如果你是一个聪明人，就应该能猜到这行代码会做什么事情。没错，就是通知ContextConfig干活了，所以实际上**将Servlet包装成Wrapper的工作是在ContextConfig.configureStart()中完成的**。

### ContextConfig类部分代码

```
	// ContextConfig监听的生命周期事件
    public void lifecycleEvent(LifecycleEvent event) {
        if (event.getType().equals(Lifecycle.CONFIGURE_START_EVENT)) {
            configureStart();
        } else if (event.getType().equals(Lifecycle.BEFORE_START_EVENT)) {
            beforeStart();
        }else if (event.getType().equals(Lifecycle.CONFIGURE_STOP_EVENT)) {
            configureStop();
        } else if (event.getType().equals(Lifecycle.AFTER_INIT_EVENT)) {
            init();
        } else if (event.getType().equals(Lifecycle.AFTER_DESTROY_EVENT)) {
            destroy();
        }

    }
   
    protected synchronized void configureStart() {
        // Called from StandardContext.start()
        ...
        webConfig();
        ...
    }

    protected void webConfig() {
    	...
    	configureContext(webXml);
    	...
    }

```

从上面的代码我们可以看出，StandardContext的startInternal方法最终会调用到ContextConfig的`configureContext(webXml)`方法，**configureContext方法才是最后完成将Servlet包装成Wrapper的地方**，方法的参数就是将Web应用的配置信息解析成的Java对象。所以，我们重点来看一下这个方法是怎么将Servlet包装成Wrapper的。

```
   private void configureContext(WebXml webxml) {
   ...
        for (ServletDef servlet : webxml.getServlets().values()) {
            Wrapper wrapper = context.createWrapper();
            // Description is ignored
            // Display name is ignored
            // Icons are ignored

            // jsp-file gets passed to the JSP Servlet as an init-param

            if (servlet.getLoadOnStartup() != null) {
                wrapper.setLoadOnStartup(servlet.getLoadOnStartup().intValue());
            }
            if (servlet.getEnabled() != null) {
                wrapper.setEnabled(servlet.getEnabled().booleanValue());
            }
            wrapper.setName(servlet.getServletName());
            Map<String,String> params = servlet.getParameterMap();
            for (Entry<String, String> entry : params.entrySet()) {
                wrapper.addInitParameter(entry.getKey(), entry.getValue());
            }
            wrapper.setRunAs(servlet.getRunAs());
            Set<SecurityRoleRef> roleRefs = servlet.getSecurityRoleRefs();
            for (SecurityRoleRef roleRef : roleRefs) {
                wrapper.addSecurityReference(
                        roleRef.getName(), roleRef.getLink());
            }
            wrapper.setServletClass(servlet.getServletClass());
            MultipartDef multipartdef = servlet.getMultipartDef();
            if (multipartdef != null) {
                if (multipartdef.getMaxFileSize() != null &&
                        multipartdef.getMaxRequestSize()!= null &&
                        multipartdef.getFileSizeThreshold() != null) {
                    wrapper.setMultipartConfigElement(new MultipartConfigElement(
                            multipartdef.getLocation(),
                            Long.parseLong(multipartdef.getMaxFileSize()),
                            Long.parseLong(multipartdef.getMaxRequestSize()),
                            Integer.parseInt(
                                    multipartdef.getFileSizeThreshold())));
                } else {
                    wrapper.setMultipartConfigElement(new MultipartConfigElement(
                            multipartdef.getLocation()));
                }
            }
            if (servlet.getAsyncSupported() != null) {
                wrapper.setAsyncSupported(
                        servlet.getAsyncSupported().booleanValue());
            }
            wrapper.setOverridable(servlet.isOverridable());
            context.addChild(wrapper);
        }
   ...
   }
```

上面的方法省略了部分代码，除了将Servlet包装成StandardWrapper并作为子容器添加到Context中，还会将其它所有web.xml属性都会被解析到context中，所以说Context容器才是真正运行Servlet的Servlet容器。一个Web应用对应一个Context容器，容器的配置属性由应用的web.xml指定，这样我们就能理解web.xml究竟起什么作用了。

## Servlet的初始化

我们已经知道了Servlet在Tomcat中被包装成了StandardWrapper，那么Servlet的初始化工作当然由StandardWrapper来完成。

### StandardWrapper类部分代码

```
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
```

由上述代码可以看出Servlet的初始化工作比较简单，就不细说了。

## Context初始化流程图

![Context初始化流程图](https://raw.githubusercontent.com/rason/rason.github.io/master/image/tomcatcontext.png)

## 总结

- **Wrapper是Servlet与Tomcat之间的接口与纽带**
- **分析Wrapper的初始化要从父容器Context分析起**
- **Context的监听器类ContextConfig完成了Web应用的配置解析和相应初始化工作**
