title: Java多线程Synchronized小知识
categories: Java
date: 2015-10-23 17:48:04
tags: Synchronized
description: Java多线程 Java Synchronized
---

还记得那个大名鼎鼎的多线程售票例子吗？让我们来复习一下：

- 先来写个简单的售票服务类：

```
//售票服务器
public class TicketService{
	//余票
	private int remain;
	public TicketService(int remain) {
		super();
		this.remain = remain;
	}
	public int sell(){
		if(remain > 0){
			remain -= 1;
			try {
				Thread.sleep(100);//模拟实际复杂业务耗时
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			return remain;
		}else{
			return -1;
		}
	}
	public int getRemain() {
		return remain;
	}
	public void setRemain(int remain) {
		this.remain = remain;
	}
}

```

<!-- more -->

- 再来写一个售票程序,可以看做一个controller

```
//售票程序，可以看做controller
public class SalesController implements Runnable {
	private TicketService ticketService;

	public SalesController(TicketService ticketService) {
		super();
		this.ticketService = ticketService;
	}
	
	public void run() {
		int remain = ticketService.sell();
		String threadName = Thread.currentThread().getName();
		if(remain >= 0){
			System.out.println("售票点"+threadName+",成功卖出一张票，余票："+remain);
		}
	}

}

```

- 现在可以模拟一下多个售票窗口同时售票了

```
public class BuyTicketTest {

	public static void main(String[] args) {
		TicketService ticketService = new TicketService(100);
		SalesController salesController = new SalesController(ticketService);
		Thread[] threads = new Thread[1000];
		int i = 0;
		for (Thread thread : threads) {
			i++;
			thread = new Thread(salesController, ""+i);
			thread.start();
		}
	}

}

```

测试代码发起了一千个线程来售票，就好像是一千个人同时在抢购春运车票。**注意：虽然开了一千个线程，但是每个线程都是跑同一个售票程序对象salesController，而该对象引用的是一个ticketService对象，因此一千个线程访问的是同一个余票变量remain。**

所以，上面的代码跑起来你应该能猜到是线程不安全的，输出结果如下：

```
售票点1,成功卖出一张票，余票：0
售票点8,成功卖出一张票，余票：0
售票点4,成功卖出一张票，余票：0
售票点7,成功卖出一张票，余票：0
...
...
售票点85,成功卖出一张票，余票：0
售票点86,成功卖出一张票，余票：0
售票点100,成功卖出一张票，余票：0
售票点99,成功卖出一张票，余票：0

```

结果跟猜测的一样，明显是会有问题的。那么怎么来避免这个问题？大家都在争抢remain这个资源，那么我们可以想到的方法当然是把remain这个资源给锁定，用完了再给别的线程用即可。Java中就是用Synchronized关键字来给资源加锁。

Synchronized加锁不仅局限于变量，也可以给方法，代码块和类加锁，同样可以达到锁定资源的效果。我们把这几种方式都试一下，改善上面的程序，使它能够线程安全。

## 给方法加锁

```
class SalesController

public synchronized void run() {
	...
}
```

给方法加锁的作用对象是调用该方法的对象，在本例中也就是salesController对象，作用范围是整个方法。也就是说多个线程的同一个salesController对象调用run方法，必须等待一个线程执行完了run方法后下一个线程才能得到执行的权限。因此，假如我们在测试代码中，为每个线程都new一个salesController的话，也是线程不安全的。

## 给代码块加锁

```
class SalesController

public void run() {
	synchronized (this) {
		int remain = ticketService.sell();
		String threadName = Thread.currentThread().getName();
		if(remain >= 0){
			System.out.println("售票点"+threadName+",成功卖出一张票，余票："+remain);
		}
	}
}
```

给代码块加锁的作用对象是调用该代码块的对象，作用范围就是{}包含的范围。上述代码的效果等同于上面的给方法加锁。

## 给一个类加锁

```
class SalesController

public void run() {
	synchronized (SalesController.class) {
		int remain = ticketService.sell();
		String threadName = Thread.currentThread().getName();
		if(remain >= 0){
			System.out.println("售票点"+threadName+",成功卖出一张票，余票："+remain);
		}
	}
}
```

给一个类加锁的作用对象当然就是一个类，无论你new了多少个实例，对于每个类的实例都是同步的。因此就可以克服给方法加锁情况中抛出的问题。另外给具体一个对象加锁和给静态方法加锁可以类推得出结论不再一一陈述。

总结：
- synchronized关键字最重要是看清作用的范围和作用的对象。
- 注意分清是作用一个类还是具体作用到一个实例。

原创文章，欢迎转载，无需注明出处。
