---
title: Spring Cloud ConfigServer
date: 2017-04-19 09:21:00
categories: SpringCloud
tags: SpringCloud
description: SpringCloud ConfigServer
---

上一篇文章主要学习了使用`Eureka`作为服务的注册和发现中心，各个微服务注册到`Eureka`中可以互相发现并进行调用。今天将会继续学习另一个基础组件——配置中心。如下图所示：

![模型](/image/Sample-ping-pong.png)

这个架构模型包含两个微服务：

- `sample-pong`服务响应`ping`消息
- `sample-ping`服务调用`sample-pong`微服务

另外还包含两个基础组件：

- `Sample-config`为上面两个微服务提供中心化配置
- `Eureka`是中心枢纽，提供服务的注册和发现功能

所以，今天我们主要学习的就是中心化配置服务组件。

首先我们先将两个基础组件搭建起来。

## Eureka

先将一个Eureka实例跑起来，很简单，只需要几行代码：

```
package com.rason.eureka;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

/**
 * Created by rason on 4/18/17.
 */
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaApplication.class, args);
    }

}

```
Eureka可以弹性地配置多实例，这里为了简单演示起见我们只运行一个单实例，配置文件如下：

<!-- more -->

```
# application.yml
server:
  port: 8761
 
eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
```

## Configuration Server(配置中心服务)

Spring Cloud提供了一个中心化配置服务组件，使得微服务可以使用它来加载自身的属性文件。

在本例子中`Configuration server`将自身注册到Eureka，当微服务启动时首先检查Eureka，找到配置中心服务，然后通过其加载他们的属性文件。

使用Spring Cloud 将配置服务跑起来也很简单：

```
package com.rason.configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

/**
 * Created by rason on 4/18/17.
 */
@SpringBootApplication
@EnableConfigServer
@EnableEurekaClient
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }

}

```

然后配置将其注册到Eureka：

```
# bootstrap.yml
spring:
  application:
    name: sample-config
  profiles:
    active: native
  cloud:
    config:
      server:
        native:
          searchLocations: classpath:/config

```


```
# application.yml
server:
  port: 8888

eureka:
  instance:
    nonSecurePort: ${server.port:8888}
  client:
    serviceUrl:
      defaultZone: http://${eureka.host:localhost}:${eureka.port:8761}/eureka/
```

这里配置服务使用8888端口启动，并且从classpath中获取配置信息。在实际的应用中，中心化的配置文件可以设置为从git仓库中加载，这样可以集中管理各版本属性文件。我们的例子程序只是简单地为两个微服务提供属性配置，所以在classpath中会有两个微服务对应的属性文件。

```
#sample-pong.yml
reply:
  message: Pong
```


```
# sample-ping.yml
send:
  message: Ping
```

## 运行Eureka 和 Configuration Server

你可以使用IDE运行，也可以使用`mvn spring-boot:run`命令行来运行。

启动之后我们在浏览器中访问[http://localhost:8761](http://localhost:8761)。在Eureka服务中我们可以看到配置服务`SAMPLE-CONFIG`已经注册到Eureka中。

![Eureka界面](/image/configserver.png)

配置服务会通过`/{application}/{profile}[/{label}`这样的端点模式为调用程序提供属性文件。

所以`sample-pong`应用想要获取它的属性文件，需要在应用内部使用以下URL来获取：

```
http://localhost:8888/sample-pong/default
```

返回的结果如下：

```
{
  "name": "sample-pong",
  "profiles": [
    "default"
  ],
  "label": null,
  "version": null,
  "state": null,
  "propertySources": [
    {
      "name": "classpath:/config/sample-pong.yml",
      "source": {
        "reply.message": "Pong From Configuration Server",
        "logging.level.root": "INFO"
      }
    }
  ]
}
```

而`sample-ping`则通过`http://localhost:8888/sample-ping/default`来获取。

## 总结

由于篇幅关系，今天就先学习将两个基础组件搭建起来。下一篇文章将会学习将利用这两个基础组件将微服务搭建起来。工程的完整代码可以在[我的GitHub](https://github.com/rason/spring-cloud-sample)上下载。
