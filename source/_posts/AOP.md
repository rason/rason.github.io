title: AOP
categories: Spring
date: 2016-01-08 14:35:22
tags: AOP
description: AOP Sring Aop
---

## 前言

在[Bean的创建](http://rason.me/2016/01/05/Bean-Create/)这篇文章的最后我们提到了扩展点，而Spring AOP就是以FactoryBean这个扩展点为基础实现的。下面我们通过ProxyFactoryBean的源码来了解一下Spring的AOP功能。

## ProxyFactoryBean

在研究AOP之前，我们需要一些简单的动态代理知识，因为Spring AOP就是基于动态代理实现的，可以首先阅读我之前写的一篇文章[我们为什么需要动态代理](http://rason.me/2015/10/22/Why-we-need-dynamic-proxy/)。我们知道，要在Spring中使用AOP，配置大概如下：

```
<bean id="sleepHelperAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
     <property name="advice" ref="sleepHelper"/>
     <property name="pointcut" ref="sleepPointcut"/>
</bean>

<bean id="humanProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
     <property name="target" ref="human"/>
     <property name="interceptorNames" value="sleepHelperAdvisor" />
     <property name="proxyInterfaces" value="test.spring.aop.bean.Sleepable" />
</bean>
```

可以看出，在Spring中是通过ProxyFactoryBean这个类来获取代理对象的，我们看一下ProxyFactoryBean的`getObject()`方法，看看它是如何返回一个我们需要的代理对象。

<!-- more -->

```
public Object getObject() throws BeansException {
	initializeAdvisorChain();
	if (isSingleton()) {
		return getSingletonInstance();
	}
	else {
		if (this.targetName == null) {
			logger.warn("Using non-singleton proxies with singleton targets is often undesirable. " +
					"Enable prototype proxies by setting the 'targetName' property.");
		}
		return newPrototypeInstance();
	}
}
```

从上面的代码我们可知其大意，先初始化拦截器链，然后返回一个代理对象的实例。初始化拦截器方法就不多说了，我们来深入看看其`getSingletonInstance（）`方法：

```
private synchronized Object getSingletonInstance() {
	if (this.singletonInstance == null) {
		this.targetSource = freshTargetSource();
		if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
			// Rely on AOP infrastructure to tell us what interfaces to proxy.
			Class<\?> targetClass = getTargetClass();
			if (targetClass == null) {
				throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
			}
			setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
		}
		// Initialize the shared singleton instance.
		super.setFrozen(this.freezeProxy);
		this.singletonInstance = getProxy(createAopProxy());
	}
	return this.singletonInstance;
}
```

先刷新目标资源，也就是发生实际动作的对象；然后设置目标对象的接口；再通过`createAopProxy`方法创建一个AopProxy,这个AopProxy实现了InvocationHandler接口，因此这个AopProxy就是我们需要的目标类了。我们看一下createAopProxy方法是怎么创建AopProxy的：

```
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<\?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```

可以看出这里会根据给出的配置信息创建一个AopProxy对象，然后根据getProxy方法就能获取到最终的动态代理对象：

```
public Object getProxy(ClassLoader classLoader) {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
	}
	Class<\?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised);
	findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
	return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

至此，我们清楚了Spring是怎么创建动态代理对象的，并且知道AopProxy的实现类就是我们需要的目标对象。因此AopProxy实现类的invoke方法就是我们需要做文章的地方了,我们看下JDK动态代理JdkDynamicAopProxy类的invoke方法：

```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	MethodInvocation invocation;
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Class<?> targetClass = null;
	Object target = null;

	try {
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
			// The target does not implement the equals(Object) method itself.
			return equals(args[0]);
		}
		if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
			// The target does not implement the hashCode() method itself.
			return hashCode();
		}
		if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			// Service invocations on ProxyConfig with the proxy config...
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;

		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		// May be null. Get as late as possible to minimize the time we "own" the target,
		// in case it comes from a pool.
		target = targetSource.getTarget();
		if (target != null) {
			targetClass = target.getClass();
		}

		// 获取该方法的拦截器链
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		// Check whether we have any advice. If we don't, we can fallback on direct
		// reflective invocation of the target, and avoid creating a MethodInvocation.
		if (chain.isEmpty()) {
			// We can skip creating a MethodInvocation: just invoke the target directly
			// Note that the final invoker must be an InvokerInterceptor so we know it does
			// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
		}
		else {
			// We need to create a method invocation...
			invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			// Proceed to the joinpoint through the interceptor chain.
			retVal = invocation.proceed();
		}

		// Massage return value if necessary.
		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// Special case: it returned "this" and the return type of the method
			// is type-compatible. Note that we can't help if the target sets
			// a reference to itself in another returned object.
			retVal = proxy;
		}
		else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
			throw new AopInvocationException(
					"Null return value from advice does not match primitive return type for: " + method);
		}
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			// Must have come from TargetSource.
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```

从上面的代码可以看出，我们的着重点应该是获取方法的拦截器链，就是然后创建ReflectiveMethodInvocation对象，再调用其`proceed()`方法：

```
public Object proceed() throws Throwable {
	//	We start with an index of -1 and increment early.
	if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
		return invokeJoinpoint();
	}

	Object interceptorOrInterceptionAdvice =
			this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
	if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
		// Evaluate dynamic method matcher here: static part will already have
		// been evaluated and found to match.
		InterceptorAndDynamicMethodMatcher dm =
				(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
		if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
			return dm.interceptor.invoke(this);
		}
		else {
			// Dynamic matching failed.
			// Skip this interceptor and invoke the next in the chain.
			return proceed();
		}
	}
	else {
		// It's an interceptor, so we just invoke it: The pointcut will have
		// been evaluated statically before this object was constructed.
		return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	}
}
```

这段代码挨个执行拦截器的前置方法，然后执行目标方法，再反过来挨个执行拦截器的后置方法，最后返回。这里的代码比较有意思，是递归式调用。这样，就实现了AOP功能了。

## 总结

根据上面分析，Spring AOP就是运用了动态代理的方式是实现的，因此重点分析的就是Spring如何实现InvocationHandler接口。所以，有了动态代理的基础，理解Spring AOP的实现就不是一个困难的问题了。
