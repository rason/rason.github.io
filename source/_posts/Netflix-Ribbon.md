---
title: Netflix Ribbon
date: 2017-05-03 14:59:54
categories: SpringCloud
tags: Ribbon
description: Client Side Load Balancing with Ribbon and Spring Cloud
---

## 使用Ribbon 和 Spring Cloud 实现客户端负载均衡

本文将学习使用Netflix Ribbon对微服务进行负载均衡访问。

我们将会创建一个客户端应用和一个服务端应用，先演示单个服务端实例单点访问，再演示多个服务端实例负载均衡访问。

## 服务端应用

服务端应用相当简单，是一个REST控制器类，随机返回一个问候语。

```
package com.rason.server;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Random;

/**
 * Created by rason on 5/3/17.
 */
@SpringBootApplication
@RestController
public class ServerApplication {

    private static final Logger logger = LoggerFactory.getLogger(ServerApplication.class);

    @RequestMapping("/greeting")
    public String greet() {
        logger.info("Access /greeting");

        List<String> greetings = Arrays.asList("Hi there", "Greetings", "Salutations");
        Random rand = new Random();

        int randomNum = rand.nextInt(greetings.size());
        return greetings.get(randomNum);
    }

    @RequestMapping(value = "/")
    public String home() {
        logger.info("Access /");
        return "Hi!";
    }

    public static void main(String[] args) {
        SpringApplication.run(ServerApplication.class);
    }

}

```

此外，我们还需要配置服务端的启动端口。

```
# application.yml
server:
  port: 8090

spring:
  application:
    name: my-server
```

## 客户端应用

客户端应用也相当简单，同样是REST控制器类。使用`RestTemplate`对服务端进行访问。

<!-- more -->

```
package com.rason.client;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

/**
 * Created by rason on 5/3/17.
 */
@SpringBootApplication
@RestController
public class ClientApplication {

    @Bean
    RestTemplate restTemplate(){
        return new RestTemplate();
    }

    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/hi")
    public String hi(@RequestParam(value="name", defaultValue="Artaban") String name) {
        String greeting = this.restTemplate.getForObject("http://localhost:8090/greeting", String.class);
        return String.format("%s, %s!", greeting, name);
    }

    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class);
    }

}

```

上面代码中，`RestTemplate`指定了服务端的地址是`http://localhost:8090`。所以现在只是演示单服务端实例的情况。

同样，我们也要为客户端配置启动端口：

```
# application.yml
server:
  port: 8888

spring:
  application:
    name: my-client
```

## 测试

现在，我们将服务端和客户端都启动起来，访问`http://localhost:8888/hi`，会得到一个随机的问候语，比如`Greetings, Artaban!`。

由于现在我们客户端已经指定了服务端的地址，所以无论我们启动多少个服务端实例，访问的还是同一个实例。所以，接下来将客户端改造一下，配置多个服务端地址，负载均衡访问。

## 跨服务器实例负载均衡

**首先**，修改客户端`application.yml`配置文件，添加配置Ribbon客户端信息：

```
server:
  port: 8888

spring:
  application:
    name: my-client

my-server:
  ribbon:
    eureka:
      enabled: false
    listOfServers: localhost:8090,localhost:9092,localhost:9999
    ServerListRefreshInterval: 15000
```

这里的`my-server`就是后面会使用到的Ribbon客户端。Spring Cloud Netflix在我们的应用程序中为每个Ribbon客户端名称创建一个`ApplicationContext`，用于客户端自定义一些组件实例。包括：

- `IClientConfig` 为客户端或者负载均衡器保存客户端配置信息
- `ILoadBalancer` 代表一个软件负载均衡器
- `ServerList` 定义了如何获取可供选择的服务器列表
- `IRule` 描述负载均衡策略
- `IPing` 说明如何执行服务器的周期性ping

上面的配置中，`my-server`就是Ribbon客户端的名字。我们设置了`eureka.enabled`，`listOfServers`，`ServerListRefreshInterval`三个属性。Ribbon中的负载均衡器默认通过Netflix Eureka 来发现服务器列表，在我们的例子中没有使用Eureka ，所以就将`ribbon.eureka.enabled`属性设置为`false`。取而代之，我们通过`listOfServers`为Ribbon提供了静态的服务器列表。而`ServerListRefreshInterval`属性则从名字可以看出，就是服务器列表的刷新时间间隔，单位为毫秒。


**然后**，在我们的`ClientApplication`类中，将`RestTemplate`切换到使用Ribbon客户端来获取服务端地址。

```
package com.rason.client;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

/**
 * Created by rason on 5/3/17.
 */
@SpringBootApplication
@RestController
@RibbonClient(name = "my-server", configuration = ServerConfiguration.class)
public class ClientApplication {

    @LoadBalanced
    @Bean
    RestTemplate restTemplate(){
        return new RestTemplate();
    }

    @Autowired
    RestTemplate restTemplate;

    @RequestMapping("/hi")
    public String hi(@RequestParam(value="name", defaultValue="Artaban") String name) {
        String greeting = this.restTemplate.getForObject("http://my-server/greeting", String.class);
        return String.format("%s, %s!", greeting, name);
    }

    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class);
    }

}

```

在`RestTemplate`上使用了`@LoadBalanced`注解，告诉Spring Cloud我们将使用负载均衡。同时，对这个类进行了`@RibbonClient`注解，并指明了客户端的名字和额外的配置类。

所以，我们需要创建这个额外的配置类。

```
package com.rason.client;

import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.AvailabilityFilteringRule;
import com.netflix.loadbalancer.IPing;
import com.netflix.loadbalancer.IRule;
import com.netflix.loadbalancer.PingUrl;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;

/**
 * Created by rason on 5/3/17.
 */
public class ServerConfiguration {

    @Autowired
    IClientConfig ribbonClientConfig;

    @Bean
    public IPing ribbonPing(IClientConfig config) {
        return new PingUrl();
    }

    @Bean
    public IRule ribbonRule(IClientConfig config) {
        return new AvailabilityFilteringRule();
    }

}

```

我们可以通过创建我们自己的具有相同名称的bean来覆盖Spring Cloud Netflix给我们的任何与Ribbon相关的bean。这里我们覆盖了默认负载均衡器使用的`IPing`和`IRule`。默认的`IPing`是一个`NoOpPing`(并没有真正ping服务端实例，而是一直报告服务端是稳定的，没有问题)，默认的`IRule`是一个`ZoneAvoidanceRule`(这避免了亚马逊EC2区域服务器故障最多的地区，因此在本地环境中可能难以尝试)。

我们的`IPing`是一个`PingUrl`，会ping一个URL来检测每个服务端是否正常，所以我们的服务端中有个匹配`/`路径的方法，就是用于ping。而`IRule`使用的是`AvailabilityFilteringRule`，这将会使用Ribbon内置的断路由功能来过滤掉“断开”的服务器。如果ping一个服务器连接失败或者读取失败，Ribbon会认为这个服务器挂掉了，直到它能够正常返回才会对其进行负载均衡访问。

## 测试

分别使用以下命令启动三个服务端实例：

```
$ mvn spring-boot:run

$ SERVER_PORT=9092 mvn spring-boot:run

$ SERVER_PORT=9999 mvn spring-boot:run

```

然后启动客户端实例：

```
$ mvn spring-boot:run
```

访问`http://localhost:8888/hi`，然后观察服务端实例，你会发现三个服务端实例都有收到Ribbon的ping信息。

```
2017-05-03 16:49:05.290  INFO 6126 --- [nio-9092-exec-4] com.rason.server.ServerApplication       : Access /
2017-05-03 16:49:28.963  INFO 6126 --- [io-9092-exec-10] com.rason.server.ServerApplication       : Access /

```

再刷新浏览器，多次访问`http://localhost:8888/hi`，你会发现请求将会轮流被三个服务端实例的其中一个处理。

```
2017-05-03 16:51:32.572  INFO 5998 --- [nio-8090-exec-6] com.rason.server.ServerApplication       : Access /greeting

```

如果你关闭一个服务端实例，一旦Ribbon已经ping到它挂了，你会发现请求将会在剩下两个实例进行负载均衡处理。

## 总结

现在，我们已经学会了使用Ribbon进行客户端的负载均衡处理。
