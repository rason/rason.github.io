---
title: Netflix Zuul
date: 2017-05-02 10:51:18
categories: SpringCloud
tags: Zuul
description: Netflix Zuul Routing and Filtering
---

## Routing and Filtering

本文将使用Netflix Zuul边缘服务库演示将请求路由和过滤到微服务应用程序的过程。

我们将会创建一个简单的微服务应用，然后使用[Netflix Zuul](https://github.com/spring-cloud/spring-cloud-netflix)创建一个反向代理应用来将请求转发到服务应用。您还将了解如何使用Zuul的过滤器功能。

## 创建一个微服务

```
package com.rason.book;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Created by rason on 5/2/17.
 */
@RestController
@SpringBootApplication
public class BookApplication {

    @RequestMapping("/available")
    public String available() {
        return "Spring in Action";
    }

    @RequestMapping("/check-out")
    public String checkOut() {
         return "Spring boot in Action";
    }

    public static void main(String[] args) {
        SpringApplication.run(BookApplication.class);
    }

}

```

`Book`服务非常简单，就是一个REST控制器。为了不和反向代理应用端口冲突，我们配置一下`Book`服务的启动端口：

```
# application.yml

server:
  port: 8090
```

## 创建反向代理服务

Spring Cloud Netflix 包含一个嵌入式的Zuul 代理，通过`@EnableZuulProxy`注解即可开启。这将使`Gateway`应用变成反向代理，将相关请求转发到其他服务（比如我们的`Book`服务）。

<!-- more -->

```
package com.rason.gateway;

import com.rason.gateway.filter.post.SimplePostFilter;
import com.rason.gateway.filter.pre.SimpleFilter;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
import org.springframework.context.annotation.Bean;

/**
 * Created by rason on 5/2/17.
 */
@EnableZuulProxy
@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class);
    }

}

```

Zuul要转发来自`Gateway`的请求，则必须需要知道如何进行路由。所以我们需要通过`zuul.routes`属性来指定路由规则。

```
# application.yml

server:
  port: 8087

zuul:
  routes:
    books:
      url: http://localhost:8090
```

上面的配置表明路径为`/books`的请求都将转发到`http://localhost:8090`应用上，也就是我们的`Book`服务。如果我们使用Eureka进行服务发现，也可以通过指定应用名字的方式进行转发。

## 测试

现在，我们同时启动两个应用，在浏览器中输入`http://localhost:8087/books/available`，会返回`Spring in Action`。这说明请求确实是通过网关应用转发到了`Book`服务。

## 添加过滤器

现在我们来看下如果通过我们的代理服务对请求进行过滤。Zuul有四种标准的过滤器类型：

- **pre** 在请求被路由之前执行过滤
- **route** 可以控制请求实际被路由的地方
- **post** 请求处理之后执行过滤
- **error** 请求处理发生异常时执行

我们来创建一个简单的前置过滤器：

```
package com.rason.gateway.filter.pre;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.http.HttpServletRequest;

/**
 * Created by rason on 5/2/17.
 */
public class SimpleFilter extends ZuulFilter {

    private static Logger log = LoggerFactory.getLogger(SimpleFilter.class);

    @Override
    public String filterType() {
        return "pre";
    }

    @Override
    public int filterOrder() {
        return 1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        log.info(String.format("%s request to %s", request.getMethod(), request.getRequestURL().toString()));

        return null;
    }
}

```

继承ZuulFilter类需要实现四个方法：

- `filterType()` 过滤器类型，就是上面的四种类型
- `filterOrder()` 过滤器执行顺序，多个过滤器时指定顺序
- `shouldFilter()` 可以通过逻辑代码进行判断是否执行过滤
- `run()` 过滤器执行的内容

Zuul过滤器将请求和状态信息保存在`RequestContext`中。通过`RequestContext`我们可以获取到`HttpServletRequest`，然后将HTTP请求方法和URL在请求转发之前打印出来。

当然，我们需要在应用上下文中能找到这个过滤器才能进行过滤。所以，我们需要在应用启动的时候将其初始化，然后交给应用上下文管理，通过`@Bean`注解即可实现。

```
package com.rason.gateway;

import com.rason.gateway.filter.post.SimplePostFilter;
import com.rason.gateway.filter.pre.SimpleFilter;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
import org.springframework.context.annotation.Bean;

/**
 * Created by rason on 5/2/17.
 */
@EnableZuulProxy
@SpringBootApplication
public class GatewayApplication {

    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class);
    }

    @Bean
    public SimpleFilter simpleFilter() {
        return new SimpleFilter();
    }

}

```

## 测试

现在我们重启一下网关应用`GatewayApplication`。再次访问`http://localhost:8087/books/available`，会看到一下日志输出：

```
2017-05-02 16:56:31.505  INFO 12929 --- [nio-8087-exec-2] c.rason.gateway.filter.pre.SimpleFilter  : GET request to http://localhost:8087/books/available
```

## 总结

本文学习来使用Zuul创建一个简单的网关服务，可以对请求进行路由和过滤。
