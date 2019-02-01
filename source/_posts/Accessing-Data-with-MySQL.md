---
title: Spring Boot 集成 MySQL
date: 2017-02-27 16:00:48
categories: SpringBoot
tags: SpringBoot MySQL
description: Spring Boot 集成 MySQL
---

## 前言

本文将学习在Spring应用中连接MySQL数据库。我们将使用Spring Data JPA 来访问数据库，当然这是很多中选择中的其中一种（比如你也可以使用简单Spring JDBC）。

### 你将会创建

你将会创建一个MySQL数据库，然后创建一个Spring应用来连接到这个数据库。

### 你将需要

- MySQL
- 大约15分钟
- 一个熟悉的IDE
- JDK 1.8以上
- MAVEN 3.0以上

## 创建数据库

在终端使用命令登录MySQL

```
mysql -u root -p

```

创建数据库

```
mysql> create database db_example;

mysql> create user 'testuser'@'localhost' identified by '123456';

mysql> grant all on db_example.* to 'testuser'@'localhost';

```

## 创建application.properties文件

如果没有配置，Spring Boot 会提供你所有默认的东西，比如默认的数据库是`H2`。所以你想使用其他数据库就必须在application.properties定义连接属性。

在资源文件夹中，我们创建一个资源文件`src/main/resources/application.properties`。

```
spring.jpa.hibernate.ddl-auto=create
spring.datasource.url=jdbc:mysql://localhost:3306/db_example
spring.datasource.username=testuser
spring.datasource.password=123456
```

这里的`spring.jpa.hibernate.ddl-auto`可以为none, update, create, create-drop。更多细节可以参考Hibernate 文档。

<!-- more -->

- none 这是MySQL的默认值，不改变数据库的结构。
- update Hibernate根据给出的实体结构改变数据库。
- create 每次都创建数据库，但是关闭时不要drop。
- create-drop 创建数据库然后当SessionFactory关闭时drop。

我们开始时使用create，因为我们还没有创建任何数据库结构。第一次运行之后，我们可以根据程序的需要改为update或者none。当你想改变数据库结构时就可以使用update。

`H2`和其他嵌入式数据库默认值是`create-drop`，但是其他比如MySQL就是`none`。

当你的数据库已经到了产品状态，将其设为`none`并撤回MySQL用户的权限，只是给它SELECT, UPDATE, INSERT, DELETE权限，这是很好的安全实践。

## 创建@Entity模型类

```
package com.rason.mysql.service;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

/**
 * Created by rason on 3/1/17.
 */
@Entity // 这将会告诉Hibernate创建一个数据库表
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Integer id;

    private String name;

    private String email;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }
}

```

Hibernate会自动将这个实体类转换成一个数据库表。

## 创建仓库

```
package com.rason.mysql.service;

import org.springframework.data.repository.CrudRepository;

/**
 * Created by rason on 3/1/17.
 */
// 这将会由Spring自动实现到一个名为userRepository的bean中
public interface UserRepository extends CrudRepository<User, Integer> {

}

```

这是仓库接口，它将会由Spring自动实现到一个名为userRepository的bean中。

## 为你的Spring应用创建一个新的Controller

```
package com.rason.mysql.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;

import java.util.Iterator;

/**
 * Created by rason on 3/1/17.
 */
@Controller
@RequestMapping("/demo")
public class MainController {

    @Autowired
    private UserRepository userRepository;

    @GetMapping(path = "/add") // 只接受GET请求
    public @ResponseBody String addNewUser(@RequestParam String name, @RequestParam String email) {
        User user = new User();
        user.setEmail(email);
        user.setName(name);
        userRepository.save(user);

        return "Saved";
    }

    @GetMapping(path = "/all")
    public @ResponseBody Iterable<User> getAllUsers() {
        return userRepository.findAll();
    }
}

```

> 上面的例子中没有指定使用GET,PUT,POST，因为`@RequestMapping`默认匹配所有类型的HTTP请求动作。你可以使用例如`@RequestMapping(method=GET)`来进行限制。

## 使应用运行起来

```
package com.rason.mysql.service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

/**
 * Created by rason on 3/1/17.
 */
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
    
}

```

将Application跑起来即可。

## 测试应用

在浏览器中输入

```
http://localhost:8080/demo/add?name=rason&email=rason2008@gmail.com
```

可以看到

```
Saved
```

然后，查看已经插入的数据

```
http://localhost:8080/demo/all
```

可以看到

```
[{"id":1,"name":"rason","email":"rason2008@gmail.com"}]
```

## 安全控制

假设我们现在已经到了产品环境，你可能会受到SQL注入攻击。黑客可能会注入`DROP TABLE`或者其他毁灭性的SQL命令。所以，当我们将应用暴露给用户时需要收回一些数据库权限。

```
revoke all on db_example.* from 'testuser'@'localhost';

```

现在Spring应用不能对数据库做任何事情。但是我们不想这样，所以

```
grant select, insert, delete, update on db_example.* to 'testuser'@'localhost';

```

这样的话应用只能修改数据而不能修改数据库结构。

现在，我们修改`src/main/resources/application.properties`文件

```
spring.jpa.hibernate.ddl-auto=none
```

替换了第一次运行时Hibernate根据实体类来创建数据库表的`create`。

当我们想对数据库表进行修改，需要重新授予权限，将`spring.jpa.hibernate.ddl-auto`改为`update`，再重新运行应用。最好的方式还是使用最用的迁移工具，比如 Flyway 或者 Liquibase。

## 总结

现在，我们已经学会了在Spring应用中集成MySQL数据库了。本文完整代码可以参考[这里](https://github.com/spring-guides/gs-accessing-data-mysql);
