title: 我们为什么需要动态代理
categories: Java
date: 2015-10-22 09:38:56
tags: 动态代理
description:  RPC动态代理 动态代理
---

什么是代理？你可以理解为现实中的中介，比如房地产中介。你需要买套二手房，过户那个手续是需要自己做的，其余什么跟房东协商跑腿之类的工作交给中介去做，那么这个中介就是你的代理了。用代码来展示这个过程就是：

```
//买房的接口
public interface HouseService {
	public void buy();
}

//你自己
public class You implements HouseService {

	public void buy() {
		System.out.println("房产过户");
	}

}

//房地产中介
public class Proxy implements HouseService {
	private You you;
	
	public Proxy(You you) {
		this.you = you;
	}

	public void buy() {
		System.out.println("跟房东沟通扯淡");
		you.buy();
		System.out.println("交完房跟房东喝酒继续扯淡，不关我的事了");
	}

}
```

代码很简单而又似曾相识，就是自己做核心的实际，其它杂事交给其他人来做。这话听起来是不是有点熟悉，你猜对了，这就是AOP。那么动态代理又是怎样的，为什么会存在动态代理？且让我再举个例子：

<!-- more -->

比如我们去商业中心吃个饭，我们需要开车过去，吃完饭，开车回来：

```
public interface EatingService {
	public void eating();
}

public class YouEating implements EatingService {

	public void eating() {
		System.out.println("和女朋友吃饭");
	}

}

public class EatingProxy implements EatingService{
	public void eating(){
		System.out.println("开车去");
		System.out.println("和女朋友吃饭");
		System.out.println("开车回");
	}
}
```

好，上周和女朋友吃饭去了，这周要和女朋友去钓鱼：

```
public interface FishingService {
	public void fishing();
}

public class YouFishing implements FishingService {

	public void fishing() {
		System.out.println("和女朋友钓鱼");
	}

}

public class FishingProxy implements FishingService{
	public void fishing(){
		System.out.println("开车去");
		System.out.println("和女朋友去钓鱼");
		System.out.println("开车回");
	}
}
```

发现问题了没有？无论跟女朋友去吃饭，去钓鱼或者去看电影什么的，我都要开车去开车回，我不得不为每件事情都添加一个代理类。可想而知，针对每件事情都新建一个代理类，马上就类爆炸了，代码洁癖的人绝对会疯掉。动态代理就在这种情况下出现了，现在大概能知道动态代理解决的是什么问题了吧？

现在写个动态代理解决上面的问题,我们写一个动态代理类来为每件事情加入开车去和开车回两个动作：

```
public class CarHandler implements InvocationHandler {
	private Object realObject;
	
	public CarHandler(Object realObject) {
		this.realObject = realObject;
	}

	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		System.out.println("开车去");
		method.invoke(realObject, args);
		System.out.println("开车回");
		return null;
	}

}

```

好，现在上面这个调用处理器帮我们为每件事情都加入了开车去和开车回两个动作，那么我们要怎么用呢？我们写个测试类看一下：

```
//和女朋友一起愉快地玩耍测试类
public class DoWithGFTest {
	public static void main(String[] args) {
		System.out.println("----------测试吃饭动态代理-----------");
		EatingService eating = new YouEating();  //实际吃饭
		CarHandler eatCarHanlder = new CarHandler(eating);   //开车去开车回动作
		EatingService dynamicEatingProxy = (EatingService) Proxy.newProxyInstance(eating.getClass().getClassLoader(),
				eating.getClass().getInterfaces(), eatCarHanlder);  //吃饭代理
		dynamicEatingProxy.eating();
		
		
		System.out.println("----------测试钓鱼动态代理-----------");
		FishingService fishing = new YouFishing();
		CarHandler fishCarHanlder = new CarHandler(fishing);
		FishingService dynamicFishingProxy = (FishingService) Proxy.newProxyInstance(fishing.getClass().getClassLoader(),
				fishing.getClass().getInterfaces(), fishCarHanlder);
		dynamicFishingProxy.fishing();
	}
}

输出：
----------测试吃饭动态代理-----------
开车去
和女朋友吃饭
开车回
----------测试钓鱼动态代理-----------
开车去
和女朋友钓鱼
开车回

```

好吧，现在你可以把吃饭代理EatingProxy和钓鱼代理FishingProxy这两个类删掉了，我们只需要CarHandler就足够了，动态代理可以为你需要的每件事情都加入开车去和开车回两个动作了。

现在，我们回想一下，在我们所以的框架中那些地方用上了动态代理？日志，远程通信，缓存，访问控制这些是不是都有？剩下的读者自己想象挖掘了。

原创文章，欢迎转载，无需注明出处。

