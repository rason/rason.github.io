---
title: Learning Ribbon
date: 2019-07-09 19:48:59
tags:
---

研究一番 Eureka 之后, 就要顺带研究一下 Ribbon 了. 服务发现了, 怎么去做负载均衡呢, Ribbon 应运而生.

简单的来讲, 客户端负载均衡就是拿到一系列服务地址之后, 自己按照某种负载均衡算法去挑一个调用就完了. 

下面我们来看下 Ribbon 怎么实现负载均衡.

### @LoadBalanced

假设我们使用 `RestTemplate` 作为 HTTP Client.

首先我们要在 classpath 中引入 `spring-cloud-starter-netflix-ribbon` 依赖, 然后在 `RestTemplate` 上标记 `@LoadBalanced` 注解, 这样使用 `RestTemplate` 进行访问时就可以实现负载均衡了.

### LoadBalancerAutoConfiguration

Ribbon 究竟是怎么做到打了个注解就可能实现负载均衡了呢? 核心就在于 `LoadBalancerAutoConfiguration`.

```
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {

	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
            for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
                for (RestTemplateCustomizer customizer : customizers) {
                    customizer.customize(restTemplate);
                }
            }
        });
	}
```

这段代码有两个核心作用:

- 将 classpath 中被标记了 `@LoadBalanced` 注解的 `RestTemplate` 全部注入到 `restTemplates` 列表中.
- 将 `restTemplates` 列表中每个 `RestTemplate` 通过 `RestTemplateCustomizer` 进行了自定义.

RestTemplateCustomizer 究竟进行了什么自定义, 继续看下面的代码片段:

```
	@Configuration
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {
		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
	}
```

从这段代码可以看出, `RestTemplateCustomizer` 为每个 `RestTemplate` 添加了一个 `LoadBalancerInterceptor` 拦截器. 到这里, 我们应该大概能猜到 Ribbon 实现负载均衡的核心逻辑了. 就是通过拦截器将请求拦截下来, 然后获取要访问的服务名称, 根据服务名名称获取到服务列表, 最后根据某种负载均衡算法从列表中挑取一个进行调用.

当然, 这只是我们的猜想, 看看源码是不是.

### LoadBalancerInterceptor

```
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}
}
```

这个拦截器的代码比较简单, 只是从请求 URI 中获取到了服务名称, 然后调用负载均衡器 LoadBalancerClient 的 execute 方法, 这跟我们上面的设想差不多.

### RibbonLoadBalancerClient

LoadBalancerClient 的实现类是 RibbonLoadBalancerClient, 我们来看下他的 execute 方法.

```
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
		Server server = getServer(loadBalancer);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		RibbonServer ribbonServer = new RibbonServer(serviceId, server, isSecure(server,
				serviceId), serverIntrospector(serviceId).getMetadata(server));

		return execute(serviceId, ribbonServer, request);
	}
```

1. 先根据 serviceId 获取到 ILoadBalancer 实例, 跟踪代码会发现每一个 serviceId 都有一个对应的 spring 上下文, 找出属于自己的 ILoadBalancer. ILoadBalancer 是 Ribbon 的核心类。可以理解成它包含了选取服务的规则(IRule)、服务集群的列表(ServerList)、检验服务是否存活(IPing)等特性，同时它也具有了根据这些特性从服务集群中选取具体一个服务的能力。
2. 通过 loadBalancer 实例根据规则选择出相应的 Server
3. 执行 execute 方法

现在, 我们再看下 execute 方法内容

```
	public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
		...

			T returnVal = request.apply(serviceInstance);
			statsRecorder.recordStats(returnVal);
			return returnVal;

		...
	}
```

注意, 上面的代码我简化掉了, 只留下了我们需要关注的核心逻辑. 实际上就是调用了 request.apply(serviceInstance) 方法, 就是将请求应用于挑选出来的那个服务系统.

### 总体流程

![Ribbon](/image/ribbon-loadbalancer.png)

本文只是简单梳理了 Ribbon 实现负载均衡的核心流程, 但是具体的负载均衡算法没有去深入考究, 也就是 ILoadBalancer 里面的方法实现, 涉及到 IRule, IRule, ServerList, 有兴趣的可以继续深入研究.

