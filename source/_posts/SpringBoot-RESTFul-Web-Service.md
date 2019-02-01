---
title: 使用SpringBoot来创建RESTFul Web Service
categories: SpringBoot
date: 2017-02-20 10:43:08
tags: SpringBoot
description: SpringBoot RESTFul Web Service
---

## 前言

本文将带领你使用Spring来创建一个简单的RESTFul Web Service。

### 你将会创建

你将会创建一个接受HTTP GET请求的服务：

```
http://localhost:8080/greeting
```

然后响应一个JSON格式的数据：

```
{"id":1,"content":"Hello, World!"}
```

如果你自定义`name`参数，比如

```
http://localhost:8080/greeting?name=User
```

则name参数会覆盖默认的`World`:

```
{"id":1,"content":"Hello, User!"}
```

<!-- more -->

### 你需要

- 大约15分钟
- 一个你熟悉的IDE
- JDK1.8以上
- MAVEN3.0以上

## 创建一个资源代表类

在开始创建之前，我们先想一下服务是如何交互的。

这个服务将会处理路径`/greeting`的`GET`请求，并且有一个可选的`name`参数。这个`GET`请求应该返回状态为`200 OK`，并且响应体为`JSON`格式，就好像这样：

```
{
    "id": 1,
    "content": "Hello, World!"
}
```

这个`id`属性代表这个greeting的唯一标识，`content`属性代表greeting的文本内容。

为了模型化这个greeting返回信息，我们需要创建一个资源代表类。

```
package com.rason.restfulservice.entity;

/**
 * Created by rason on 2/20/17.
 */
public class Greeting {
    
    private final long id;
    private final String content;

    public Greeting(long id, String content) {
        this.id = id;
        this.content = content;
    }

    public long getId() {
        return id;
    }

    public String getContent() {
        return content;
    }
}

```

> 在后面你会看到，Spring会使用[Jackson JSON](http://wiki.fasterxml.com/JacksonHome)库来将`Greeting`实例自动转换成JSON格式。

下一步我们将创建一个资源控制器来服务这些greeting。

## 创建一个资源控制器

在Spring中创建一个RESTFul的web服务，HTTP请求是通过控制器来处理的。这些组建都可以很简单地通过`@RestController`注解来实现，下面的GreetingController将处理路径`/greeting`的GET请求，并且返回一个新的`Greeting`类的实例。

```
package com.rason.restfulservice.controller;

import com.rason.restfulservice.entity.Greeting;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.atomic.AtomicLong;

/**
 * Created by rason on 2/20/17.
 */
@RestController
public class GreetingController {

    private static final String template = "Hello, %s!";
    private final AtomicLong counter = new AtomicLong();

    @RequestMapping("/greeting")
    public Greeting greeting(@RequestParam(value = "name", defaultValue = "World") String name) {
        return new Greeting(counter.incrementAndGet(),String.format(template, name));
    }
}

```

这个例子虽然很简单，但是包含了不少知识，我们来一起看看。

`@RequestMapping`注解确保了HTTP请求`/greeting`路径会匹配到`greeting()`方法。

> 上面的例子没有特别知名是`GET`还是`PUT`、`POST`，因为`@RequestMapping`会默认匹配所有的HTTP操作请求。使用`@RequestMapping(method=GET)`则可以指定只匹配`GET`方法。

`@RequestParam`绑定参数`name`到greeting()方法。这个查询字符串参数明确标记为可以选的（默认为required=true）：如果请求中没有传name参数，则默认使用`World`。方法的实现返回一个新的Greeting对象。

传统MVC控制和RESTFul web service控制器之间一个关键的不同点就是HTTP响应体的创建方式。相对于使用视图技术来展示服务器端的返回数据到HTML页面，RESTFul web service控制器简单地封装并返回一个Greeting对象。这个对象数据会被转化为JSON格式直接写到HTTP响应体。

代码使用了Spring 4的新注解`@RestController`，被它标记的类将作为一个控制器，并且每个方法都应该返回一个领域对象而不是一个视图。它相当于同时使用`@Controller` 和 `@ResponseBody`来标记一个类。

这里的`Greeting`对象必须被转换成JSON。得益于Spring的HTTP消息转换器支持，我们不需要手工进行处理转换。因为`Jackson 2`在类路径中，Spring的[MappingJackson2HttpMessageConverter](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/http/converter/json/MappingJackson2HttpMessageConverter.html)会自动被选择将`Greeting`实例转换成JSON。

## 使应用运行起来

虽然我们可能需要将服务打包成传统的WAR文件然后发布到一个应用服务器中，为了简化演示我们来创建一个单独的应用。

```
package com.rason.restfulservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * Created by rason on 2/20/17.
 */
@SpringBootApplication
public class Application {

    public static void main(String[] args){
        SpringApplication.run(Application.class, args);
    }

}

```

### 测试服务

在浏览器中输入

```
http://localhost:8080/greeting
```

你会看到

```
{"id":1,"content":"Hello, World!"}
```

现在，我们在name参数中传入查询字符串，比如

```
http://localhost:8080/greeting?name=rason
```

你会看到

```
{"id":2,"content":"Hello, rason!"}
```

## 总结

现在我们已经学会了使用Spring来创建一个简单的RESTful web service。本文完整代码可以参考[这里](https://github.com/spring-guides/gs-rest-service)。
