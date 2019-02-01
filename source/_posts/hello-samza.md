---
title: Samza实时日志分析
date: 2017-08-02 14:09:37
categories: BigData
tags: Samza
description: Samza实时日志分析
---

## 前言

前段时间初步研究了一下基于HDFS的离线日志分析，相应地自然会继续学习一下实时日志分析。实时分析最火的框架显然是Storm，但是在对比时发现Samza，据说跟Kafka结合的比较好，于是就先用Samza。以后有时间再研究Storm。

不过经过一番折腾，发现Samza的资料实在是太少了，期间遇到不少问题，断断续续花了几天的时间将官网的文档大概啃了一下。这故事告诉我们在技术选型的时候还是得选成熟，多用人的框架好，因为很多坑都被别人踩过一遍了。

## Samza入门

入门一个新框架的很好方法就是按照官网的示例走一遍。这里就不详细说了，可以见官网的[示例](http://samza.apache.org/startup/hello-samza/0.13/)。

## 使用Samza统计实时PV

下面将介绍自己模仿官网示例做的一个小功能：统计公司官网一级菜单的实时PV。

### 基础环境

Samza通常要用到三个系统：**YARN,Kafka,ZooKeeper**。

Samza作业运行在YARN上，Kafka用于收集官网的实时Nginx访问日志，ZooKeeper用于协调Kafka集群。

这些环境的搭建都在前面的文章中介绍，本次的小功能也运行在之前搭建的环境上。

### 整体运行流程

1. 使用Flume采集Nginx日志信息，并且Sink到Kafka集群中的`qhee-net-access-log` Topic中。
2. 定义一个继承于`StreamApplication`的Samza应用过滤`qhee-net-access-log` Topic中的无用日志信息，只留下一级菜单的访问日志，并输出到Kafka集群中的`nginx-access-log-output` Topic中。
3. 定义另一个继承于`StreamApplication`的Samza应用读取`nginx-access-log-output` Topic中的消息，并统计某个时间段内的PV，然后输出到`qhee-pageview-output` Topic中。

### 具体实现

- Flume配置

```
kafkaagent.sources = execSrc
kafkaagent.channels = memoryChannel
kafkaagent.sinks = kafkaSink

kafkaagent.sources.execSrc.type = exec
kafkaagent.sources.execSrc.command = tail -F /usr/local/nginx/logs/www.qhee.net.access.log
kafkaagent.sources.execSrc.channels = memoryChannel

kafkaagent.sinks.kafkaSink.type = org.apache.flume.sink.kafka.KafkaSink
kafkaagent.sinks.kafkaSink.kafka.topic = qhee-net-access-log
kafkaagent.sinks.kafkaSink.kafka.bootstrap.servers = kafka1:9092,kafka2:9092,kafka3:9092
kafkaagent.sinks.kafkaSink.kafka.linger.ms = 1
kafkaagent.sinks.kafkaSink.kafka.flumeBatchSize = 20
kafkaagent.sinks.kafkaSink.channel = memoryChannel

kafkaagent.channels.memoryChannel.type = memory
kafkaagent.channels.memoryChannel.capacity = 1000
kafkaagent.channels.memoryChannel.transactionCapacity = 100
```

<!-- more -->

- 用于过滤的Samza应用

```
public class NginxAccessLogApp implements StreamApplication {

    private static final Logger LOG = LoggerFactory.getLogger(NginxAccessLogApp.class);
    private static final String INPUT_TOPIC = "qhee-net-access-log";
    private static final String OUTPUT_TOPIC = "nginx-access-log-output";
    private static final Set<String> FILTER_KEY = new HashSet<String>(16);

    static {
        FILTER_KEY.add("/");
        FILTER_KEY.add("/node/service-chain/listed");
        FILTER_KEY.add("/node/listed-company/");
        FILTER_KEY.add("/node/system/");
        FILTER_KEY.add("/node/today/pub");
        FILTER_KEY.add("/node/about-us/");
        FILTER_KEY.add("/node/user/reg");
    }

    @Override
    public void init(StreamGraph graph, Config config) {
        MessageStream<String> accessLogs = graph.<String, String, String>getInputStream(INPUT_TOPIC, (k, v) -> v);

        Function<String, String> keyFn = accessLog -> new NginxAccessLog(accessLog).getRemote_addr();

        OutputStream<String, String, String> outputStream = graph
                .getOutputStream(OUTPUT_TOPIC, keyFn, m -> m);

        FilterFunction<String> filterFn = accessLog -> FILTER_KEY.contains(new NginxAccessLog(accessLog).getRequest_uri());

        accessLogs
                .partitionBy(keyFn)
                .filter(filterFn)
                .sendTo(outputStream);
    }


}
```

- 用于过滤的Samza应用配置信息nginx-access-log.properties

```
# Job
job.factory.class=org.apache.samza.job.yarn.YarnJobFactory
job.name=nginx-access-log

# YARN
yarn.package.path=hdfs://10.16.1.24:8020/samza/qhee-net/${project.artifactId}-${pom.version}-dist.tar.gz

# Task
app.class=samza.examples.qhee.NginxAccessLogApp
task.inputs=kafka.qhee-net-access-log

# Serializers
serializers.registry.json.class=org.apache.samza.serializers.JsonSerdeFactory
serializers.registry.string.class=org.apache.samza.serializers.StringSerdeFactory

# Kafka System
systems.kafka.samza.factory=org.apache.samza.system.kafka.KafkaSystemFactory
systems.kafka.samza.msg.serde=string
systems.kafka.samza.key.serde=string
systems.kafka.consumer.zookeeper.connect=10.16.1.24:2181,10.16.1.25:2181,10.16.2.28:2181
systems.kafka.producer.bootstrap.servers=kafka1:9092,kafka2:9092,kafka3:9092
systems.kafka.default.stream.replication.factor=1

# Job Coordinator
job.coordinator.system=kafka
job.coordinator.replication.factor=1

job.default.system=kafka
job.container.count=2
```

- 用于统计PV的Samza应用

```
public class QheePageViewApp implements StreamApplication {

  private static final Logger LOG = LoggerFactory.getLogger(QheePageViewApp.class);
  private static final String INPUT_TOPIC = "nginx-access-log-output";
  private static final String OUTPUT_TOPIC = "qhee-pageview-output";

  @Override
  public void init(StreamGraph graph, Config config) {

    MessageStream<String> accessLogs = graph.<String, String, String>getInputStream(INPUT_TOPIC, (k, v) -> v);

    OutputStream<String, String, WindowPane<String, Integer>> outputStream = graph
        .getOutputStream(OUTPUT_TOPIC, m -> m.getKey().getKey(), m -> m.getMessage().toString());

    Function<String, String> keyFn = accessLog -> new NginxAccessLog(accessLog).getRequest_uri();

    accessLogs
        .partitionBy(keyFn)
        .window(Windows.keyedTumblingWindow(keyFn, Duration.ofSeconds(60), () -> 0, (m, prevCount) -> prevCount + 1))
        .sendTo(outputStream);
  }
}
```

- 用于统计PV的Samza应用的配置信息

```
# Job
job.factory.class=org.apache.samza.job.yarn.YarnJobFactory
job.name=qhee-pageview

# YARN
yarn.package.path=hdfs://10.16.1.24:8020/samza/qhee-net/${project.artifactId}-${pom.version}-dist.tar.gz

# Task
app.class=samza.examples.qhee.QheePageViewApp
task.inputs=kafka.nginx-access-log-output

# Serializers
serializers.registry.json.class=org.apache.samza.serializers.JsonSerdeFactory
serializers.registry.string.class=org.apache.samza.serializers.StringSerdeFactory

# Kafka System
systems.kafka.samza.factory=org.apache.samza.system.kafka.KafkaSystemFactory
systems.kafka.samza.msg.serde=string
systems.kafka.samza.key.serde=string
systems.kafka.consumer.zookeeper.connect=10.16.1.24:2181,10.16.1.25:2181,10.16.2.28:2181
systems.kafka.producer.bootstrap.servers=kafka1:9092,kafka2:9092,kafka3:9092
systems.kafka.default.stream.replication.factor=1

# Job Coordinator
job.coordinator.system=kafka
job.coordinator.replication.factor=1

job.default.system=kafka
job.container.count=1
```

## 打包部署

1. 先本地mvn打包
2. 上传到YARN ResourceManager节点，并上传一份到配置文件中yarn.package.path的hdfs路径中
3. 解压YARN ResourceManager节点上的包，然后run-app.sh将两个应用运行起来
4. 访问页面测试，观察Kafka Topic的输出

## 遇到的问题

在这过程中主要遇到了三个问题：

- 运行应用是无法连接到YARN ResourceManager，原因是因为没有配置`HADOOP_CONF_DIR`环境变量，导致作业没有读取到ResourceManager的Host，使用默认的Host无法连接。

- 成功运行应用一会之后因为虚拟内存益处导致应用被杀死，原因是因为虚拟内存不够用，需要配置`yarn-site.xml`的`yarn.nodemanager.vmem-pmem-ratio`属性设置一个合适的虚拟内存-物理内存比例。

- 成功运行应用，也没内存益处，但是Kafka中就是没有消息输出，原因是自己手贱将运行应用的命令输错了，将其作为一个作业运行起来了，即将`run-app.sh`搞成了`run-job.sh`。

## 总结

- 初步入门了Samza
- 因为几个问题折腾了不少时间，但也学到了一点东西

