title: 分布式Session基本思路
categories: Java Web
date: 2015-12-14 10:59:48
tags: distributed-session
description: distributed-session 分布式Session 分布式Session基本思路
---

## 前言

上篇文章说到Cookie和Session的原理和作用，Session可以解决Cookie存在的一些问题。由于Session是存在每一台应用服务器中，所以在分布式系统中,Tomcat中的Session管理方式就不能满足我们的需求了。因为每台机器都保存在不同的Session，所以随机访问不同的服务器不能解决身份识别问题。另外，在大型的系统中通常会有很多的独立子系统，而他们有可能是公用同一域名，所以当系统太多Cookie管理起来也是一个问题。抛给我们的问题就是：如何解决分布式系统中Session共享，并且能够对Cookie进行统一管理？

## 思路

**问题1：**如何解决Session共享？

很显然，现在的问题是每台应用服务器都保存和管理自身的Session，这是无法共享的。因此**我们需要将Tomcat中Session管理这一块功能单独抽离出来自行实现保存和管理工作,**。

大概的思路应该是找一统一的地方保存Session，在`request.getSession()`方法中返回统一地方保存的地方即可。由于Cookie本身就是分布式的，根据客户端发送过来的JESSIONID到统一保存Session的地方获取Session，则可解决共享问题，这个统一的地方可以是分布式缓存，比如MemCache。

<!-- more -->

具体怎么实现？

就拿Tomcat服务器而言，现有的HttpServletRequest获取到的Session是StandardSession的Facade对象。所以首先得改造的就是`request.getSession()`方法的实现，因此我们可以做一个全局的Filter，将请求拦截下来替换成我们自己管理的Session。

```
public class DistributedSessionFilter implements Filter {
	private CustomSessionManager sessionManager;
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
                                                                                             throws IOException,
                                                                                             ServletException {
        
        CustomRequestWrapper req = new CustomRequestWrapper(request, sessionManager);
        chain.doFilter(req, response);
    }
}

```

上述的简要代码将请求拦截下来，封装我们自己的ServletRequest，为了就是能够重写getSession()方法，获取我们自己管理的Session。另一个参数sessionManager是我们自定义的Session管理器，管理Session的生命周期。

```
public class CustomRequestWrapper{
	@Override
    public HttpSession getSession(){
    	session = sessionManager.findSession(requestedSessionId);
    	if(session == null){
    		session = sessionManager.createSession();
    	}
	}
}

```

上述的方法是示意作用，具体实现可以模仿Tomcat的获取Session方法，主要是把Session的获取和创建替换成我们自定义的管理器获取和创建即可。

```
public class CustomSessionManager{
	// 保存到分布式缓存，从分布式缓存获取等一系列方法
}
```

分布式的Session解决方案大致思路如此。

**问题2：**如何解决Cookie的集中式管理？

这个问题的解决思路应该差不多，我们用服务订阅服务器统一管理Cookie的配置即可。应用可以从该订阅服务器中获取可以读写那些Cookie和Session，这样就能达到统一管理的作用，Zookeeper正是为了解决这类型问题而生。

## 方案图

综上，分布式Session的解决方案大致如图所示：

![分布式Session管理](https://raw.githubusercontent.com/rason/rason.github.io/master/image/javaweb-distributed-session.png)

以上，本文只是一个非常简单的概要思路，并没有严格考虑很多问题，仅供开脑洞。
