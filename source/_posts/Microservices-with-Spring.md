---
title: Microservices with Spring
date: 2017-04-11 13:18:32
categories: SpringCloud
tags: SpringCloud
description: Microservices with Spring
---

## 简介

本文将介绍使用Spring, Spring Boot 和Spring Cloud来搭建一个简单的微服务系统。

微服务可以将一个大型的系统分成一系列互相协作的部分。它在进程级别完成和Spring 在组件级别完成的事情：松耦合进程而不是松耦合组件。

比如一个在线购物系统，使用微服务可以将其分成用户账户(user-accounts)，商品管理(product-catalog)，订单处理(order-processing)和购物车(shopping-carts)。如下图所示：

![在线购物微服务系统](/image/shopping-system.jpg)

为了简单起见，本文只实现这个庞大系统的一部分——用户账户。

Web-Application 使用RESTful API 请求Account-Service 微服务。我们还需要添加*发现服务*——使得其他进程可以找到对方。如下图所示：

![用户账户微服务](/image/mini-system.jpg)

## 服务注册

当多个进程需要互相协作时，必须需要知道对方的存在。如果您曾经使用Java的RMI机制，您可能会记得它依赖于一个中央注册表，以便RMI进程可以互相发现。微服务其实也差不多。

`Netflix` 公司的开发者在开发系统他们自己的系统过程中也遇到过同样的问题，所以他们创造了一个服务注册系统——Eureka。幸运的是，他们开源了这个系统并且Spring 将其并入了Spring Cloud项目。所以我们可以很方便地运行一个Eureka服务。下面就是一个完整的*发现服务*应用。

```
@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistrationServer {
    public static void main(String[] args) {
        // 指定Spring Boot查找的配置文件
        System.setProperty("spring.config.name", "registration-server");
        SpringApplication.run(ServiceRegistrationServer.class, args);
    }
}
```

是不是非常简单？

<!-- more -->

Spring Cloud 是建立在 Spring Boot 基础之上的，利用到了一些parent 和 starter 的依赖，本工程的依赖文件如下：

```
<dependencies>
    <dependency>
        <!-- Setup Spring Boot -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <dependency>
        <!-- Setup Spring MVC & REST, use Embedded Tomcat -->
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <!-- Spring Cloud starter -->
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter</artifactId>
    </dependency>

    <dependency>
        <!-- Eureka for service registration -->
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
</dependencies>
```

Spring Boot 应用默认是查找`application.properties` 或者 `application.yml` 读取配置信息。通过设置`spring.config.name` 属性我们可以指定查找的文件。如果你在一个工程中有多个Spring Boot 应用的话就很有用了。下面我们就会看到。

上面的*发现服务*应用将会查找`registration-server.properties` 或者 `registration-server.yml`。下面是`registration-server.yml`文件的配置内容：

```
server:
  port: 1111

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

Eureka的默认端口是8761，这里我们将其修改为1111。通过注册代码可以注册为服务端或者客户端，这里我们指定不是客户端。

运行`ServiceRegistrationServer`，你就可以通过`http://localhost:1111`进入Eureka控制台，观察到Applications部分应该是空的，因为目前还没有服务注册上去。

## 创建一个微服务：Account-Service

一个微服务是一个处理明确请求的独立进程。

使用Spring配置应用程序时，我们强调松耦合和紧密衔接。现在我们将在**进程**互相协作上运用这一概念，而不是在相互作用的**组件**(Spring Beans)上。

![松耦合和紧密衔接](/image/beans-vs-processes.jpg)

在本例中，我们有一个简单的账户管理微服务，使用Spring Data 实现一个JPA `AccountRepository` 和使用Spring REST 提供RESTful 接口来获取账户信息。在大多数情况下这就是一个简单的Spring Boot应用。

不同的是在启动的时候我们将其注册到Eureka。下面是这个Spring Boot 的启动类。

```
@EnableAutoConfiguration
@EnableDiscoveryClient
@Import(AccountConfiguration.class)
public class AccountServer {
    @Autowired
    protected AccountRepository accountRepository;

    protected Logger logger = Logger.getLogger(AccountServer.class.getName());

    public static void main(String[] args) {
        // Will configure using account-server.yml
        System.setProperty("spring.config.name", "account-server");

        SpringApplication.run(AccountServer.class, args);
    }
}

```

注解解释：

- `@EnableAutoConfiguration` 定义这是一个Spring Boot 应用。

- `@EnableDiscoveryClient` 开启服务注册和发现。在这里将自身注册到Eureka。

- `@Import(AccountConfiguration.class)` 这个配置类建立一些其它信息。

注册配置文件`account-server.yml`

```
# Spring properties
spring:
  application:
    name: accounts-service
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: none
  h2:
    console:
      enabled: true


# Discovery Server Access
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:1111/eureka/

# HTTP Server
server:
  port: 2222   # HTTP (Tomcat) port
```

重点关注这个文件的几点内容：

1. 将应用的名字设置为`accounts-service`。服务以这个名字注册，也可以通过此名字访问。

2. 指定自定义端口号为2222。我们所有的进程都使用Tomcat，不能都使用8080。

3. Eureka服务的URL。

运行`AccountServer`应用，刷新`http://localhost:1111`控制台，然后我们可以看见ACCOUNTS-SERVICE在应用列表中。注册时间长达30秒（默认），所以需要耐心等待，可以检查一些`ServiceRegistrationServer`的日志输入。

通过`http://localhost:1111/eureka/apps/`可以看到更多注册服务的信息。也可以通过`http://localhost:1111/eureka/apps/ACCOUNTS-SERVICE`查看我们刚注册的`ACCOUNTS-SERVICE`，如果返回404证明没有注册成功。

## 访问微服务：Web-Service

为了访问RESTful服务，Spring 提供了`RestTemplate`类。允许我们发送HTTP请求到RESTful 服务器并以一定的格式获取数据——比如JSON和XML。

使用哪种格式取决于我们在类路径中使用了哪些转换类的依赖。比如通常会检测到JAXB，因为它是Java标准的一部分。如果类路径中有Jackson则会使用JSON格式。

下面是客户端应用`WebAccountService`：

```
@Service
public class WebAccountService {

    @Autowired
    @LoadBalanced
    private RestTemplate restTemplate;

    private String serviceUrl;

    public WebAccountService(String serviceUrl) {
        this.serviceUrl = serviceUrl;
    }

    public AccountDTO getByNumber(String accountNumber) {
        AccountDTO accountDTO = restTemplate.getForObject(serviceUrl + "/accounts/{number}", AccountDTO.class, accountNumber);

        if (accountDTO == null)
            throw new AccountNotFoundException(accountNumber);
        else
            return accountDTO;
    }
}
```

WebAccountService只是一个封装类，使用RestTemplate从微服务中获取数据。最重要的两部分就是`serviceUrl` 和 `RestTemplate`。

下面是客户端的入口程序：

```
@SpringBootApplication
@EnableDiscoveryClient
@ComponentScan(useDefaultFilters = false) //Disable component scan
public class WebServer {

    public static final String ACCOUNTS_SERVICE_URL = "http://accounts-service";

    public static void main(String[] args) {
        // Will configure using web-server.yml
        System.setProperty("spring.config.name", "web-server");
        SpringApplication.run(WebServer.class, args);
    }

    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean
    public WebAccountService accountsService() {
        return new WebAccountService(ACCOUNTS_SERVICE_URL);
    }

    @Bean
    public WebAccountController accountsController() {
        return new WebAccountController(accountsService());
    }
}
```

关注点：

1. 这里的`WebAccountController`就是一个典型的Spring MVC 基于视图的控制器，返回HTML。这个客户端应用将使用Thymeleaf作为视图技术。

2. `WebServer`也被标注`@EnableDiscoveryClient`，将其注册到Eureka中。但现在它没有为其它提供服务，所以不要也可以。

3. 默认的component-scanner会自动扫描`@Component`的类，这里应该要查找`WebAccountController`并创建它。但是我们想自己手动来创建，所以就使用`@ComponentScan(useDefaultFilters=false)`关闭自动扫描功能。

4. 传到`WebAccountController`的service-url就是我们需要访问的微服务的名字，上面的账户微服务在`account-service.yml`文件中配置的名字是`account-service`。使用大些的方式不是必要的，因为这是不是大小写敏感的，但是使用大些可以强调*ACCOUNTS-SERVICE*是一个逻辑主机（通过发现来获取）而不是一个实际的物理主机。

## RestTemplate 负载均衡

`RestTemplate` 将被Spring Cloud拦截并自动配置（由于@LoadBalanced注释）使用自定义HttpRequestClient，使用Netflix [Ribbon](http://techblog.netflix.com/2013/01/announcing-ribbon-tying-netflix-mid.html)进行微服务查找。Ribbon 同样也是负载均衡的，所以如果你有多个实例，会挑选一个进行服务。（Eureka 和 Consul 都不负责负载均衡，所以我们使用Ribbon来代替）。

**注意**： 从Brixton版本（Spring Cloud 1.1.0.RELEASE）开始，RestTemplate 不再会自动创建。之前的会自动创建，这样会造成混乱和潜在的冲突。

如果你进入[ RibbonClientHttpRequestFactory ](https://github.com/spring-cloud/spring-cloud-netflix/blob/master/spring-cloud-netflix-core/src/main/java/org/springframework/cloud/netflix/ribbon/RibbonClientHttpRequestFactory.java)类，你会发现这样的代码：

```
String serviceId = originalUri.getHost();
ServiceInstance instance =
         loadBalancer.choose(serviceId);  // loadBalancer uses Ribbon
... if instance non-null (service exists) ...
URI uri = loadBalancer.reconstructURI(instance, originalUri);
```

`loadBalancer` 将逻辑服务名字转换成被选择微服务的真实主机。

`RestTemplate` 实例是线程安全的，可以被多个服务同时使用。

## web客户端配置文件web-server.yml

```
# Spring Properties
spring:
  application:
     name: web-service
  freemarker:
     enabled: false # Ignore Eureka dashboard FreeMarker templates
  thymeleaf:
     cache: false # Allow Thymeleaf templates to be reloaded at runtime
     prefix: classpath:/web-server/templates/

# Discovery Server Access
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:1111/eureka/

# HTTP Server
server:
  port: 3333   # HTTP (Tomcat) port
```

## 整体运行

1. 先运行`ServiceRegistrationServer`将Eureka运行起来。

2. 再运行`AccountServer`将微服务accounts-service注册到Eureka。

3. 最后web应用`WebServer`作为客户端访问微服务accounts-service。

4. 在浏览器中输入`http://localhost:3333/accounts/123456789`即可获取到账户123456789的信息。

