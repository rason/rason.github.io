---
title: Spring Boot 缓存数据
date: 2017-03-02 13:13:02
categories: SpringBoot
tags: SpringBoot 缓存
description: SpringBoot 缓存
---

## 前言

本文将学习在Spring管理的Bean中使用缓存。

### 你将会创建

你将会创建一个简单书籍仓库中使用缓存的应用。

### 你将需要

- 大约15分钟
- 一个熟悉的IDE
- JDK 1.8以上
- MAVEN 3.0以上

## 创建一个书籍仓库

首先，我们来创建一个简单的实体类来代表书籍。

```
package com.rason.cache.service;

/**
 * Created by rason on 2/28/17.
 */
public class Book {

    private String isbn;
    private String title;

    public Book(String isbn, String title) {
        this.isbn = isbn;
        this.title = title;
    }

    public String getIsbn() {
        return isbn;
    }

    public void setIsbn(String isbn) {
        this.isbn = isbn;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    @Override
    public String toString() {
        return "Book{" + "isbn='" + isbn + '\'' + ", title='" + title + '\'' + '}';
    }
}

```

为实体添加仓库：

```
package com.rason.cache.service;

/**
 * Created by rason on 2/28/17.
 */
public interface BookRepository {

    public Book getByIsbn(String isbn);
    
}

```

你可以使用Spring Data来提供仓库的实现，存储在SQL或者NoSQL中。但是这篇教材的重点不是这个，所以我们使用简单的代码来模拟一些潜在因素（比如网络服务，延迟等）。

在getByIsbn方法中我们调用了simulateSlowService方法来模拟耗时业务。在后面的例子中，我们将会用缓存使得获取速度更快。

<!-- more -->

## 使用仓库

下一步，使用仓库来访问一些书籍。

```
package com.rason.cache.service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;


/**
 * Created by rason on 2/28/17.
 */
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}

```

使用CommandLineRunner来注入BookRepository然后使用不同的参数调用多次。

```
package com.rason.cache.service;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

/**
 * Created by rason on 2/28/17.
 */
@Component
public class AppRunner implements CommandLineRunner {

    private static final Logger logger = LoggerFactory.getLogger(AppRunner.class);

    private final BookRepository bookRepository;

    public AppRunner(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }

    @Override
    public void run(String... args) throws Exception {
        logger.info(".... Fetching books");
        logger.info("isbn-1234 -->" + bookRepository.getByIsbn("isbn-1234"));
        logger.info("isbn-4567 -->" + bookRepository.getByIsbn("isbn-4567"));
        logger.info("isbn-1234 -->" + bookRepository.getByIsbn("isbn-1234"));
        logger.info("isbn-4567 -->" + bookRepository.getByIsbn("isbn-4567"));
        logger.info("isbn-1234 -->" + bookRepository.getByIsbn("isbn-1234"));
        logger.info("isbn-1234 -->" + bookRepository.getByIsbn("isbn-1234"));
    }

}
```

如果你运行Application类，你会注意到虽然你多次获取相同的书籍，但是还是非常慢。

```
2017-02-28 15:19:29.833  INFO 27253 --- [           main] com.rason.cache.service.AppRunner        : isbn-1234 -->Book{isbn='isbn-1234', title='Some Book'}
2017-02-28 15:19:32.834  INFO 27253 --- [           main] com.rason.cache.service.AppRunner        : isbn-4567 -->Book{isbn='isbn-4567', title='Some Book'}
2017-02-28 15:19:35.834  INFO 27253 --- [           main] com.rason.cache.service.AppRunner        : isbn-1234 -->Book{isbn='isbn-1234', title='Some Book'}
2017-02-28 15:19:38.835  INFO 27253 --- [           main] com.rason.cache.service.AppRunner        : isbn-4567 -->Book{isbn='isbn-4567', title='Some Book'}
2017-02-28 15:19:41.835  INFO 27253 --- [           main] com.rason.cache.service.AppRunner        : isbn-1234 -->Book{isbn='isbn-1234', title='Some Book'}
2017-02-28 15:19:44.836  INFO 27253 --- [           main] com.rason.cache.service.AppRunner        : isbn-1234 -->Book{isbn='isbn-1234', title='Some Book'}
```

从上面的时间可以看出，获取每本书都需要大概三秒的时间，即使是相同的书籍被重复获取。

## 使用缓存

让我们在`SimpleBookRepository`中开启缓存，使得我们的书籍可以缓存在`books`缓存中。

```
package com.rason.cache.service;

import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Component;

/**
 * Created by rason on 2/28/17.
 */
@Component
public class SimpleBookRepository implements BookRepository {


    @Override
    @Cacheable("books")
    public Book getByIsbn(String isbn) {
        simulateSlowService();
        return new Book(isbn, "Some Book");
    }

    private void simulateSlowService() {
        long time = 3000L;
        try {
            Thread.sleep(time);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

}

```

你现在需要开启缓存注解的处理。

```
package com.rason.cache.service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;


/**
 * Created by rason on 2/28/17.
 */
@SpringBootApplication
@EnableCaching
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

}

```

`@EnableCaching`注解触发一个后置处理器来检测每个Spring Bean中带有缓存注解的公有方法。如果发现有注解，则会创建一个代理类来拦截方法的调用，进行相应的缓存动作。

[Cacheable](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/Cacheable.html),[CachePut](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/CachePut.html),[CacheEvict](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/annotation/CacheEvict.html)这几个注解都会由这个后置处理器来管理。你可以参照[文档](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/cache.html)获取更多的细节信息。

Spring Boot会自动配置一个合适的[CacheManager](http://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/cache/CacheManager.html)。更多信息可以见[这里](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-caching.html)。

我们的例子中没有使用特别的缓存库，使用默认的ConcurrentHashMap进行简单的缓存。市面上有很多缓存库可以供我们使用。

## 测试应用

现在已经开启了缓存，我们再次运行Application，发现有很大的不同。

```
2017-02-28 15:28:47.797  INFO 27537 --- [           main] com.rason.cache.service.AppRunner        : isbn-1234 -->Book{isbn='isbn-1234', title='Some Book'}
2017-02-28 15:28:50.798  INFO 27537 --- [           main] com.rason.cache.service.AppRunner        : isbn-4567 -->Book{isbn='isbn-4567', title='Some Book'}
2017-02-28 15:28:50.801  INFO 27537 --- [           main] com.rason.cache.service.AppRunner        : isbn-1234 -->Book{isbn='isbn-1234', title='Some Book'}
2017-02-28 15:28:50.801  INFO 27537 --- [           main] com.rason.cache.service.AppRunner        : isbn-4567 -->Book{isbn='isbn-4567', title='Some Book'}
2017-02-28 15:28:50.801  INFO 27537 --- [           main] com.rason.cache.service.AppRunner        : isbn-1234 -->Book{isbn='isbn-1234', title='Some Book'}
2017-02-28 15:28:50.801  INFO 27537 --- [           main] com.rason.cache.service.AppRunner        : isbn-1234 -->Book{isbn='isbn-1234', title='Some Book'}
```

第一次获取书籍时需要三秒左右，之后获取相同的书籍就非常快了。

## 总结

现在，我们已经学会来在Spring管理的Bean中开启缓存了。本文完整代码可见[这里](https://github.com/spring-guides/gs-caching)。

