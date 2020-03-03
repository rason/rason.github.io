---
title: sleuth zipkin kafka elasticsearch 链路追踪
date: 2019-06-18 10:05:59
tags: sleuth zipkin kafka elasticsearch
---

在一个大型的分布式系统中, 链路追踪是比不可少的, 否则排查问题将会是一个灾难. 关于链路追踪的相关介绍就不展开讨论了, 可以参考下面的链接, 本文着重介绍实际操作.

- [Dapper](https://ai.google/research/pubs/pub36356)
- [zipkin](https://zipkin.io/)
- [Spring Cloud Sleuth](https://cloud.spring.io/spring-cloud-sleuth/spring-cloud-sleuth.html)

#### kafka 安装

参考官方的 [Quick Start](https://kafka.apache.org/quickstart)

#### elasticsearch 安装

1. 下载 `wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.1.1-linux-x86_64.tar.gz`
2. 解压 `tar -zxf elasticsearch-7.1.1-linux-x86_64.tar.gz`
3. 启动 `./bin/elasticsearch`
4. 检查 `curl localhost:9200`

如果以 root 用户启动会报错

`org.elasticsearch.bootstrap.StartupException: java.lang.RuntimeException: can not run elasticsearch as root`


因此我们可以创建一个非 root 用户来启动


```
groupadd es 
useradd es -g es -p es
chown -R es:es ${elasticsearch_HOME}/
sudo su es
./bin/elasticsearch
```

#### zipkin 服务

1. 下载 `curl -sSL https://zipkin.io/quickstart.sh | bash -s`
2. 启动 `STORAGE_TYPE=elasticsearch ES_HOSTS=http://127.0.0.1:9200 KAFKA_BOOTSTRAP_SERVERS=127.0.0.1:9092 java -jar zipkin.jar &`
3. 检查 在浏览器中打开 `http://127.0.0.1:9411`

![zipkin](/image/zipkin.png)

#### spring boot 微服务配置

1. pom.xml 需要加入以下依赖

```
<dependencyManagement> 
      <dependencies>
          <dependency>
              <groupId>org.springframework.cloud</groupId>
              <artifactId>spring-cloud-dependencies</artifactId>
              <version>${release.train.version}</version>
              <type>pom</type>
              <scope>import</scope>
          </dependency>
      </dependencies>
</dependencyManagement>

<dependency> 
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>

 <dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

2. application.yml 配置

```
spring:
  sleuth:
    sampler:
      probability: 1.0
  zipkin:
    sender:
      type: kafka
  kafka:
    bootstrap-servers: 127.0.0.1:9092
```

3. logback 配置

在日志的 pattern 中加入 `[%X{X-B3-TraceId:-},%X{X-B3-SpanId:-}]` 就可以在日志中将 traceId 和 spanId 打印出来了.

#### 验证

启动两个微服务, 然后发起一个服务之间调用的请求. 

1. 日志的输出应该有类似以下的内容:

```
2019-06-18 10:49:46,797 [304b31da6dbe1609,304b31da6dbe1609] 
```

2. 根据 traceId 在 zipkin 中搜索

![zipkin-search](/image/zipkin-search.png)

3. 检查内容是不是真的保存在 elasticsearch 中

```
curl -H "Content-Type: application/json" 'localhost:9200/_search' -d '{"query":{"match":{"traceId":"304b31da6dbe1609"}}}'
```




