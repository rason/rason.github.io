title: IOC容器如何工作
categories: Spring
date: 2015-12-31 13:39:54
tags: IOC容器
description: IOC容器
---

## 前言

[上篇文章](http://rason.me/2015/12/28/Spring-Structure/)大概地了解了Spring的基本结构，知道了Spring的三个核心组件：Context，Bean和Core。IOC容器实际上是Context结合其他两个组件共同构建一个Bean的关系网。今天这篇文章我们来了解一下Context是如何构建这张关系网。

## 基础的IOC容器

我们知道BeanFactory是一个最基本的IOC容器，当然了Spring为我们准备了许多种IOC容器来使用，这样可以方便我们从不同的层面，不同的资源位置，不同的形式的定义信息来建立我们需要的IOC容器。而Context是在BeanFactory的基础上添加了访问资源，国际化和事件等功能。因此我们可以先从最基本的BeanFactory开始了解IOC容器，一个简单的IOC容器的创建过程：

```
ClassPathResource res = new ClassPathResource("beans.xml");  
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();  
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);  
reader.loadBeanDefinitions(res);  
```

上面的过程可以表示为一下几个步骤：

- 创建IOC配置文件的抽象资源
- 创建一个BeanFactory
- 把读取配置信息的XmlBeanDefinitionReader配置给BeanFactory
- 从定义好的资源位置读入配置信息，完成整个载入bean定义的过程

了解了上面几个步骤，基本上就对Spring IOC有了基础的了解。由于BeanFactory 是一个接口，在实际应用中我们一般使用ApplicationContext来使用IOC容器，它们也是IOC容器展现给应用开发者的使用接口。下面我们来看一下ApplicationContext是怎么建立一个IOC容器。

<!-- more -->

## ApplicationContext

通过ApplicationContext来建立一个IOC容器十分简单，我们可以通过new它的一个子类即可，比如：

```
ApplicationContext app = new ClassPathXmlApplicationContext("spring.xml");
```

我们看一下ClassPathXmlApplicationContext的构造方法：

```
public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
}

public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
	throws BeansException {

	super(parent);
	setConfigLocations(configLocations);
	if (refresh) {
		refresh();
	}
}
```
可以看出，在实例化的时候会调用`refresh()`方法。因此，我们可以猜测该方法所做的工作应该就和上面四个步骤差不多，读取资源，解析资源，完成Bean定义的载入。那么究竟是否真的如此，该方法定义在AbstractApplicationContext抽象类中，我们来看看该方法的源码：

```
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// 准备刷新，设置开始时间和活动标志
		prepareRefresh();

		// 告诉子类来刷新其内部的BeanFactory，返回一个刷新后的BeanFactory实例
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// 为此Context准备可以的BeanFactory，配置工厂的标准Context特性，比如Context的ClassLoader和post-processors
		prepareBeanFactory(beanFactory);

		try {
			// 允许Context的子类中的BeanFactory使用post-processing
			postProcessBeanFactory(beanFactory);

			// 实例化并调用在Context中注册为BeanFactoryPostProcessor的beans 
			invokeBeanFactoryPostProcessors(beanFactory);

			// 实例化并调用在Context中注册为BeanPostProcessor的beans,拦截bean的创建
			registerBeanPostProcessors(beanFactory);

			// 初始化MessageSource
			initMessageSource();

			// 初始化 event multicaster
			initApplicationEventMulticaster();

			// 具体的子类初始化其他特别的beans
			onRefresh();

			// 检查监听器beans并注册他们
			registerListeners();

			// 初始化所有的non-lazy-init单例
			finishBeanFactoryInitialization(beanFactory);

			// 完成刷新，调用LifecycleProcessor的onRefresh()方法并发布ContextRefreshedEvent事件
			finishRefresh();
		}

		catch (BeansException ex) {
			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}
	}
}
```

以上就是建立一个IOC的完整过程，了解里面的每一行代码基本上就了解大部分Spring原理和功能。其中，最重要的一个步骤当属获取一个刷新后的BeanFactory：

```
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

我们重点来看一下该行代码做了什么工作：

```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	refreshBeanFactory();
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (logger.isDebugEnabled()) {
		logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
	}
	return beanFactory;
}
```

可以看出，刷新BeanFactory的实际工作交给了可刷新的抽象子类AbstractRefreshableApplicationContext来实现：

```
protected final void refreshBeanFactory() throws BeansException {
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		customizeBeanFactory(beanFactory);
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
	catch (IOException ex) {
		throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
	}
}
```

在这个刷新方法中，如果有BeanFactory的话会关闭当前的BeanFactory，然后初始化一个新的BeanFactory。注意这里获得的BeanFactory原始类型为DefaultListableBeanFactory，我们知道BeanFactory有很多子类，在什么情况下使用什么样的子类非常重要，因此要主要BeanFactory类型的变化。在下面的loadBeanDefinitions方法中，BeanFactory的类型有作为BeanDefinitionRegistry的时候。

获取到了新的BeanFactory之后，就会为此BeanFactory加载Bean的定义，我们看一下loadBeanDefinitions(beanFactory)是如何加载Bean的定义,该方法是由抽象子类AbstractXmlApplicationContext来实现：

```
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	// 为给出的BeanFactory创建一个新的XmlBeanDefinitionReader实例
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	// 设置资源加载环境
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	//让子类提供reader自定义初始化，然后开始实际的加载Bean定义
	initBeanDefinitionReader(beanDefinitionReader);
	loadBeanDefinitions(beanDefinitionReader);
}
```

从上面的代码可知，该方法将完成加载，解析Bean的定义，也就是把用户定义的数据结构转化为IOC容器中的特定数据结构。Bean的解析和注册流程时序图如下图所示：

![Bean解析注册时序图](/image/springloadBeanDefinitions.jpg)

创建BeanFactory后，添加一些Spring本身需要的工具类，这个操作在AbstractApplicationContext的prepareBeanFactory方法中完成。AbstractApplicationContext中refresh方法接下来的三行代码对Spring的功能扩展性起了至关重要的重要。

```
postProcessBeanFactory(beanFactory);

invokeBeanFactoryPostProcessors(beanFactory);

registerBeanPostProcessors(beanFactory);
```

前两行让我们可以对已经构建的BeanFactory的配置做修改，后面一行可以让我们对创建Bean的实例对象时添加一些自定义的操作。

由于篇幅关系AbstractApplicationContext中refresh方法中的其他几行代码暂时不作深入的研究，今天主要是要消耗掉loadBeanDefinitions这行代码，了解Bean的解析和注册这一过程。

## 总结

- **首先我们从BeanFactory开始，了解了一个最简单IOC容器是如何构建起来的**
- **其次我们从实际应用出发，实例化一个ClassPathXmlApplicationContext，了解到构建应用的IOC容器的真正入口在AbstractApplicationContext的refresh方法**
- **然后大概说明了一个refresh方法中每行代码的大概作用**
- **再重点地了解了一下loadBeanDefinitions这行代码，了解Bean的解析和注册这一过程**
