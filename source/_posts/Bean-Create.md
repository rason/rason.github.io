title: Bean的创建
categories: Spring
date: 2016-01-05 15:35:54
tags: Bean
description: Bean的创建
---

## 前言

在上篇文章[IOC容器如何工作](http://rason.me/2015/12/31/How-do-IOC-work/)中，我们了解了Spring IOC容器创建过程，同时重点了解了Bean定义的加载这一步骤。在IOC容器创建过程中还有一个重要的步骤就是非懒加载单例的实例化工作，通过这个步骤可以大概地了解Spring IOC中Bean的创建。

## 实例化单例

实例化单例在AbstractApplicationContext的refresh()方法中的`finishBeanFactoryInitialization(beanFactory)`一行代码实现。其实际操作在DefaultListableBeanFactory的`preInstantiateSingletons()`方法完成，我们来看一下这个方法的源码：

```
public void preInstantiateSingletons() throws BeansException {
	if (this.logger.isInfoEnabled()) {
		this.logger.info("Pre-instantiating singletons in " + this);
	}

	synchronized (this.beanDefinitionMap) {
		for (String beanName : this.beanDefinitionNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					final FactoryBean factory = (FactoryBean) getBean(FACTORY_BEAN_PREFIX + beanName);
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
							public Boolean run() {
								return ((SmartFactoryBean) factory).isEagerInit();
							}
						}, getAccessControlContext());
					}
					else {
						isEagerInit = factory instanceof SmartFactoryBean && ((SmartFactoryBean) factory).isEagerInit(); 
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
				else {
					getBean(beanName);
				}
			}
		}
	}
}
```

代码的逻辑不算复杂，这里重点是出现了FactoryBean。FactoryBean是一个生生产Bean的Bean，开发者可以通过继承FactoryBean的方式，实现其getObject()方法即可获得自己想要的Bean。也就是说FactoryBean可以让我们很灵活地获取想要的Bean。比如一个Bean的初始化有很多复杂的配置，我们是不是得在xml文件中填写很多配置项？而通过继承FactoryBean的方法，我们就可以很灵活地精简掉这些配置。另外，Spring获取FactoryBean本身的对象是通过前面加上&来完成的。

<!-- more -->

实际上获取Bean的实例是通过AbstractBeanFactory的`getBean()`方法，该方法有点长，这里就不贴代码了。请看下面的总体流程图：

![实例化单例过程](https://raw.githubusercontent.com/rason/rason.github.io/master/image/springpreInstantiateSingletons.jpeg)

最终，我们可以看出，Bean的创建是在自动装配工厂`AbstractAutowireCapableBeanFactory`中的`createBean()`方法完成。

## IOC扩展点

Spring支持一些扩展点可以让我们更灵活地得到想要的Bean,主要有：

- **BeanFactoryPostProcessor**
- **BeanPostProcessor**
- **InitializingBean**
- **DisposableBean**
- **FactoryBean**

这些扩展点的具体用户可以在网上找一些例子看一下。要想熟悉的使用Spring就必须熟悉这些扩展点的使用。

## 总结

- **大致了解Bean的创建过程，具体需自行源码学习**
- **IOC容器的扩展点**
