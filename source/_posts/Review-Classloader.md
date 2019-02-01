---
title: 温故Classloader
date: 2017-06-07 17:01:47
categories: Java
tags: ClassLoader
description: ClassLoader工作机制
---

## 类加载器

之前写过关于类加载器的文章，今天再次回顾一下。

类加载器的作用就是读取编译好的`.class`文件，并转化成`java.lang.Class`类的一个实例。

类加载器(ClassLoader)的主要三个方法：

- findClass(String name) 直接由本加载器查找类文件并对其进行定义
- loadClass(String name) 具有委托关系的加载，会调用defineClass来对类进行定义
- defineClass(String name, byte[] b, int off, int len) 对类进行定义

需要关注的几点：

### 1.通常自己实现一个ClassLoader的话是通过重写findClass方法，找到`.class`文件然后调用defineClass方法对类进行定义。

比如：

```
class NetworkClassLoader extends ClassLoader {
    String host;
    int port;

    public Class findClass(String name) {
      byte[] b = loadClassData(name);
      return defineClass(name, b, 0, b.length);
    }

    private byte[] loadClassData(String name) {
      // load the class data from the connection
    }
}
```


<!-- more -->

### 2.Java 虚拟机通过加载此类的类加载器和类的全名判定两个 Java 类是否相同，也就是说两者都相同的话才认为是同一个类。

比如：

```
package com.rason.loader;

import com.rason.service.impl.UserServiceImpl;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;

/**
 * Created by rason on 6/7/17.
 */
public class SystemFileClassLoader extends ClassLoader {

    private String rootDir;

    public SystemFileClassLoader(String rootDir) {
        this.rootDir = rootDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classData = getClassData(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            return defineClass(name, classData, 0, classData.length);
        }
    }

    private byte[] getClassData(String name) {
        File file = new File(rootDir + File.separator + name.replace(".", File.separator) + ".class");
        try {
            FileInputStream fis = new FileInputStream(file);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            byte[] buffer = new byte[4096];
            int count = 0;
            while ((count = fis.read(buffer)) != -1) {
                baos.write(buffer, 0, count);
            }
            return baos.toByteArray();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    public static void main(String[] args) {
        SystemFileClassLoader classLoader = new SystemFileClassLoader("/home/rason/Downloads/demo-master/spring-ioc/target/classes");
        try {
            Class<?> clazz = classLoader.findClass("com.rason.service.impl.UserServiceImpl");
            System.out.println(clazz.getClassLoader().toString());
            try {
                // will throw exception
                UserServiceImpl userService = (UserServiceImpl) clazz.newInstance();

            } catch (Exception e) {
                e.printStackTrace();
            }

        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

```

上面是我自己写的一个测试类，其中` UserServiceImpl userService = (UserServiceImpl) clazz.newInstance();`这行代码就会抛出类转换异常：

```
java.lang.ClassCastException: com.rason.service.impl.UserServiceImpl cannot be cast to com.rason.service.impl.UserServiceImpl
	at com.rason.loader.SystemFileClassLoader.main(SystemFileClassLoader.java:58)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:147)
```

因为它们的全名虽然都是`com.rason.service.impl.UserServiceImpl`，但却不是同一个类加载器加载出来的。一个是我自定义的加载器`SystemFileClassLoader`，另一个是系统的加载器`AppClassLoader`。

```
com.rason.loader.SystemFileClassLoader@6e0be858
sun.misc.Launcher$AppClassLoader@2503dbd3
```

### 3.类属于哪个加载器是由defineClass这个方法确定的，哪个类加载器对这个类进行的定义，这个类就属于哪个类加载器。

这里，也有两点需要注意的地方：

- 哪个类加载器调用loadClass方法，并不代表这个类就属于这个类加载器，因为loadClass方法会先委托父加载器对其进行加载。只有在父加载器加载不了的时候自身才会对`.class`文件进行查找，然后对其定义，这个类才属于这个加载器。
- 为了不破坏类加载器的委托关系，所以我们在自定义类加载器的时候最好不好重写loadClass，可以通过重写findClass方法，就正如第一点所说的。


## 线程上下文类加载器

我们经常会见到这样的代码：

```
ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
```

这个就是线程上下文类加载器，通常在SPI(Service Provider Interface)中会用到。比如JDBC,JMS之类的，Java制定了不少J2EE的标准，但是只是定义了其接口和一些辅助类（比如工厂类），实现是由不同的厂商实现。

如果SPI中有工厂方法通过newInstance()方法来建造一个实例，这样就会出现一个问题：SPI 的接口是 Java 核心库的一部分，是由引导类加载器来加载的。SPI 实现的 Java 类一般是由系统类加载器来加载的。引导类加载器是无法找到 SPI 的实现类的，因为它只加载 Java 的核心库。它也不能代理给系统类加载器，因为它是系统类加载器的祖先类加载器。也就是说，类加载器的委托模式无法解决这个问题。

线程上下文类加载器正好解决了这个问题。如果不做任何的设置，Java 应用的线程的上下文类加载器默认就是系统类加载器(AppClassLoader)。在 SPI 接口的代码中使用线程上下文类加载器，就可以成功的加载到 SPI 实现的类。线程上下文类加载器在很多 SPI 的实现中都会用到。

## 网络类加载器

下面将通过一个网络类加载器来说明如何通过类加载器来实现组件的动态更新。即基本的场景是：Java 字节代码（.class）文件存放在服务器上，客户端通过网络的方式获取字节代码并执行。当有版本更新的时候，只需要替换掉服务器上保存的文件即可。通过类加载器可以比较简单的实现这种需求。

类 NetworkClassLoader负责通过网络下载 Java 类字节代码并定义出 Java 类。它的实现与 FileSystemClassLoader类似。在通过 NetworkClassLoader加载了某个版本的类之后，一般有两种做法来使用它。第一种做法是使用 Java 反射 API。另外一种做法是使用接口。需要注意的是，并不能直接在客户端代码中引用从服务器上下载的类，因为客户端代码的类加载器找不到这些类。使用 Java 反射 API 可以直接调用 Java 类的方法。而使用接口的做法则是把接口的类放在客户端中，从服务器上加载实现此接口的不同版本的类。在客户端通过相同的接口来使用这些实现类。

## 类加载器与 Web 容器

对于运行在 Java EE容器中的 Web 应用来说，类加载器的实现方式与一般的 Java 应用有所不同。不同的 Web 容器的实现方式也会有所不同。以 Apache Tomcat 来说，每个 Web 应用都有一个对应的类加载器实例。该类加载器也使用委托模式，所不同的是它是首先尝试去加载某个类，如果找不到再委托给父类加载器。这与一般类加载器的顺序是相反的。这是 Java Servlet 规范中的推荐做法，其目的是使得 Web 应用自己的类的优先级高于 Web 容器提供的类。这种委托模式的一个例外是：Java 核心库的类是不在查找范围之内的。这也是为了保证 Java 核心库的类型安全。

绝大多数情况下，Web 应用的开发人员不需要考虑与类加载器相关的细节。下面给出几条简单的原则：

- 每个 Web 应用自己的 Java 类文件和使用的库的 jar 包，分别放在 WEB-INF/classes和 WEB-INF/lib目录下面。
- 多个应用共享的 Java 类文件和 jar 包，分别放在 Web 容器指定的由所有 Web 应用共享的目录下面。
- 当出现找不到类的错误时，检查当前类的类加载器和当前线程的上下文类加载器是否正确。

## 总结

类加载器是 Java 语言的一个创新。它使得动态安装和更新软件组件成为可能。

学而时习之，温故而知新。
  

