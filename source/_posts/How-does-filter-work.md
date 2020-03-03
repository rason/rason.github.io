title: Filter是如何工作的
categories: Tomcat
date: 2015-12-03 09:46:54
tags: Filter
description: How does filter work
---

## 前言

在上篇文章[Tomcat是怎么找到特定的Servlet来处理请求](http://rason.me/2015/12/01/How-do-Tomcat-find-the-specific-servlet/)中谈到在请求正式进入Servlet之前，会先执行Filter链。那么这个Filter链究竟是什么东西？Filter是如何工作的？

## Filter是什么

正如名字一样，Filter是过滤器，过滤器的作用当然是过滤掉一些东西。比如，我们日常用的洗手盆，在排水处会有个滤网，就是为了避免大块的物品流水了下水管导致阻塞。同理，Servlet技术中的Filter作用也是一样的，就是在请求到达Servlet之前先对请求进行过滤。那么，这样的一个功能，用代码是如何实现的？请先看一个示意图：

![Filter工作示意图](/image/tomcatFilterWork.png)

从上图可以看出，请求在到达Servlet之前会经过若干个Filter，而这些Filter是保存在一个FilterChain里面。

<!-- more -->

## Filter工作原理

我们假象一下Filter的工作原理应该是这样的，FilterChain对象里面保存了一个数组，数组里面装着若干个Filter。在请求到达Servlet之前，会先挨个执行数组里面的Filter，执行完了之后再执行Servlet的service方法。而从Tomcat的源码中也印证了这一点：

```
	StandardWrapperValve类的invoke方法

    @Override
    public final void invoke(Request request, Response response)
        throws IOException, ServletException{
        ...
        	  // Create the filter chain for this request
        ApplicationFilterChain filterChain =
                ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);

        ...
        filterChain.doFilter(request.getRequest(),
                                    response.getResponse());
    }

    ApplicationFilterChain类internalDoFilter方法

    private void internalDoFilter(ServletRequest request,
                                  ServletResponse response)
        throws IOException, ServletException {
        	// Call the next filter if there is one
	  		if (pos < n) {
	            ApplicationFilterConfig filterConfig = filters[pos++];
	            Filter filter = null;
	            try {
	                filter = filterConfig.getFilter();
	            }
	            ...
	            filter.doFilter(request, response, this);
	        }

            // We fell off the end of the chain -- call the servlet instance
            servlet.service(request, response);
    }
```

## Filter在Tomcat中的调用

![Tomcat中Filter调用过程](/image/tomcatfilter.png)

## Filter来自哪里

从上面的示意图我们看到有若干个Filter，那么究竟有多少个，会不会没有，Filter究竟是来自哪里？这时我们应该就想起了应用的web.xml文件了，和Servlet类似，Filter也是通过名字和匹配路径配置在web.xml文件中，即`<filter>`和`<filter-mapping>`。我们发现Filter的配置和Servlet的配置是极其的相似，其实Filter可以完成与Servlet一样的工作，因为他们的接口也是类似的。

```
public interface Filter {

    public void init(FilterConfig filterConfig) throws ServletException;

    public void doFilter(ServletRequest request, ServletResponse response,
        FilterChain chain) throws IOException, ServletException;

    public void destroy();
}
```

可以看出，Filter和Servlet的生命周期都是何其的类似。所以在理解Servlet的基础上理解Filter是一件非常简单的事情，比如FilterConfig可以类比ServletConfig，不同点就是Filter提供了FilterChain对象，所以在控制请求流转问题上比Servlet使用起来会更加灵活。

## 总结

通过这篇文章，我们应该学到以下内容：

- **Filter是过滤作用，FilterChain保存了匹配的Filter，会挨着执行完这些Filter才会执行Servlet的service方法**
- **Filter在Tomcat中的调用过程，见上面的流程图**
- **Filter与Servlet类似，可以类比学习**
