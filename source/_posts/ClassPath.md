title: ClassPath
categories: Java
date: 2016-05-16 11:15:26
tags: ClassPath
description: ClassPath JavaClassPath
---

## 前言

基础不扎实，就如搭建空中楼阁。这个浮躁的世界，我们总想着很快地实现自己的目标。然而这个世界并不是那么容易就取得成功，冯仑也曾说过“伟大是熬出来的”。编程也是一样的道理，学习基础知识可能很枯燥无味，而且很费时间，可是为了走得更远我们又不得不这样做。明白成功并不那么容易，戒掉浮躁，就是一种伟大的进步。或许，我们**把目标实现的时间设置得更长，比如五年、十年，浮躁就会离我们远一点**。

扯得有点远了，假如我问一个问题：**不利用任何IDE，你能编写一个Java程序在命令行中输入伟大的‘Hello World’吗？**如果可以，那么阅读到这里就可以去陪女朋友或老婆了。

## 使用javac,java编译执行java文件

**第一步：**在Eclipse中创建一个Java工程，并在包com.rason.algs4.binarysearch下创建一个Test.java的文件

```
package com.rason.algs4.binarysearch;

public class Test {

	public static void main(String[] args) {
		System.out.println("Hello world!Hello " + args[0]);
	}

}
```

前面提出的问题是不用IDE，为什么我用IDE创建？这里我只是为了方便后面演示问题。

**第二步：**打开Test.java的存放路径，在我的机器中是`~/Documents/workspace/algs4/src/com/rason/algs4/binarysearch`,然后使用`javac Test.java`编译

`rason@rason-ThinkPad-T430:~/Documents/workspace/algs4/src/com/rason/algs4/binarysearch$ javac Test.java` 

编译之后会在该文件夹下生成`Test.class`文件。

**第二步：**使用java命令执行Test.class

执行`rason@rason-ThinkPad-T430:~/Documents/workspace/algs4/src/com/rason/algs4/binarysearch$ java Test rason`

最后的`rason`字符串作为第一个参数传给Test的main函数。

按回车之后，发现错误了`Error: Could not find or load main class Test`。这是什么原因？从字面的意思我们可以看出，是因为找不到或者加载不到Test这个class文件。然后想了一下，是不是因为我们没写包名？加上包名再试一次。

执行`rason@rason-ThinkPad-T430:~/Documents/workspace/algs4/src/com/rason/algs4/binarysearch$ java com.rason.algs4.binarysearch.Test rason`

发现还是错误`Error: Could not find or load main class com.rason.algs4.binarysearch.Test`，意思还是找不到，此时你应该意识到自己并不熟悉使用java命令执行class文件了吧？

<!-- more -->

## java命令的执行顺序

1. 查找class
2. 加载class
3. 检查class是否具有main方法
4. 调用main方法并把命令行中的参数作为一个String数据传递过去

## 使用java命令的正确姿势

从上面的错误得知，是因为根本就没有找到Test.class文件，我们就应该知道是因为在当前的`classpath`中找不到Test.class文件。

java命令的完整语法如下：

```
java [ <option> ... ] <class-name> [<argument> ...]
```
其中，option选项可以让我们指定当前的一些运行环境，这就包括`-classpath`选项。完整的option选项可以[参考这里](http://docs.oracle.com/javase/7/docs/technotes/tools/windows/java.html)。

`class-name`当然就是指需要运行的class的名字，包含包名的完整名字。

`argument`就是传递给main函数的参数，多个参数之间可以用空格隔开。

那么，我们如何正确地使用`java -classpath`指定类的加载路径呢？

- **对于包含.class文件的.jar或者.zip文件，指定的classpath路径就是以.jar或者.zip的文件名结束**
- **对于没有包名的.class文件，指定的classpath路径就是包含.class文件的那个文件夹**
- **对于具有包名的.class文件，指定的classpath路径就是包含包的根目录的那个文件夹**

默认不指定的话用户自定义类的classpath就是当前文件夹。

因此，从上面可以得知，我们直接在`~/Documents/workspace/algs4/src/com/rason/algs4/binarysearch`下执行`java com.rason.algs4.binarysearch.Test rason`,并没有使用-classpath指定类的查找路径，那么默认就是当前路径了。

然而，具有包名的.class文件的classpath正确的路径应该是包含包的根目录的那个文件夹，也就是`~/Documents/workspace/algs4/src`。因此，我们可以回到`~/Documents/workspace/algs4/src`文件夹中执行`java com.rason.algs4.binarysearch.Test rason`。

```
rason@rason-ThinkPad-T430:~/Documents/workspace/algs4/src$ java com.rason.algs4.binarysearch.Test rason
Hello world!Hello rason

```
终于正确了！

当然，如果我们不回到`~/Documents/workspace/algs4/src`文件夹中，直接使用`java -classpath ~/Documents/workspace/algs4/src/ com.rason.algs4.binarysearch.Test rason`指定正确的classpath也是可以正确执行的。

## ClassPath通配符

classpath可以使用*通配符，情况如下：

- foo/*,指定的classpath是foo目录下的所有jar文件，注意不包含foo目录下的class文件。
- foo;foo/*,这样就可以指定foo目录下的所有jar文件和class文件了。

此处可以看出，多个classpath可以用分号`；`隔开。

另外，多个classpath是有先后顺序的，如果再前面的已经查到到了相应的class文件，就不再继续向后查找了。

## 总结

本文主要是介绍了java命令中如何指定classpath的问题，我们在日常开发中经常遇到的类找不到的异常就可以通过本文延伸思考得出解决方案。
