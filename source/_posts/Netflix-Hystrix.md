---
title: Netflix Hystrix
date: 2017-04-28 15:00:45
categories: SpringCloud
tags: Hystrix
description: Netflix Hystrix
---

## Circuit Breaker(断路由)

本文将简单介绍使用Netflix Hystrix容错库将断路器应用于潜在故障的方法调用。我们将会创建两个微服务，一个`Bookstore`服务和一个`Reading`服务。`Reading`服务会调用`Bookstore`服务读取书籍列表。使用Netflix Hystrix，如果服务调用成功则会返回正常的数据；如果服务调用失败，则会调用相应的失败方法。

## Bookstore微服务

Bookstore微服务很简单，只有一个endpoint，返回字符串格式的书籍列表。

```
package com.rason.bookstore;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * Created by rason on 4/28/17.
 */
@SpringBootApplication
@RestController
public class BookstoreApplication {

    @RequestMapping("/recommended")
    public String readingList() {
        return "Spring in Action (Manning), Cloud Native Java (O'Reilly), Learning Spring Boot (Packt)";
    }


    public static void main(String[] args) {
        SpringApplication.run(BookstoreApplication.class);
    }

}

```

我们还需要配置应用的端口，在resources文件夹下面创建`application.yml`:

```
server:
  port: 8090
```

<!-- more -->

## Reading微服务

Reading微服务也很简单，只有一个endpoint，该endpoint会使用RestTemplate调用Bookstore微服务。

```
package com.rason.reading;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import java.net.URI;

/**
 * Created by rason on 4/28/17.
 */
@SpringBootApplication
@RestController
public class ReadingApplication {

    @RequestMapping("/toRead")
    public String readingList() {
        RestTemplate restTemplate = new RestTemplate();
        URI uri = URI.create("http://localhost:8090/recommended");

        return restTemplate.getForObject(uri, String.class);
    }

    public static void main(String[] args) {
        SpringApplication.run(ReadingApplication.class);
    }

}

```

同样也要为该微服务配置端口：

```
server:
  port: 8080
```

然后我们启动两个微服务，访问`http://localhost:8080/toRead`，会返回正确的结果。但是，如果我们没有启动Bookstore微服务，则会返回500错误。`Netflix Hystrix`正是为了解决这样的问题而出现的，当调用其它服务超时或者失败时，提供后备处理方案。

## 使用Circuit Breaker模式

Netflix的 Hystrix库提供断路由模式：当我们将断路由模式应用到一个方法，Hystrix会监控该方法的调用失败次数，如果失败达到设定的阈值，Hystrix会断开路由即后来的调用会自动失败。当路由打开，Hystrix将调用重定向，将它们传递到我们指定的回退方法。使用起来相当简单，只需要加入`spring-cloud-starter-hystrix`依赖和使用相应的注解即可。

现在，我们来增加一个BookService，加入Circuit Breaker模式：

```
package com.rason.reading;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

import java.net.URI;

/**
 * Created by rason on 4/28/17.
 */
@Service
public class BookService {

    @Autowired
    RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "reliable")
    public String readingList() {
        URI uri = URI.create("http://localhost:8090/recommended");
        return restTemplate.getForObject(uri, String.class);
    }

    public String reliable() {
        return "Cloud Native Java (O'Reilly)";
    }

}

```

如上所示，只需要在调用方法上添加`@HystrixCommand`，然后设置其失败后的调用方法名即可，这里是`reliable`方法，我们在reliable方法中返回后备的结果。

Spring Cloud Netflix Hystrix 将会查找使用`@HystrixCommand`注释注释的任何方法，并将该方法包含在连接到断路由的代理中，以便Hystrix可以监视它。目前只能在被标记为`@Component` 或者 `@Service` 的类中使用`@HystrixCommand`，所以我们抽出了一个新的`BookService`类。

然后，稍微简单地改造一下ReadingApplication：

```
package com.rason.reading;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;
import org.springframework.context.annotation.Bean;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import java.net.URI;

/**
 * Created by rason on 4/28/17.
 */
@SpringBootApplication
@RestController
@EnableCircuitBreaker
@EnableHystrixDashboard
public class ReadingApplication {

    @Autowired
    BookService bookService;

    @Bean
    RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.build();
    }

    @RequestMapping("/toRead")
    public String readingList() {
        return bookService.readingList();
    }

    public static void main(String[] args) {
        SpringApplication.run(ReadingApplication.class);
    }

}

```

## 测试

当我们两个微服务都启用，访问`http://localhost:8080/toRead`会返回

```
Spring in Action (Manning), Cloud Native Java (O'Reilly), Learning Spring Boot (Packt)
```

此时，如果我们将Bootstore微服务关闭，再次访问则会返回：

```
Cloud Native Java (O'Reilly)
```

## 总结

我们学会了使用 Circuit Breaker模式来预防级联失败和为潜在失败可能的方法提供后备处理方法。
