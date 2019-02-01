title: 使用SpringBoot来创建应用
categories: SpringBoot
date: 2017-02-17 09:28:15
tags: SpringBoot
description: 使用SpringBoot来创建应用
---

## 前言

本教程使用一个简单的例子来演示SpringBoot是如何帮助你提升应用开发速度。

### 你需要

- 大约15分钟
- 一个IDE
- JDK1.8以上
- MAVEN3.0以上

## 通过SpringBoot你可以做什么

SpringBoot提供一种快速的方式来创建应用。它通过查找类路径和你配置的Bean,合理地假象你缺少什么，然后自动添加进去。使用SpringBoot你可以更多地聚焦到业务特性而不用过多关心基础结构。

比如：

- 使用Spring MVC？有好几个特殊的Bean几乎经常会用到，所以SpringBoot会自动添加他们。一个Spring MVC应用通常需要一个Servlet容器，所以SpringBoot会自动配置一个嵌入式的Tomcat。

- 使用Jetty？如果是这样，你可能不需要Tomcat，而是替换为嵌入式Jetty。SpringBoot可以很轻易帮你做到。

- 使用Thymeleaf？也有一些Bean需要被添加到应用上下文中，SpringBoot也可以帮你自动添加。

> SpringBoot不会生成代码或者编辑你的文件。而是在你启动应用的时候，自动织入一些Bean，然后配置应用到你的应用上下文。

<!-- more -->

## 创建一个简单的Web应用

现在我们来创建一个简单web应用控制器

```
package com.rason.webapplication.controller;

/**
 * Created by rason on 2/16/17.
 */

import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.bind.annotation.RequestMapping;

@RestController
public class HelloController {

    @RequestMapping("/")
    public String index() {
        return "Greetings from Spring Boot!";
    }

}
```

这个类标志为一个`@RestController`，表示它将被Spring MVC用来处理web请求。`@RequestMapping`将映射`//`到`index()`方法。当通过浏览器访问根路径，该方法会返回纯文本。这是因为`@RestController`相当于同时配置了`@Controller`和`@ResponseBody`，这两个注解会令web请求返回数据而不是一个视图。

## 创建一个Application类

```
package com.rason.webapplication;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;

import java.util.Arrays;

/**
 * Created by rason on 2/16/17.
 */
@SpringBootApplication
public class Application {

    public static void main(String[] args){
        SpringApplication.run(Application.class,args);
    }

    @Bean
    public CommandLineRunner commandLineRunner(ApplicationContext context){
        return args ->{
            System.out.println("Let‘s inspect the beans provided by Spring Boot:");
            String[] beanNames = context.getBeanDefinitionNames();
            Arrays.sort(beanNames);
            int count = 1;
            for (String beanName : beanNames) {
                System.out.println((count++) + ":" + beanName);
            }
        };
    }
}

```

`@SpringBootApplication`是一个便捷的注解，它会添加以下内容：

- `@Configuration` 标记这个类为应用上下文的`bean definitions`资源。

- `@EnableAutoConfiguration`告诉SpringBoot启动时根据类路径、Bean和属性文件自动添加一些需要的Bean。

- 一般情况下你会为Spring MVC应用添加`@EnableWebMvc`注解，但是SpringBoot观察到类路径中含有**spring-webmvc**依赖则会自动添加。这标志着这个应用是一个Web应用，然后会激活一些关键的动作比如建立一个`DispatcherServlet`。

- `@ComponentScan`告诉Spring在com.rason.webapplication包中查找其他组件，配置和服务，允许它找到我们的controller。

`main()`方法中使用SpringBoot的`SpringApplication.run()`方法来启动应用。你可能已经注意到了这个应用没有一行XML，也没有web.xml文件。这个Web应用是百分百纯Java代码，我们不需要处理各种工程配置。

上述代码中还含有一个标记为一个`@Bean`的`CommandLineRunner`方法，它会随着应用的启动运行。获取应用创建的所有Bean和由SpringBoot添加的Bean，然后将其打印出来。

## 运行应用

可以直接在IDE中直接将Application类的main()方法跑起来，会看到大概有以下的Bean被打印出来：

```
Let's inspect the beans provided by Spring Boot:
application
beanNameHandlerMapping
defaultServletHandlerMapping
dispatcherServlet
embeddedServletContainerCustomizerBeanPostProcessor
handlerExceptionResolver
helloController
httpRequestHandlerAdapter
messageSource
mvcContentNegotiationManager
mvcConversionService
mvcValidator
org.springframework.boot.autoconfigure.MessageSourceAutoConfiguration
org.springframework.boot.autoconfigure.PropertyPlaceholderAutoConfiguration
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration$DispatcherServletConfiguration
org.springframework.boot.autoconfigure.web.EmbeddedServletContainerAutoConfiguration$EmbeddedTomcat
org.springframework.boot.autoconfigure.web.ServerPropertiesAutoConfiguration
org.springframework.boot.context.embedded.properties.ServerProperties
org.springframework.context.annotation.ConfigurationClassPostProcessor.enhancedConfigurationProcessor
org.springframework.context.annotation.ConfigurationClassPostProcessor.importAwareProcessor
org.springframework.context.annotation.internalAutowiredAnnotationProcessor
org.springframework.context.annotation.internalCommonAnnotationProcessor
org.springframework.context.annotation.internalConfigurationAnnotationProcessor
org.springframework.context.annotation.internalRequiredAnnotationProcessor
org.springframework.web.servlet.config.annotation.DelegatingWebMvcConfiguration
propertySourcesBinder
propertySourcesPlaceholderConfigurer
requestMappingHandlerAdapter
requestMappingHandlerMapping
resourceHandlerMapping
simpleControllerHandlerAdapter
tomcatEmbeddedServletContainerFactory
viewControllerHandlerMapping
```

你会清晰地看到**org.springframework.boot.autoconfigure **的一些Bean，也有一个`tomcatEmbeddedServletContainerFactory`。

在浏览器中输入`http://localhost:8080/`，会返回`Greetings from Spring Boot!`。

## 添加单元测试

使用Spring Test可以很方便地为你的应用添加单元测试，只需要添加一个Maven依赖即可。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

现在我们来写一个简单的测试案例。

```
package com.rason.webapplication.controller;

import static org.hamcrest.Matchers.equalTo;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.content;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;

/**
 * Created by rason on 2/17/17.
 */
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class HelloControllerTest {

    @Autowired
    private MockMvc mvc;

    @Test
    public void getHello() throws Exception {
        mvc.perform(MockMvcRequestBuilders.get("/").accept(MediaType.APPLICATION_JSON_UTF8))
                .andExpect(status().isOk())
                .andExpect(content().string(equalTo("Greetings from Spring Boot!")));
    }
}

```

这个`MockMvc`来自于Spring Test框架，允许我们通过一系列方便的builder类来发起HTTP请求到`DispatcherServlet`然后断言结果。要注意的是`@AutoConfigureMockMvc`需要和`@SpringBootTest`一起使用才能注入一个`MockMvc`实例。

使用`@SpringBootTest`注解会请求建立整个应用上下文，我们也可以使用`@WebMvcTest`注解来替换它，这样只会创建上下文的web层。

除了模拟HTTP请求，我们还可以利用SpringBoot来写一个简单的全流程集成测试。比如：

```
package com.rason.webapplication.controller;

import static org.hamcrest.Matchers.equalTo;
import static org.junit.Assert.assertThat;

import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.context.embedded.LocalServerPort;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;

import java.net.URL;

/**
 * Created by rason on 2/17/17.
 */
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class HelloControllerIT {

    @LocalServerPort
    private int port;

    private URL base;

    @Autowired
    private TestRestTemplate template;

    @Before
    public void setUp() throws Exception {
        this.base = new URL("http://localhost:" + port + "/");
    }

    @Test
    public void getHello() throws Exception {
        ResponseEntity<String> responseEntity = template.getForEntity(base.toString(),String.class);
        assertThat(responseEntity.getBody(),equalTo("Greetings from Spring Boot!"));
    }
}

```

通过配置`webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT`，嵌入式服务器会使用一个随机端口启动。然后实际的端口可以通过`@LocalServerPort`在运行时发现。

## 添加产品级别的服务

如果你为你的业务创建一个网站，你可能需要添加一些管理服务。SpringBoot提供了一些[监控模块](http://docs.spring.io/spring-boot/docs/1.5.1.RELEASE/reference/htmlsingle/#production-ready)，比如健康检查、统计等。

我们也可以很方便地添加这些管理功能，只需要加入Maven依赖即可。

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

运行应用，你会发现多了一些RESTful的端点，这些管理服务都是由SpringBoot来提供的。

这些包括errors, environment, health, beans, info, metrics, trace, configprops, 和 dump。

你可以很容易地检查应用的健康情况，通过在浏览器中输入`localhost:8080/health`即可。

## 查看SpringBoot的starters

我们已经学习了一些SpringBoot的starter，完整的代码可以查看[这里](http://docs.spring.io/spring-boot/docs/1.5.1.RELEASE/reference/htmlsingle/#using-boot-starter)。

## 总结

我们学会了使用SpringBoot来快速地创建一个简单的web应用，还可以开启一些产品级别的管理服务。本文完整代码可以参考[这里](https://github.com/spring-guides/gs-spring-boot)。更加深入的学习可以见[官方文档](http://docs.spring.io/spring-boot/docs/1.5.1.RELEASE/reference/htmlsingle/)。
