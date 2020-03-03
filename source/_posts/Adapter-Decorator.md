title: 'Adapter&Decorator'
categories: DesignPattern
date: 2016-02-11 18:21:30
tags: Adapter Decorator
description: Adapter Decorator
---

## 前言

今天简单地复习一下另外两个设计模式，适配器模式和装饰器模式。

## 适配器模式（Adapter）

适配器就是把一个类的接口边换成客户端所能接收的另一种接口，从而使两个接口不匹配而无法一起工作的两个类能购在一起工作。适配器模式一般包含以下三个角色：

- Target（目标接口）：所要转换的所期待的接口。
- Adaptee（源角色）：需要适配的接口。
- Adapter（适配器）：将源接口适配成目标接口，实现目标接口。

适配器的作用就是将一个接口适配到另一个接口，在Java的I/O类库中有很多这样的需求，如将字符串数据转变成字节数据保存到文件中，将字节数据转变成流数据等。下面以InputStreamReader和OutputStreamWriter类为例介绍适配器模式。

InputStreamReader和OutputStreamWriter类分别继承了Reader和Writer类，但是创建它们的对象必须在构造函数中传入一个InputStream和OutputStream的实例。InputStreamReader和OutputStreamWriter的作用就是将InputStream和OutputStream适配到Reader和Writer。

InputStreamReader的类结构图如下：

![InputStreamReader类结构图](/image/web-inputstreamreader.png)

<!-- more -->

InputStreamReader继承了Reader类，并且持有InputStream的引用，这里是通过StreamDecoder类间接持有，因为从byte到char要经过编码。

很显然适配器就是InputStreamReader类，而源角色就是InputStream的实例，目标接口就是Reader类。同理，OutputStreamWriter也是一样的方式。

在I/O类库中还有很多类似的用法，如StringReader将一个String类适配到Reader，ByteArrayInputStream适配器将byte数组适配到InputStream流处理接口。

## 装饰器模式（Decorator）

装饰器模式，顾名思义，就是将某个类重新装扮一下，使得它比原来更“漂亮”，或者功能上更强大，这就是装饰器模式所要达到的目的。但是作为原来的这个类使用者还不应该感受到装饰前后有什么不同，否则就破坏了原有类的结构了，所以装饰器模式要做到对被装饰类的使用者透明，这是对装饰器模式的一个要求。

典型的装饰器模式类结构如下：

![装饰器模式类结构图](/image/web-decorator.png)

图中角色描述如下：

- Component：抽象组件角色，定义一组抽象的接口，规定这个被装饰组件都有哪些功能。
- ConcreteComponent：实现这个抽象组件的所有功能。
- Decorator：装饰器角色，它持有一个Component对象实例的引用，定义一个抽象组件一致的接口。
- ConcreteDecorator：具体的装饰器实现，负责实现装饰器角色定义的功能。

在Java I/O类库中有很多不同的功能组合情况，这些不同的功能组合都是使用装饰器模式实现的，下面以FilterInputStream为例介绍装饰器模式的使用。

FilterInputStream类结构图：

![FilterInputStream类结构图](/image/FilterInputStream.png)

InputStream类就是以抽象组件存在的，而FileInputStream就是具体组件，它实现了抽象组件的所有接口。FilterInputStream就是装饰角色，它实现了InputStream类的所有接口，并且持有InputStream的对象实例的引用，BufferedInputStream是具体的装饰器实现者，它给InputStream类附加了功能，这个装饰器的作用就是使得InputStream读取的数据保存在内存中，而提高读取的性能。与这个装饰器类有类似功能的还有LineNumberInputStream类，它的作用就是提高按行读取数据的功能，它们都让InputStream类增强了功能，或者提升了性能。

## 总结

装饰器模式和适配器模式都有一个别名就是包装模式（Wrapper），它们的作用看似都是起到包装一个对象的作用，但是使用它们的目的是很不一样的。

- 适配器模式的意义是要将一个接口转变成另外一个接口，它的目的是通过改变接口来达到重复使用的目的。
- 装饰器模式不是要改变被装饰对象的接口，而恰恰要保持原有的接口，但是增强原有对象的功能，或者改变原有对象的处理方法而提高性能。
