title: 你们公司的CTO就是一个JAVA线程池
categories: Java
date: 2015-10-30 10:50:24
tags: JAVA线程池
description: JAVA线程池 线程池
---

上篇文章谈了一下多线程中Synchronized关键字的使用，但实际的开发中，不建议自己new一个线程来处理耗时比较长的业务。

为什么不建议自己为耗时较长的任务开一个单独线程呢？假如我们每来一个任务都开一个线程来处理，那么如果任务很多而且耗时又长会带来什么问题？
- 创建和销毁线程耗时耗资源
- 活动的线程按时间片来调度，线程太多会对CPU和内存等资源造成压力

举个简单的例子：
一家创业公司，做一个O2O项目。CEO会不会为开发任务中的每个任务都聘请一个程序员来写代码？显然是不会的。CEO当然会聘请一个CTO，让CTO把这个项目架构搭起来，任务按轻重缓急细化，假设细化成了100个子任务。此时CTO也不可能请100个程序员来完成这100个任务啊。真的这样做会带来什么问题？很明显是烧钱啊，烧钱还浪费资源啊，做完了之后哪需要那么多程序员。那么CTO会怎么做就是线程池的问题了。我们分步骤看一下这个问题：

- 你就是CEO（写代码当然是你说了算，你怎么写你的事）
- 你需要CTO（CTO就是线程池，就是你写代码创建一个线程池咯，当然你有钱任性可以不用线程池）
- CTO招聘五个全职程序员（程序员就是线程，五个就是线程池的大小）
- CTO把项目分成100个子任务列入项目管理系统（项目管理系统就是工作队列）
- 五个程序员从羡慕管理系统中取出任务来完成（线程从队列中取出任务）
- 程序员做完了任务继续在项目管理中领任务（苦逼的程序员啊）

大体上应该按照上述的方式才算合理地使用线程，而不是每个任务都自己手动开启一个线程，那么用代码应该怎么实现这样的功能？JDK已经封装好了线程池的相关代码，我们来简单看一下应该怎么用。

<!-- more -->

## ExecutorService

先说重点，ExecutorService就代表一个线程池。来看一个例子：

```
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolTest {
	static class ServiceThread implements Runnable{

		public void run() {
			System.out.println("五个程序员做任务");
		}
		
	}
	public static void main(String[] args) {
		final ExecutorService executorService = Executors.newSingleThreadExecutor();//创建一个线程池
		executorService.execute(new ServiceThread());//把任务放进线程池中管理执行
		executorService.shutdown();//关闭线程池
	}

}
```

代码中有注释，各行代码的作用就不说。我们来探讨一下Executors这个类，查看JDK源码会发现这是一个工具类，可以返回以下接口的实例：

- ExecutorService             线程池接口
- ScheduledExecutorService    类似Timer/TimerTask类似，可以做一些重复执行任务的线程池接口，继承于ExecutorService
- ThreadFactory               线程工程
- Callable                    与Runnable类似，有返回值

ExecutorService接口有多个实现，上述代码中只是获取一个最简单的实例（只有一个线程），具体的一些实例如下：

- newFixedThreadPool           固定线程数量，队列无界的线程池，无可用线程则放入队列中等候执行。
- newSingleThreadExecutor      单一线程，队列无界的线程池，可以保证任务按顺序执行。
- newCachedThreadPool          缓存线程池，最多可有Integer.MAX_VALUE个线程。主要用于频繁且耗时短的异步任务，线程闲置一分钟会被关闭。

ScheduledExecutorService接口的返回实例就不一一说了，大家看下JDK的源码即可，里面也有详细的解析。

## 线程池类图结构

说到这里，大家肯定已经被线程池所用到的一些类会接口搞乱了，借用一张类图让大家理清一下结构：

![线程池类图结构](/image/threadpool.png)

类图还比较好理解，查看JDK源码直接从Executors入口，顺着查看很好理解，就不详说了。

## executorService.execute

还有一个重点就是`executorService.execute`方法究竟做了什么事情，具体内容见JDK源码，大概内容就正如我文章开头举得例子一样。

- 当任务到达时，首先判断核心线程满了没，没有就新建线程，满了执行下一步
- 判断队列是否满了，没有则放入任务队列，满了执行下一步
- 判断线程池是否满了，没有则创建新线程执行，满了执行下一步
- 按策略处理无法执行的任务

## 总结

- 上面的简单代码解析思路可能不清晰，建议读者自行从`Executors`类入口一探究竟。
- 文章开头的例子思路才是重要的，思路懂了代码看起来就容易多了。
