title: ClassLoader工作机制（一）
categories: Java
date: 2016-02-25 09:14:24
tags: ClassLoader
description: ClassLoader工作机制
---

## 前言

ClassLoader这个名字应该不算陌生，虽然我们经常会见到这个名字，但通常都简单地去了解它的工作机制。ClassLoader顾名思义就是类加载器，其作用包括：

- 将Class加载到JVM中
- 审查每个类应该由谁加载，父优先委托加载机制
- 将Class字节码重新解析成JVM统一要求的对象格式

这篇文章主要分析前面两点。

## ClassLoader类结构

ClassLoader是一个抽象类，在扩展ClassLoader的时候我们通常会用到以下几个方法：

- defineClass(byte[],int,int)
- findClass(String)
- loadClass(String)
- resolveClass(Class<?>)

defineClass方法用来将字节流解析成JVM能够识别的Class对象，可以从class文件或者网络的方式得到一个类的字节流，从而创建类的Class对象形式的实例化对象。

findClass顾名思义就是根据类名字去找到相应的class文件，找到文件读取字节流再调用defineClass就可以得到类的类对象实例。该方法是的默认实现是抛出一个ClassNotFoundException，ClassLoader的子类应该重载该方法并且要遵循委托加载模式来加载class。我们要自定义类的加载规则通常鼓励重写findClass方法而不是下面的loadClass方法。

loadClass是根据特定的类名字去加载class，它与findClass方法有所不同。findClass是根据名字直接查找相应的资源，而loadClass会先查找是否已经加载，没有的话会根据父优先委托加载机制来加载。

<!--more-->

resolveClass方法用于`link`一个特定的class。关于什么是link，下面说谈到。

如果你不想重新定义加载类的规则，也没有复杂的处理逻辑，只想在运行时能够加载自己指定的一个类而已，那么你可以用`this.getClass().getClassLoader().loadClass("class")`即可。如果要实现自己的ClassLoader,一般都会继承URLClassLoader这个子类，因为这个类已经帮我们实现了大部分工作，我们只需要在适当的地方做些修改好了。

## ClassLoader的委托加载机制

什么是委托加载机制，我们看一下ClassLoader的`loadClass()`方法的实现就会一目了然。

```
    protected synchronized Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        // First, check if the class has already been loaded
        Class c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClass0(name);
                }
            } catch (ClassNotFoundException e) {
                // If still not found, then invoke findClass in order
                // to find the class.
                c = findClass(name);
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }

```
从上面的方法实现我们可以看出，加载class会遵循以下顺序：

- **先调用findLoadedClass，检查这个class是否已经被加载过**
- **调用父加载器的loadClass方法，如果父加载器为null的话，则会调用JVM内部的类加载器加载，即BootstrapClassLoader**
- **如果都没有加载成功，那就自己调用findClass方法来加载**

为了更清楚地理解loadClass和findClass的区别，这里把子类URLClassLoader的findClass方法实现也贴出来：

```
    protected Class<?> findClass(final String name)
         throws ClassNotFoundException
    {
        try {
            return (Class)
                AccessController.doPrivileged(new PrivilegedExceptionAction() {
                    public Object run() throws ClassNotFoundException {
                        String path = name.replace('.', '/').concat(".class");
                        Resource res = ucp.getResource(path, false);
                        if (res != null) {
                            try {
                                return defineClass(name, res);
                            } catch (IOException e) {
                                throw new ClassNotFoundException(name, e);
                            }
                        } else {
                            throw new ClassNotFoundException(name);
                        }
                    }
                }, acc);
        } catch (java.security.PrivilegedActionException pae) {
            throw (ClassNotFoundException) pae.getException();
        }
    }

```

findClass方法就是直接去查找资源，然后调用defineClass解析成JVM能够识别的Class对象。

相信看完这两段代码，大家都应该大概清楚类加载的流程，但是还会有一些疑问：父加载器是哪个？JVM中有哪些类加载器？

在JVM中提供了三个ClassLoader类：

1. **Bootstrap ClassLoader**，主要加载JVM自身工作需要的类，这个ClassLoader完全是JVM自己控制的，需要加载哪个类，怎么加载都由JVM控制，别人也访问不到这个类。所以这个ClassLoader是不遵循前面介绍的加载规则，仅仅是一个类加载工具而已，没有父加载器，也没有子加载器。

2. **ExtClassLoader**，这个类加载器有点特殊，用于加载`System.getProperty("java.ext.dirs")`目录下的class。

3. **AppClassLoader**，它的父类是ExtClassLoader，所有在`System.getProperty("java.class.path")`目录下的类都可以被这个类加载器加载，这个目录就是我们经常用到的classpath。

如果我们要实现自己的类加载器，不管是直接实现抽象类ClassLoader，还是继承URLClassLoader，或是其它子类，它的父加载器都是AppClassLoader。因为不管调用哪个父类的构造器，都是调用`getSystemClassLoader()`作为父加载器，这个方法获取到的正是AppClassLoader。比如ClassLoader的构造方法：

```
 protected ClassLoader() {
	this(checkCreateClassLoader(), getSystemClassLoader());
 }
```

因此，结合ClassLoader的loadClass方法和上面的委托关系可知，**加载一个类时，首先BootStrap进行寻找，找不到再由Extension ClassLoader寻找，最后才是App ClassLoader。**

将ClassLoader设计成委托模型的一个重要原因是出于安全考虑，比如在Applet中，如果编写了一个java.lang.String类并具有破坏性。假如不采用这种委托机制，就会将这个具有破坏性的String加载到了用户机器上，导致破坏用户安全。但采用这种委托机制则不会出现这种情况。因为要加载java.lang.String类时，系统最终会由Bootstrap进行加载，这个具有破坏性的String永远没有机会加载

通常一个应用中类加载器的委托结构如下图所示：

![应用中类加载器的委托层次](/image/web-ClassLoaderLevel.png)

其中App1ClassLoader和App2ClassLoader是应用中自定义的类加载器。有些文章把Bootstrap ClassLoader列在ExtClassLoader的上一级中，其实Bootstrap ClassLoader并不属于JVM的类委托层次，因为Bootstrap ClassLoader并没有遵循ClassLoader的加载规则。另外它并没有子类，ExtClassLoader的父类也不是Bootstrap ClassLoader，ExtClassLoader并没有父类，我们应用中能取到的顶层父类是ExtClassLoader。

如果java应用中没有定义其他ClassLoader，那么除了System.getProperty("java.ext.dirs")目录下的类是由ExtClassLoader加载外，其他类都由AppClassLoader来加载。

JVM加载class文件到内存有两种方式。

- 隐式加载：就是不通过代码里调用ClassLoader来加载需要的类，而是通过JVM来自动加载需要的类到内存中。例如，当我们在类中继承或者引用某个类时，JVM在解析当前这个类时发现引用的类不在内存中，那么就是自动将这些类加载到内存中。Java装载类使用“全盘负责委托机制”，即一个ClassLoader装载一个类时，除非显式使用另外一个ClassLoader，否则该类所依赖及引用的类也由这个ClassLoader载入。

- 显示加载：就是在代码中通过调用ClassLoader类来加载一个类的方式。例如，调用this.getClass.getClassLoader.loadClass()或者Class.forName()，或者我们自己实现的ClassLoader的findClass()方法等。

## 如何加载class文件

ClassLoader加载一个class文件到JVM时需要经过以下步骤：

![JVM加载类的阶段](/image/web-JVM-load.png)

- **第一阶段：找到.class文件并把这个文件包含的字节码加载到内存中。**
- **第二阶段：分为三个步骤，分别是字节码验证，Class类数据结构分析及相应的内存分配和最后的符号表的链接。**
- **第三阶段：类中静态属性和初始化赋值，以及静态块的执行等。**

### 加载字节码到内存

在抽象类ClassLoader中并没有定义如何去加载，如何去找到指定类并且将它的字节码加载到内存需要子类实现，也就是要实现findClass()方法。

我们看一下子类URLClassLoader是如何实现findClass()的，在URLClassLoader中通过一个URLClassPath类帮助取得要加载的class文件字节流，这个URLClassPath定义了**到哪里**去找这个class文件,如果找到了这个class文件，再读取它的byte字节流通过调用defineClass()方法来创建类对象。

URLClassLoader的构造函数必须要指定一个URL数据，也就是必须要指定这个ClassLoader默认到哪个目录下去查找class文件。

```
    public URLClassLoader(URL[] urls) {
        super();
        // this is to make the stack depth consistent with 1.1
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkCreateClassLoader();
        }
        ucp = new URLClassPath(urls);
        acc = AccessController.getContext();
    }
```

从URLClassPath的名字中就可以发现它是通过URL的形式来表示ClassPath路径的。在创建URLClassPath对象时会根据传过来的URL数组中的路径来判断是**文件**还是**jar包**,根据路径的不同分别创建FileLoader或者JarLoader，或者使用默认的加载器（注意：此加载器非上面说的ClassLoader，它是实际加载class的加载器，是ClassLoader需要用到的）。当JVM调用findClass时由这几个加载器来将class文件中的字节码加载到内存中。

这里还需要提到的是：**ExtClassLoader和AppClassLoader继承于URLClassLoader**，那么如何设置每个ClassLoader的搜索路径呢？下方表格所示是Bootstrap ClassLoader，ExtClassLoader和AppClassLoader的参数形式。

| ClassLoader类型        | 参数选项                            | 说明  |
| --------------------- |:----------------------------------:| ---------------------------------------------:|
| Bootstrap ClassLoader | -Xbootclasspath:                  | 设置Bootstrap ClassLoader的搜索路径              |
|                       | -Xbootclasspath/a:                | 把路径添加到已存在Bootstrap ClassLoader搜索路径后面 |
|                       | -Xbootclasspath/p:                | 把路径添加到已存在Bootstrap ClassLoader搜索路径    |
| ExtClassLoader        | -Djava.ext.dirs                    | 设置ExtClassLoader的搜索路径                     |
| AppClassLoader        | -Djava.class.path=-cp或者-classpath | 设置AppClassLoader的搜索路径                     |

上面的参数设置中，最常用到的就是设置classpath的环境变量，因为通常都是让java运行指定的程序。如果通过命令执行一个类时出现NoClassDefFoundError错误，那么很可能是没有指定classpath所致，或者指定了classpath但是没有指明包名。

### 验证与解析

- 字节码验证，类加载器对于类的字节码要做许多检测，以确保格式正确，行为正确。
- 类准备，这个阶段准备代表每个类中定义的字段，方法和实现接口所必须的数据结构。
- 解析，在这个阶段类加载器加载所引用的其他所有类。

### 初始化Class对象

类中包含的静态初始化器都被执行，在这一阶段末尾静态字段初始化成默认值。

## 总结

- **ClassLoader的作用**
- **抽象类ClassLoader的一些方法的作用**
- **ClassLoader的委托加载机制**
- **JVM中的几种ClassLoader及其关系**
- **ClassLoader加载class的几个步骤**
- **如何设置ClassLoader的搜索路径**

