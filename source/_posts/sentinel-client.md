---
title: Redis Sentinel Client
date: 2020-01-20 15:00:36
tags:
---

前几篇文章研究了 Redis 服务端的高可用方案 Sentinel 模式，今天我们来研究一下 Sentinel 的客户端工作原理。

本文以 SpringBoot 和 Jedis 为例，先介绍客户端的使用，然后分析工作流程。在开始阅读文章之前，请大家思考一个问题，Sentinel 模式下，如果主节点发生了切换，客户端是怎么知道的？

### SpringBoot Redis 配置

SpringBoot 中使用 Redis 很简单，只需要引入相应的依赖和配置相关参数即可：

- 依赖引入

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
	<groupId>redis.clients</groupId>
	<artifactId>jedis</artifactId>
</dependency>
```

- 配置

```
spring:
  redis:
	database: 0
	password: ${REDIS_PWD}
	jedis: 
	  pool:
	    max-active: 8  
	    max-wait: -1    
	    max-idle: 8      
	    min-idle: 2       
	sentinel:
	  master: mymaster
	  nodes:
	  - sentinel-0:26379
	  - sentinel-1:26379
	  - sentinel-2:26379
```

上面是一个示例配置，在使用 Sentinel 模式之后，我们并不是直接配置 Redis 服务器主节点的 host 和 port 了，因为主节点是有可能切换的，我们并不能写死一个主服务器的 host 和 port。取而代之，我们配置的是 Sentinel 节点的 host 和 port，所以我们可以大胆猜测在使用 Sentinel 之后客户端的工作流程：

1. 通过任一可用的 Sentinel 节点获取 Redis 主服务器 host 和 port
2. 建立与 Redis 主服务器的连接池
3. 通过连接池获取一个连接与主服务进行读写等操作
4. 如果发生了主服务器切换，重建连接池的连接到新的主服务器

又回到了文章开头的问题，客户端怎么知道主服务器发生了切换呢？以我浅薄的工作经验再次大胆猜测：

1. 不断地去问 Sentinel 主服务器有没有变啊？
2. 要不 Sentinel 直接告诉我？Redis 不是有个发布订阅功能吗。

嗯，上面的工作流程和切换问题都是我瞎猜的，带着问题去看源码是我一贯的作风。

### SpringBoot Redis 启动流程

如果从使用的源头查看，比如 `redisTemplate.opsForValue().set("key1","value1");` ，跟踪其实现方法，我们会发现是通过一个 `RedisConnection` 进行操作。而 RedisConnection 是通过 `RedisConnectionFactory` 获取的，所以 RedisConnectionFactory 的初始化对于我们了解整个流程至关重要。

我们回头看一下，在 SpringBoot 中使用 Redis 只引入了依赖和配置参数，并没有编写任何一行代码，这其实是 SpringBoot 的 AutoConfiguration 帮我们“偷偷”干了活。由于篇幅原因这部分就不展开了，我画了一个大致的关系图：

![SpringBoot AutoConfigruation](/image/springboot-autoconfiguration.png)

由上图可知，`RedisAutoConfiguration` 帮我们实现了 Redis 的自动装配，我们看下 RedisAutoConfiguration 顶部的注解内容：


```
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {
	...
}
```

Import 了另外两个配置类，由于我们的例子引入的是 Jedis 客户端，所以主要是看 JedisConnectionConfiguration 干了那些活。

### JedisConnectionConfiguration

我们再来看一下 `JedisConnectionConfiguration` 的部分代码：

```
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ GenericObjectPool.class, JedisConnection.class, Jedis.class })
class JedisConnectionConfiguration extends RedisConnectionConfiguration {
	@Bean
	@ConditionalOnMissingBean(RedisConnectionFactory.class)
	JedisConnectionFactory redisConnectionFactory(
			ObjectProvider<JedisClientConfigurationBuilderCustomizer> builderCustomizers) throws UnknownHostException {
		return createJedisConnectionFactory(builderCustomizers);
	}

	private JedisConnectionFactory createJedisConnectionFactory(
			ObjectProvider<JedisClientConfigurationBuilderCustomizer> builderCustomizers) {
		JedisClientConfiguration clientConfiguration = getJedisClientConfiguration(builderCustomizers);
		if (getSentinelConfig() != null) {
			return new JedisConnectionFactory(getSentinelConfig(), clientConfiguration);
		}
		...
	}
}
```

看到这里，我们就知道核心的入口了，这里创建了一个 JedisConnectionFactory ，顾名思义，我们需要用到的 Redis 连接就是从这个工厂类获取的。在第二个方法中，有个 `getSentinelConfig()` 方法，一看就知道这是获取我们文章开头的 Sentinel 配置。

所以，JedisConnectionConfiguration 的作用大致就是读取我们的文章开头的配置，然后初始化一个 `JedisConnectionFactory` 的 Bean。

### JedisConnectionFactory

我们再来看一下创建 JedisConnectionFactory 的时候做了哪些事情。

从上面的方法看出，JedisConnectionFactory 是通过 `new JedisConnectionFactory(getSentinelConfig(), clientConfiguration)` 创建的，我们看下这个构造方法的内容：

```
	public JedisConnectionFactory(RedisSentinelConfiguration sentinelConfig, JedisClientConfiguration clientConfig) {

		this(clientConfig);

		Assert.notNull(sentinelConfig, "RedisSentinelConfiguration must not be null!");

		this.configuration = sentinelConfig;
	}

	private JedisConnectionFactory(JedisClientConfiguration clientConfig) {

		Assert.notNull(clientConfig, "JedisClientConfiguration must not be null!");

		this.clientConfiguration = clientConfig;
	}
```



似乎没干啥，只是简单地设置了 clientConfiguration 和 configuration 两个属性。到这里思路好像断了，难道设置两个属性就能干活了吗？前面推论的连接池呢，按理说 JedisConnectionFactory 应该会创建一个连建池，然后获取连接的时候从连接池取一个可用连接出来，但现在并没有看到这样的代码。

此时，只能继续翻看一下 JedisConnectionFactory 还有哪些方法，果然有所发现，有一个属性设置之后的后置方法：

```
public void afterPropertiesSet() {
	...
	if (getUsePool() && !isRedisClusterAware()) {
		this.pool = createPool();
	}
	...
}

private Pool<Jedis> createPool() {

	if (isRedisSentinelAware()) {
		return createRedisSentinelPool((RedisSentinelConfiguration) this.configuration);
	}
	return createRedisPool();
}

protected Pool<Jedis> createRedisSentinelPool(RedisSentinelConfiguration config) {

	GenericObjectPoolConfig poolConfig = getPoolConfig() != null ? getPoolConfig() : new JedisPoolConfig();
	return new JedisSentinelPool(config.getMaster().getName(), convertToJedisSentinelSet(config.getSentinels()),
			poolConfig, getConnectTimeout(), getReadTimeout(), getPassword(), getDatabase(), getClientName());
}
```

可以看出，当我们配置了 Sentinel 信息之后，会创建一个 `JedisSentinelPool`。

### JedisSentinelPool

我们再来看下创建 JedisSentinelPool 的构造方法：

```
 public JedisSentinelPool(String masterName, Set<String> sentinels,
      final GenericObjectPoolConfig poolConfig, final int connectionTimeout, final int soTimeout,
      final String password, final int database, final String clientName) {
    ...
    HostAndPort master = initSentinels(sentinels, masterName);
    initPool(master);
  }
```

到这里，大概就可以证实我们前两点猜想：

1. 通过 Sentinel 获取主节点的 host 和 port
2. 初始化到主节点的连接池

我们分别看下这两个方法是怎么实现的：

```
  private HostAndPort initSentinels(Set<String> sentinels, final String masterName) {

    HostAndPort master = null;
	...
    for (String sentinel : sentinels) {
        ...
        List<String> masterAddr = jedis.sentinelGetMasterAddrByName(masterName);
		...
        if (masterAddr == null || masterAddr.size() != 2) {
          log.warn("Can not get master addr, master name: {}. Sentinel: {}", masterName, hap);
          continue;
        }
        ...
        master = toHostAndPort(masterAddr);
        break;
    }

    ...
    for (String sentinel : sentinels) {
      final HostAndPort hap = HostAndPort.parseString(sentinel);
      MasterListener masterListener = new MasterListener(masterName, hap.getHost(), hap.getPort());
      masterListener.setDaemon(true);
      masterListeners.add(masterListener);
      masterListener.start();
    }

    return master;
  }
```

原来这里不仅仅从可用的 Sentinel 中获取主节点的 host 和 port，还为每个 Sentinel 注册了一个 `MasterListener` 的监听器，看情况就是这里告诉我们主节点变更了。为了不打断主分析流程，我们先不看这个监听器，接着看获取到主节点信息之后怎么初始化连接池：

```
  private void initPool(HostAndPort master) {
    synchronized(initPoolLock){
      if (!master.equals(currentHostMaster)) {
        currentHostMaster = master;
        if (factory == null) {
          factory = new JedisFactory(master.getHost(), master.getPort(), connectionTimeout,
              soTimeout, password, database, clientName);
          initPool(poolConfig, factory);
        } else {
          factory.setHostAndPort(currentHostMaster);
          internalPool.clear();
        }
      }
    }
  }

   public void initPool(final GenericObjectPoolConfig poolConfig, PooledObjectFactory<T> factory) {
    if (this.internalPool != null) {
      try {
        closeInternalPool();
      } catch (Exception e) {
      }
    }
    this.internalPool = new GenericObjectPool<T>(factory, poolConfig);
  }
```

嗯，判断入参的主节点是不是当前主节点，如果不是的话就会初始化连接池，第一次调用该方法时，当前主节点为空也会进行初始化（重写了 equals 方法），连接池使用的是 apache commons pool2。

获取主节点信息和初始化连接池我们已经了解了，接下来看看 initSentinel 时注册的 `MasterListener` 干了啥。

### MasterListener

```
 protected class MasterListener extends Thread {
 	  public void run() {

      running.set(true);

      while (running.get()) {

        j = new Jedis(host, port);
          ...
          List<String> masterAddr = j.sentinelGetMasterAddrByName(masterName);  
          if (masterAddr == null || masterAddr.size() != 2) {
            log.warn("Can not get master addr, master name: {}. Sentinel: {}：{}.",masterName,host,port);
          }else{
              initPool(toHostAndPort(masterAddr)); 
          }

          j.subscribe(new JedisPubSub() {
            @Override
            public void onMessage(String channel, String message) {

              String[] switchMasterMsg = message.split(" ");

              if (switchMasterMsg.length > 3) {

                if (masterName.equals(switchMasterMsg[0])) {
                  initPool(toHostAndPort(Arrays.asList(switchMasterMsg[3], switchMasterMsg[4])));
                } else {
				  ...
                }

              } else {
              	 ...
              }
            }
          }, "+switch-master");

		...     
      }
    }
 }
```

到这里，文章开头的问题终于真相大白，原来是有单独的线程监听每个 Sentinel 的 `+switch-master` 主题，当 Sentinel 发生主节点切换时会通过 `+switch-master` 主题发布消息，然后 `MasterListener` 线程监听到之后重新初始化连接池。

### 总结 

我们回顾一下整个流程

1. SpringBoot 通过 AutoConfiguration 自动装配创建 JedisConnectionFactory
2. JedisConnectionFactory 根据配置信息创建 JedisSentinelPool
3. JedisSentinelPool 通过 Sentinel 获取主节点信息，然后初始化连接池
4. JedisSentinelPool 在 initSentinel 时除了获取主节点信息，还会为每个 Sentinel 创建一个 MasterListener 监听器
5. MasterListener 通过监听 Sentinel 的 `+switch-master` 主题观测主节点的切换，然后创建连接池




