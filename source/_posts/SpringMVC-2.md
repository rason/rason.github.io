title: 从“请求-响应”问题出发看一下SpringMVC的架构（下）
categories: Spring
date: 2015-11-12 09:26:35
tags: SpringMVC
description: springmvc spring架构 SpringMVC原理
---

上篇文章说到SpringMVC是由一个核心分发器DispatcherServlet来将请求分发到相应的controller进行处理，那么DispatcherServlet是怎么找到相应的controller和处理controller返回的信息呢？

带着这个问题来思考，DispatcherServlet作为一个核心分发器，当然是会有一套完整的处理“请求-响应”问题的规范化流程，我们可以想到这个流程大概是这样的：

- **1.对请求进行初步的处理，然后找出相应的controller**
- **2.调用相应的controller进行逻辑处理**
- **3.对controller处理过程中发生的异常进行处理**
- **4.根据controller处理的返回结果，进行http响应处理**

在SpringMVC中，对上述过程抽象出的接口就是：

- **1.HandlerMapping**
- **2.HandlerAdapter**
- **3.HandlerExceptionResolver**
- **4.ViewResolver**

以上这些接口称之为组件，SpringMVC除了上述组件外还有其它的组件可以查看官方文档，所以说**SpringMVC就是通过DispatcherServlet将一堆组件串联起来的Web框架**。
那么DispatcherServlet是怎么把这些组件组合在一起完成这个“请求-响应”过程呢？我们看一下DispatcherServlet的数据结构：

<!-- more -->

```
@SuppressWarnings("serial")
public class DispatcherServlet extends FrameworkServlet {

	/** Detect all HandlerMappings or just expect "handlerMapping" bean? */
	private boolean detectAllHandlerMappings = true;

	/** Detect all HandlerAdapters or just expect "handlerAdapter" bean? */
	private boolean detectAllHandlerAdapters = true;

	/** Detect all HandlerExceptionResolvers or just expect "handlerExceptionResolver" bean? */
	private boolean detectAllHandlerExceptionResolvers = true;

	/** Detect all ViewResolvers or just expect "viewResolver" bean? */
	private boolean detectAllViewResolvers = true;

	/** Throw a NoHandlerFoundException if no Handler was found to process this request? **/
	private boolean throwExceptionIfNoHandlerFound = false;

	/** Perform cleanup of request attributes after include request? */
	private boolean cleanupAfterInclude = true;

	/** MultipartResolver used by this servlet */
	private MultipartResolver multipartResolver;

	/** LocaleResolver used by this servlet */
	private LocaleResolver localeResolver;

	/** ThemeResolver used by this servlet */
	private ThemeResolver themeResolver;

	/** List of HandlerMappings used by this servlet */
	private List<HandlerMapping> handlerMappings;

	/** List of HandlerAdapters used by this servlet */
	private List<HandlerAdapter> handlerAdapters;

	/** List of HandlerExceptionResolvers used by this servlet */
	private List<HandlerExceptionResolver> handlerExceptionResolvers;

	/** RequestToViewNameTranslator used by this servlet */
	private RequestToViewNameTranslator viewNameTranslator;

	/** FlashMapManager used by this servlet */
	private FlashMapManager flashMapManager;

	/** List of ViewResolvers used by this servlet */
	private List<ViewResolver> viewResolvers;
}
```
可以看到DispatcherServlet包含了许多SpringMVC组件，这些组件就是实现DispatcherServlet核心逻辑的基础，我们看一下DispatcherServlet的核心源码，看看它是怎么把这些组件运用起来完成“请求-响应”这一过程。

```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	...

	// Determine handler for the current request.
	mappedHandler = getHandler(processedRequest);
	if (mappedHandler == null || mappedHandler.getHandler() == null) {
		noHandlerFound(processedRequest, response);
		return;
	}

	// Determine handler adapter for the current request.
	HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

	// Process last-modified header, if supported by the handler.
	String method = request.getMethod();
	boolean isGet = "GET".equals(method);
	if (isGet || "HEAD".equals(method)) {
		long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
		if (logger.isDebugEnabled()) {
			logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
		}
		if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
			return;
		}
	}

	if (!mappedHandler.applyPreHandle(processedRequest, response)) {
		return;
	}

	// Actually invoke the handler.
	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

	if (asyncManager.isConcurrentHandlingStarted()) {
		return;
	}

	applyDefaultViewName(request, mv);
	mappedHandler.applyPostHandle(processedRequest, response, mv);

	processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);

}
```

从上述的源码注释可以清楚地了解到核心的处理步骤跟我们前面分析设想的处理步骤是一样的，就是简单地获取相应的组件并进行调用。那么问题来了，这些组件是哪里来的呢？此时我们应该能想到SpringMVC的核心配置文件`[servlet-name]-servlet.xml`。

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://www.springframework.org/schema/p"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:mvc="http://www.springframework.org/schema/mvc" xmlns:task="http://www.springframework.org/schema/task"
	xsi:schemaLocation="http://www.springframework.org/schema/beans  
                        http://www.springframework.org/schema/beans/spring-beans-3.1.xsd  
                        http://www.springframework.org/schema/context  
                        http://www.springframework.org/schema/context/spring-context-3.1.xsd  
                        http://www.springframework.org/schema/mvc  
                        http://www.springframework.org/schema/mvc/spring-mvc-4.0.xsd
                        http://www.springframework.org/schema/task  
						http://www.springframework.org/schema/task/spring-task-3.1.xsd ">

    <context:component-scan base-package="com.rason.springmvc" />
	<mvc:annotation-driven />
	<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">  
        <property name="prefix" value="/" />  
        <property name="suffix" value=".jsp" />  
    </bean>  

</beans>
```

springmvc就是通过读取这个配置文件来获取各个组件的实现方式并对其进行管理。我们上面说到每一个组件都定义为一个接口，那么接口肯定是可以有多种实现方式的，因此上面的核心配置文件配置的都是具体的实现类。SpringMVC各种不同的组件实现体系成为了SpringMVC行为模式扩展的有效途径。至于每个组件的各种实现行为模式有何不同，这里就不继续探讨了，可以自行查看源码的注释了解。

到目前为止，我们应该可以大概地回答文章开头的问题了。DispatcherServlet的数据结构中就包含了各种组件的引用，通过核心配置文件读取到组件的具体实现方式，然后按照上面的四个处理步骤相继进行处理找到相应的controller进行处理并返回结果（**见上面的核心源码doDispatch方法**）。

那么现在的疑惑应该是DispatcherServlet究竟是怎么读取核心配置文件获取具体的组件呢？让我们来了解DispatcherServlet的初始化过程：

在了解DispatcherServlet初始化过程之前，我们首先得了解一下它的继承结构。

![DispatcherServlet继承结构](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springmvcsnapshot2.png)

在这棵继承树中，我们可以看到`DispatcherServlet`继承成与`FrameworkServlet`，而`FrameworkServlet`继承于`HttpServletBean`，我们分别来了解一下这两个类。

HttpServletBean是Spring对于Servlet最低层次的抽象。在这一层抽象中，Spring会将这个Servlet视作是一个Spring的bean，并将init-param中的值作为bean的属性注入进来：

```
/**
 * Map config parameters onto bean properties of this servlet, and
 * invoke subclass initialization.
 * @throws ServletException if bean properties are invalid (or required
 * properties are missing), or if subclass initialization fails.
 */
@Override
public final void init() throws ServletException {
	if (logger.isDebugEnabled()) {
		logger.debug("Initializing servlet '" + getServletName() + "'");
	}

	// Set bean properties from init parameters.
	try {
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
		ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
		bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
		initBeanWrapper(bw);
		bw.setPropertyValues(pvs, true);
	}
	catch (BeansException ex) {
		logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
		throw ex;
	}

	// Let subclasses do whatever initialization they like.
	initServletBean();

	if (logger.isDebugEnabled()) {
		logger.debug("Servlet '" + getServletName() + "' configured successfully");
	}
}
```

从源码中，我们可以看到HttpServletBean利用了Servlet的init方法的执行特性，将一个普通的Servlet与Spring的容器联系在了一起。在这其中起到核心作用的代码是：`BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this)`;将当前的这个Servlet类转化为一个BeanWrapper，从而能够以Spring的方式来对init-param的值进行注入。BeanWrapper的相关知识属于Spring Framework的内容，我们在这里不做详细展开，可以具体参考HttpServletBean的注释获得更多的信息。 


`FrameworkServlet`则是在HttpServletBean的基础之上的进一步抽象。通过FrameworkServlet真正初始化了一个Spring的容器（WebApplicationContext），并引入到Servlet对象之中：
```
/**
 * Overridden method of {@link HttpServletBean}, invoked after any bean properties
 * have been set. Creates this servlet's WebApplicationContext.
 */
@Override
protected final void initServletBean() throws ServletException {
	getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
	if (this.logger.isInfoEnabled()) {
		this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
	}
	long startTime = System.currentTimeMillis();

	try {
		this.webApplicationContext = initWebApplicationContext();
		initFrameworkServlet();
	}
	catch (ServletException ex) {
		this.logger.error("Context initialization failed", ex);
		throw ex;
	}
	catch (RuntimeException ex) {
		this.logger.error("Context initialization failed", ex);
		throw ex;
	}

	if (this.logger.isInfoEnabled()) {
		long elapsedTime = System.currentTimeMillis() - startTime;
		this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
				elapsedTime + " ms");
	}
}
```

从这段代码可以看到这个FrameworkServlet将调用其内部的方法initWebApplicationContext()对Spring的容器（WebApplicationContext）进行初始化。同时，FrameworkServlet还暴露了`public final WebApplicationContext getWebApplicationContext()`可供子类DispatcherServlet调用。因此，**DispatcherServlet的继承体系架起了DispatcherServlet与Spring容器进行沟通的桥梁**。

由此，我们可以总结得出**SpringMVC的运行体系是由DispatcherServlet、组件和容器这三者共同构成的**。在这个运行体系中，**DispatcherServlet是逻辑处理的调度中心，组件则是被调度的操作对象。而容器在这里所起到的作用，是协助DispatcherServlet更好地对组件进行管理**。引用spring官方的示意图简单描述三者关系：

![SpringMVC运行体系](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springmvcstruct.png)

三者之间的两两关系可以归纳为：

- **DispatcherServlet - 容器 —— DispatcherServlet对容器进行初始化**
- **容器 - 组件 —— 容器对组件进行全局管理**
- **DispatcherServlet - 组件 —— DispatcherServlet对组件进行逻辑调用**

明确了DispatcherServlet的继承体系和DispatcherServlet，容器，组件三者之间关系之后，用一张图来看看DispatcherServlet的初始化过程：

![DispatcherServlet初始化过程](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springmvcinit.png)

细心的读者应该注意到了上图中左侧还有一个WebApplicationContext的初始化，应该会疑惑这个WebApplicationContext是干什么用的，如果你做过SpringMVC的开发应该曾经配置过这样一段代码：

```
	<context-param>  
	    <param-name>contextConfigLocation</param-name>  
	    <param-value>classpath:context/applicationContext-*.xml</param-value>  
	</context-param>  
	      
	<listener>  
	    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
	</listener>  
```

在上面的代码中，我们定义了一个Listener，它会在整个Web应用程序启动的时候运行一次，并初始化传统意义上的Spring的容器。这也是一般情况下，当并不使用SpringMVC作为我们的表示层解决方案，却希望在我们的Web应用程序中使用Spring相关功能时所采取的一种配置方式(比如配置使用第三方框架mybatis,shiro等)。 **ContextLoaderListener所初始化的容器，我们称之为Root WebApplicationContext；而DispatcherServlet所初始化的容器，是SpringMVC WebApplicationContext。**

![Context](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springmvccontext.png)

说到这里，SpringMVC的核心DispatcherServlet也差不多是了解了个大概了，再详细的就不再往下写了。简单地总结一下这篇文章，就是先介绍了DispatcherServlet的继承体系，然后从中道出了DispatcherServlet的初始化过程。而在初始化过程中最主要的就是在FrameworkServlet的初始化方法中初始化了WebApplicationContext，这个WebApplicationContext就是spring的容器，用于管理各个组件。最后缕清了一下WebApplicationContext，容器，组件这三者之间的关系。

最后的最后，引用一个spring官方的图片来总结SpringMVC的架构来结束文章。

![SpringMVC](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springmvcspringmvc.png)
