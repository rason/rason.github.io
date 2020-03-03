---
title: TraceId 不见了
date: 2020-03-02 15:37:55
tags:
---

最近在帮忙排查一个问题，通过 Spring 集成 MQTT，在 接收 MQTT 消息的业务入口处打印出的日志没有 Sleuth 的 TraceId。但是，将接收到的消息丢到线程池去处理的时候，却又有了 TraceId。

代码大概逻辑如下：

```
public class MqttMessageReceiver implements MqttCallbackExtended {
    @Override
    public void messageArrived(final String topic, final MqttMessage message) {

        log.info("这行日志没有 TraceId ");

        mqttAsyncTaskExecutor.execute(() -> {
            this.messageHandle(topic, message.getPayload());
        });

    }

    @Override
    public void messageHandle(String key, Object value) {
		log.info("这行日志有 TraceId ");
    ｝
}
```

而另外一个工程集成的 Sleuth 和 MQTT 是有 TraceId 的，所以下意识只做了一些简单的判断：

1. 难道是 Sleuth 的集成方式不对？梳理了一下 pom 依赖及其版本，没有解决问题。其实也能想到，因为部分是能打印出来的。
2. 跟着了一下代码，发现代码中有进行 MDC 的相关操作，莫非没有的部分日志是把上下文清空了？跟踪了一下代码并没有这样的操作，只有自定义的 MDC 字段操作，即使删除了也没有解决问题。
3. 难道是 Logback 配置不对导致没打印出来？检查了一下配置，有 TraceId 和没有 TraceId 的打印配置都是一样的，也排除了这个原因。

冷静下来再想一下，既然上面的一顿操作都没有解决问题，那么问题的本质一定是：**第一行日志线程上下文中有 TraceId，第二行日志线程上下文中没有 TraceId**。

由于自己没有对 Sleuth 做过深入的研究，所以得出这个结论的时候还是对认知有点冲击的，因为：

1. 业界的链路追踪技术，一般都是在应用入口处就初始化了追踪上下文信息，所以对于第一行日志应该是有 TraceId 信息才对的。
2. 反倒第二行日志没有才符合自己的认知的，因为新开了一个线程，追踪上下文应该没有传递过去才对。

没辙，只好带着这两个问题粗略地去跟着一下 Sleuth 的源码。

### 为什么入口没有追踪上下文信息

首先是第一个问题，第一行日志没有追踪上下文信息，难道是 MQTT 的集成入口没有初始化追踪上下文？有这样的推测是因为 MQTT 消息确实不是常规的业务入口，因为之前做的都是 Web 方面的开发较多，这是第一次在项目中使用 MQTT。

对于 Web 应该，一般会通过过滤器或者拦截器的技术方案初始化追踪上下文，这样到业务代码时就已经有了 TraceId 信息。所以**推测通过 MQTT 消息的方式进来可能没有经过类似的过滤器或者拦截器**。

作出了这样的推测，那就沿着这个方向是跟踪代码。对于 Sleuth 我并不熟悉，怎么快速的找到我需要的信息呢？

#### 全局搜索大法

此时只好祭出我常用的全局搜索大法了，IDEA 快捷键 `command + shift + F` 全文搜索 traceId，立马就搜到了重要的信息：

`org.springframework.cloud.sleuth.log.Slf4jScopeDecorator`

简单浏览了一下这个类，里面有一个 `TraceContext` ，这个命名是在是太赞了，一看就是我们要找的东西，保存追踪上下文的信息。

#### 引用搜索大法

知道了保存我们需要信息的类，那么通过快捷键 `option + F7` 查看哪些类引用了这个 TraceContext 就知道哪里会设置追踪上下文信息了。

如下图所示，发现引用的类还不少，大概看了下类名，初步判断红框里面这些类相关性可能会大一点：

![TraceContext-ref](/image/TraceContext-ref.png)

简单浏览了这几个类的注释，发现 `TracingChannelInterceptor` 应该就是我需要找的类，注释如下：

```
This starts and propagates Span.Kind#PRODUCER span for each message sent (via native headers. It also extracts or creates a Span.Kind#CONSUMER span for each message received. This span is injected onto each message so it becomes the parent when a handler later calls MessageHandler.handleMessage(Message), or a another processing library calls nextSpan(Message).

This implementation uses ThreadLocalSpan to propagate context between callbacks. This is an alternative to ThreadStatePropagationChannelInterceptor which is less sensitive to message manipulation by other interceptors.
```

大致的意思对每条发送的消息开始并传递一个 `Span.Kind#PRODUCER` 类型的 span。从这里大致可以推测，如果链路经过了这个 `TracingChannelInterceptor`，那么应该是会有 TraceId 的，难道是因为没有经过？

#### 断点调试大法

要知道链路有没有经过一个类最简单的方法当然是断点调试了。于是我就在打印第一行日志的地方打了个断点，发现请求通过的链路截图如下：

![has_no_trace_id](/image/has_no_trace_id.png)

链路比想象中的要短，通过简单的回调函数就到了我的业务类 `MqttMessageReceiver`，也就是文章开头的类，并没有经过 `TracingChannelInterceptor`，可能这就是没有 TraceId 的原因。

#### 对比验证大法

既然另外一个工程集成的 Sleuth 和 MQTT 是有 TraceId 的，那么对比一下调用堆栈就好了，于是也在业务类接收 MQTT 消息的地方进行了调试：

![has_trace_id](/image/has_trace_id.png)

发现调用堆栈多了很多类，图中我用红框框出来了，最后的 `MqttReceiver` 是这个工程接收 MQTT 消息的业务类。但是多出的堆栈调用中也没有 `TracingChannelInterceptor` 这个类，这时开始怀疑自己之前的推测是错误的。但是既然不一样的地方就是在红框这些地方，那么玄机一定就在这里面，所以我逐个逐个类点开分析了一遍，果然有所发现：

![TracingChannelInterceptor](/image/TracingChannelInterceptor.png)

原来在 `AbstractMessageChannel` 类发送消息之前会经过 `TracingChannelInterceptor.png` 这个拦截器，到这里基本就验证了之前的推测，问题得到了实质性的突破。但是，为了两个工程经过的堆栈会有这么大的不同呢？

再次使用对比验证大法，通过走读两个工程接入 MQTT 的代码终于发现了问题的原因：**两个工程集成 MQTT 的方式并不相同，出现问题的工程是直接 MQTT client，没有问题的工程是通过 Spring Integration 集成的**。也就是说，只有 `Spring Integration` 集成的 MQTT 才会经过 `TracingChannelInterceptor`，才会有追踪上下文信息。这里也不得不感叹，**Spring 封装得太好了，但对于使用者来说掩盖了太多东西了，遇到问题的时候排查起来还是得把封装的内容一层层剥开** 。

### 为什么丢到线程池反倒有追踪上下文信息

第一个问题已经解决了，但是为什么丢到线程池反倒有追踪上下文信息呢，推测应该是在某个地方传递过去了。有了上面的经验，这次直接就祭出断点调试大法：

![TraceRunnable](/image/TraceRunnable.png)

一下子就看到了问题的本质，经过了一个 `TraceRunnable` 的类，看了一下这个类的内容，就是简单的代理了 `Runable` ，处理了追踪上下文信息，所以就是因为经过了这层代理才会有了追踪上下文信息。

但奇怪的是，我的代码中开启的线程池并没有显式地使用 `TraceRunnable` 这个类。再一次祭出调试大法，在 `mqttAsyncTaskExecutor.execute` 这行代码打上断点，然后 `Step Into`，发现 `ThreadPoolTaskExecutor` 被 `CGLIB` 增强了： 

![cglib_enhancer](/image/cglib_enhancer.png)

增强之后的代理类是 `LazyTraceThreadPoolTaskExecutor`，如下图所示：

![LazyTraceThreadPoolTaskExecutor](/image/LazyTraceThreadPoolTaskExecutor.png)

`LazyTraceThreadPoolTaskExecutor` 的核心作用就是将 `Runable` 包装成 `TraceRunnable`，那么现在的问题又变成了：我也没显式配置代理增加变成 `LazyTraceThreadPoolTaskExecutor` 啊，看来又是 Spring 封装搞的鬼。

继续查上面堆栈信息，又有了新的发现，如下图所示：

![ExecutorBeanPostProcessor](/image/ExecutorBeanPostProcessor.png)

原来引入了 Sleuth 之后，会有一个 Executor 的后置处理器 `ExecutorBeanPostProcessor`，核心源码如下：

```
@Override
	public Object postProcessAfterInitialization(Object bean, String beanName)
			throws BeansException {
		if (bean instanceof LazyTraceThreadPoolTaskExecutor
				|| bean instanceof TraceableExecutorService
				|| bean instanceof LazyTraceAsyncTaskExecutor
				|| bean instanceof LazyTraceExecutor) {
			log.info("Bean is already instrumented " + beanName);
			return bean;
		}
		if (bean instanceof ThreadPoolTaskExecutor) {
			if (isProxyNeeded(beanName)) {
				return wrapThreadPoolTaskExecutor(bean);
			}
			else {
				log.info("Not instrumenting bean " + beanName);
			}
		}
		else if (bean instanceof ExecutorService) {
			if (isProxyNeeded(beanName)) {
				return wrapExecutorService(bean);
			}
			else {
				log.info("Not instrumenting bean " + beanName);
			}
		}
		else if (bean instanceof AsyncTaskExecutor) {
			if (isProxyNeeded(beanName)) {
				return wrapAsyncTaskExecutor(bean);
			}
			else {
				log.info("Not instrumenting bean " + beanName);
			}
		}
		else if (bean instanceof Executor) {
			return wrapExecutor(bean);
		}
		return bean;
	}
```

作用就是将各类 Executor 包装成含有插入追踪上下文的代理 Executor。到这里，两个问题终于得到了最终的解决。

### 总结

回顾了一下整个过程，还是因为引入一项新的技术点(如 Sleuth 和 MQTT)，Spring 封装了很多细节内容。

如：

只有通过 `Spring Integration` 集成的 MQTT 才有上下文信息，这也告诉我们 Spring 很多东西都是配套使用的，有官方的提供的集成方式还是优先使用官方的，少用原生集成方式。

又如：

引入了 Sleuth 之后，会自动通过 `ExecutorBeanPostProcessor` 对 `ThreadPoolTaskExecutor` 进行增强。

对于 Spring 这种高度的封装，我们还是需要一层层剥开探讨其细节才能更好解决问题，当排查的次数多了，基本的套路也是类似的。




