---
title: Spring Cloud Microservice
date: 2017-04-20 10:16:23
categories: SpringCloud
tags: SpringCloud
description: SpringCloud Microservice
---

上一篇博客已经将`Eureka`和`ConfigServe`搭建起来了，今天将开发两个简单的微服务，一个`pong`服务和一个调用`pong`服务的`ping`服务。

![模型](/image/Sample-ping-pong.png)

## Sample-Pong 微服务

`pong`服务接受`ping`服务的请求，是一个典型的Spring MVC端点：

```
package com.rason.pong.controller;

import com.rason.pong.domain.Message;
import com.rason.pong.domain.MessageAcknowledgement;
import com.rason.pong.service.MessageHandlerService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.hateoas.Resource;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

/**
 * Created by rason on 4/19/17.
 */
@RestController
public class PongController {

    @Autowired
    private MessageHandlerService messageHandlerService;

    @RequestMapping(value = "/message", method = RequestMethod.POST)
    public Resource<MessageAcknowledgement> pongMessage(@RequestBody Message input) {
        return new Resource<>(this.messageHandlerService.handleMessage(input));
    }
}

```

它接受一个消息然后响应一个确认信息。消息处理只是简单地返回一个确认信息，如下所示。

<!-- more -->

```
package com.rason.pong.service.impl;

import com.rason.pong.domain.Message;
import com.rason.pong.domain.MessageAcknowledgement;
import com.rason.pong.service.MessageHandlerService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

/**
 * Created by rason on 4/19/17.
 */
@Service
public class MessageHandlerServiceImpl implements MessageHandlerService {

    private static final Logger logger = LoggerFactory.getLogger(MessageHandlerServiceImpl.class);

    @Value("${reply.message}")
    private String replyMessage;

    @Override
    public MessageAcknowledgement handleMessage(Message message) {
        logger.info("About to Acknowledge");

        return new MessageAcknowledgement(message.getId(), message.getPayload(), this.replyMessage);
    }

}

```

重点关注的是如何通过配置中心注入`reply.message`属性。

`pong`服务如何找到配置中心？有两种潜在的方式：

- 直接指定配置中心服务的位置
- 通过Eureka找到配置中心服务

这里我们采用第二种方法，使用Spring Cloud很容易实现，只需要简单的配置：

```
# bootstrap.yml
spring:
  application:
    name: sample-pong
  cloud:
    config:
      enabled: true
      discovery:
        enabled: true
        serviceId: SAMPLE-CONFIG
```

通过`spring.cloud.config.discovery.enabled`属性开启通过Eureka来发现配置中心服务，并且通过`spring.cloud.config.discovery.serviceId`制定配置中心服务的ID，就是我们上一篇博客中注册的配置服务。

现在，我们需要将`pong`服务跑起来，使用Spring Boot只需要几行代码：

```
package com.rason.pong;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

/**
 * Created by rason on 4/19/17.
 */
@SpringBootApplication
@EnableDiscoveryClient
public class PongApplication {

    public static void main(String[] args) {
        SpringApplication.run(PongApplication.class, args);
    }

}

```

## Sample-ping 微服务

接下来就是`ping`服务，`ping`服务是`pong`服务的消费者。本质上`ping`服务和`pong`服务是一样的，本例子中`ping`服务需要调用`pong`服务，所以唯一不同的是需要用到`RestTemplate`来访问`pong`的REST服务。

`ping`服务的入口端点也是也是REST服务，如下：

```
package com.rason.ping.controller;

import com.rason.ping.domain.Message;
import com.rason.ping.domain.MessageAcknowledgement;
import com.rason.ping.service.PingService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class PingController {

    @Autowired
    private PingService pingService;

    @Value("${send.message}")
    private String sendMessage;

    @RequestMapping("/send")
    public MessageAcknowledgement sendMessage() {
        Message message = new Message();
        message.setId("1");
        message.setPayload(sendMessage);
        return this.pingService.sendMessage(message);
    }
}

```

和`ping`服务类似，这里也会通过配置中心服务注入发送信息`send.message`。因此，我们也需要配置通过Eureka来发现配置中心服务加载属性文件。

```
# bootstrap.yml
spring:
  application:
    name: sample-ping
  cloud:
    config:
      enabled: true
      discovery:
        enabled: true
        serviceId: SAMPLE-CONFIG
```

接下来就是通过`RestTemplate`来访问`pong`的REST服务，如下：

```
package com.rason.ping.service.impl;

import com.google.common.collect.Maps;
import com.rason.ping.domain.Message;
import com.rason.ping.domain.MessageAcknowledgement;
import com.rason.ping.service.PingService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

/**
 * Created by rason on 4/19/17.
 */
@Service
public class PingServiceImpl implements PingService {

    private static final Logger LOGGER = LoggerFactory.getLogger(PingServiceImpl.class);

    @Autowired
    @LoadBalanced
    private RestTemplate restTemplate;

    @Override
    public MessageAcknowledgement sendMessage(Message message) {
        HttpEntity<Message> requestEntity = new HttpEntity<>(message);
        ResponseEntity<MessageAcknowledgement> response =  this.restTemplate.exchange("http://sample-pong/message", HttpMethod.POST, requestEntity, MessageAcknowledgement.class, Maps.newHashMap());
        return response.getBody();
    }

}

```

最后，就是将`ping`服务跑起来注册到Eureka，通过SpingBoot是一件很简单的事情。

```
package com.rason.ping;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
@EnableDiscoveryClient
public class PingApplication {

    public static void main(String[] args) {
        SpringApplication.run(PingApplication.class, args);
    }

    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

```

## 将整个应用运行起来

- 1.先将Eureka跑起来，运行EurekaApplication.java
- 2.再将配置中心服务跑起来，运行ConfigServerApplication.java
- 3.最后运行两个微服务，PingApplication.java,PongApplication

都跑起来之后在Eureka界面可以看到已经注册的服务，如下图：

![完整服务](/image/eureka.png)

然后在浏览器中访问`http://localhost:8080/send`，即通过ping服务发送消息到pong服务，返回结果如下：

```
{"id":"1","received":"Ping","payload":"Pong From Configuration Server"}
```

pong服务收到的发送信息是“Ping”，响应信息是“Pong From Configuration Server”，这正是我们在`sample-config`工程中`sample-ping.yml`和`sample-pong.yml`文件配置的内容。

## 总结

这两篇教程的核心内容就是通过Eureka作为服务的注册和发现中心，ConfigServer作为中心化配置的地方，两个简单的微服务通过从ConfigServer加载属性信息并进行调用。工程的完整代码可以在[我的GitHub](https://github.com/rason/spring-cloud-sample)下载。
