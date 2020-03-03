title: 将程序打包到JAR文件中
categories: Java
date: 2016-05-17 08:39:17
tags: Java
description: 将程序打包到JAR文件中
---

## 前言

上篇文章介绍了在java命令中设置classpath的问题，直接用java命令运行一个class。趁热打铁，这篇文章介绍一下使用java命令运行一个.jar文件。

## JAR文件的好处

- 安全：你可以对JAR文件的内容进行数字签名
- 减少下载次数：不用为JAR文件中的每项内容都发起一个Http请求下载
- 压缩：占用空间少
- 作为扩展包：我们知道java平台有一些扩展的包，JAR文件可以使我们的软件作为扩展包放到java平台上
- 包封装：封装一个JAR文件意味着所以依赖的classes都应该被打包在JAR文件中，作为一个整体方便使用
- 版本信息：JAR文件可以有版本信息
- 可移植性

## 使用JAR文件的基础

| 操作 | 命令 | 
| :------: | :------: | 
| 创建JAR文件  | jar cf jar-file input-file(s) |
| 查看JAR文件内容 | jar tf jar-file | 
| 提取JAR文件内容 | jar xf jar-file | 
| 更新JAR文件内容（input-file为添加的文件） | jar uf jar-file input-file(s) | 
| 提取JAR文件中指定内容  | jar xf ar-file archived-file(s) | 
| 运行JAR文件（需要有入口类Main-Class） | java -jar jar-file | 

<!-- more -->

## 创建一个JAR文件

继续以上篇文章的内容为例。

- 使用`jar cf`命令打包，如下图：

![创建一个JAR文件](/image/cf1.png)

上面的`.`表示将当前文件夹下的内容作为input-file(s)，也可以使用通配符`*`达到同样的效果。

- 另外，还可以使用选项`v`在打包过程中输入一些打包信息，如下图所示：

![使用v输出打包信息](/image/cf2.png)

- 使用选项`m`指定入口类

首先，根据上一篇文章中classpath的知识，我们需要回到包含包的根的文件夹中进行打包工作，才能让程序正确运行。

其次，我们需要创建一个Manifest.txt文件，文件的内容是`Main-Class: com.rason.algs4.binarysearch.Test`,用于指定入口类。

然后再进行打包工作，如下图所示：

![指定入口类进行打包](/image/cf3.png)

打包完成之后，使用`java -jar`命令执行了一下这个jar文件，并将`rason`字符串作为参数传递给入口程序的main函数，输出结果正确。

## 查看JAR文件的内容

使用`jar tf jar-file`查看内容，如下图所示：

![查看JAR文件内容](/image/tf.png)

## 将classes添加到JAR文件的Classpath中

如果打出来的JAR文件，用到另一个JAR文件中的class，那么就需要在当前的JAR文件中指定Classpath为引用的JAR文件。

同样是在Manifest.txt文件中设置，如下所示：

`Class-Path: jar1-name jar2-name directory-name/jar3-name`

下面用一个例子来演示如何使用：

**第一步：**新建了一个新的工程2，编写了一个Test2.java类

```
package com.rason.ref;

public class Test2 {
	public String sex(int sex){
		if(sex == 0){
			return "女";
		}else{
			return "男";
		}
	}
}

```

**第二步：**对Test2.java类进行编译打包，如图所示：

![Test2编译打包](/image/cf-classpath1.png)

**第三步：**工程1中的Test.java引用工程2中的Test2.java类

```
package com.rason.algs4.binarysearch;

import com.rason.ref.Test2;

public class Test {

	public static void main(String[] args) {
		Test2 t2 = new Test2();
		System.out.println("Hello world!Hello " + args[0] + ", sex: "+t2.sex(Integer.parseInt(args[1])));
	}

}

```

**第四步：**不设置`Manifest.txt`中的`Class-Path`对Test.java进行编译、打包、执行，如下图所示：

![不设置Manifest.txt中Class-Path的结果](/image/cf-classpath2.png)

发现报错，没有找到Test2.class的定义，证明不设置Class-Path是有问题的。

**第五步：**在`Manifest.txt`中添加`Class-Path`对Test.java进行编译、打包、执行

在`Manifest.txt`中添加Class-Path,如下所示：

```
Main-Class: com.rason.algs4.binarysearch.Test
Class-Path: Test2.jar
```

![设置Manifest.txt中Class-Path的结果](/image/cf-classpath3.png)

如下图所示，当我们设置了Class-Path后打包执行还是报错，是因为Test2.jar没有跟rason.jar在同一个目录下。使用`cp`命令将其移动到同一目录下，执行成功。

## 其它信息

Manifest中还可以设置`Name`，`Specification-Title`，`Specification-Version`等信息，具体可自行搜索。
另还有一些关于签名增强安全性等内容也可以自行了解，此处只做一些简单应用的介绍。

## 总结

本文主要介绍了JAR文件的使用，包括如何创建，查看，提取和执行，还介绍了如何将另一个JAR文件作为Classpath引用。
