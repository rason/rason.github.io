---
title: 客户端访问RESTFul Web Service
date: 2017-02-21 11:03:35
categories: SpringBoot
tags: RestTemplate
description: Consuming a RESTful Web Service
---

## 前言

本文将学习创建一个应用来访问RESTFul Web Service。

### 你将会创建

你将会创建一个使用Spring的`RestTemplate`来获取Spring Boot的RESTFul服务 `http://gturnquist-quoters.cfapps.io/api/random`中的随机返回信息的应用。

### 你将需要

- 大约15分钟
- 一个熟悉的IDE
- JDK 1.8以上
- MAVEN 3.0以上

## 获取一个REST资源

`http://gturnquist-quoters.cfapps.io/api/random`是一个已存在的RESTFul服务，我们在浏览器中访问这个URL，会返回一个JSON格式的文档，类似这样：

```
{
   type: "success",
   value: {
      id: 10,
      quote: "Really loving Spring Boot, makes stand alone Spring apps easy."
   }
}
```

例子足够简单，但是直接通过浏览器来访问URL来获取不是很有用。

一个更加有用的方式来访问REST服务应该是使用编程的方式。为了完成这个任务，Spring提供了一个方便的模板类[RestTemplate](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/client/RestTemplate.html)。`RestTemplate`只需要一行代码就可以和RESTFul服务进行交互。它还可以绑定数据到自定义的领域对象。

<!-- more -->

首先，我们来创建一个包含我们需要数据的领域类。

```
package com.rason.resttemplate.entity;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

/**
 * Created by rason on 2/21/17.
 */
@JsonIgnoreProperties(ignoreUnknown = true)
public class Quote {

    private String type;
    private Value value;

    public String getType() {
        return type;
    }

    public void setType(String type) {
        this.type = type;
    }

    public Value getValue() {
        return value;
    }

    public void setValue(Value value) {
        this.value = value;
    }

    @Override
    public String toString() {
        return "Quote{" +
                "type='" + type + '\'' +
                ", value=" + value +
                '}';
    }
}

```

正如你所见，这个简单的Java类包含少数的属性和getter方法。然后使用Jackson JSON的`@JsonIgnoreProperties`注解来标志不存在的属性则被忽略。

为了使用你能够直接绑定数据到自定义的类型，你需要指定变量名需要和API返回的JSON文档的key一致。如果你的变量名跟JSON文档的key不一致，你可以用`@JsonProperty`注解指明属性对应哪个key。

然后创建附加的类Value。

```
package com.rason.resttemplate.entity;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;

/**
 * Created by rason on 2/21/17.
 */
@JsonIgnoreProperties(ignoreUnknown = true)
public class Value {

    private Long id;
    private String quote;

    public Value() {
    }

    public Long getId() {
        return this.id;
    }

    public String getQuote() {
        return this.quote;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public void setQuote(String quote) {
        this.quote = quote;
    }

    @Override
    public String toString() {
        return "Value{" +
                "id=" + id +
                ", quote='" + quote + '\'' +
                '}';
    }
}

```

## 使应用运行起来

现在我们来写一个Application类使用`RestTemplate`来从Spring Boot 服务中获取数据。

```
package com.rason.resttemplate.entity;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.client.RestTemplate;

/**
 * Created by rason on 2/21/17.
 */
public class Application {

    private static final Logger log = LoggerFactory.getLogger(Application.class);

    public static void main(String[] args) {
        RestTemplate restTemplate = new RestTemplate();
        Quote quote = restTemplate.getForObject("http://gturnquist-quoters.cfapps.io/api/random", Quote.class);
        log.info(quote.toString());
    }

}

```

因为Jackson JSON处理库在classpath中，RestTemplate会使用它将JSON数据转换成Quote对象。

到这里你只是使用了RestTemplate来发起一个HTTP GET请求。但是RestTemplate还支持其它方式的HTTP请求，比如POST，PUT和DELETE。

## 使用Spring Boot来管理Application的生命周期

到目前为止我们没有使用Spring Boot，但是使用它会有一些好处。其中一个好处就是如果我们想用Spring Boot来管理RestTemplate的消息转换器，我们可以很容易地用声明的方式来自定义。

```
package com.rason.resttemplate.entity;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.client.RestTemplateBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

/**
 * Created by rason on 2/21/17.
 */
@SpringBootApplication
public class Application {

    private static final Logger log = LoggerFactory.getLogger(Application.class);

    public static void main(String[] args) {
        SpringApplication.run(Application.class);
    }

    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder.build();
    }

    @Bean
    public CommandLineRunner run(RestTemplate restTemplate) throws Exception {
        return args -> {
            Quote quote = restTemplate.getForObject("http://gturnquist-quoters.cfapps.io/api/random", Quote.class);
            log.info(quote.toString());
        };
    }

}

```

这里的`RestTemplateBuilder`由Spring注入。

## 总结

现在，你已经学会了使用Spring来创建一个简单的REST客户端了。本文完整代码可以参考[这里](https://github.com/spring-guides/gs-consuming-rest)。


