---
title: ELK日志监控系统
date: 2017-08-17 15:33:59
categories: BigData
tags: ELK
description: ELK
---

毫无疑问，日志对于开发者来说相当重要，通过日志可以追踪系统出现的问题和运行情况。

目前的工作依旧还是直接SSH登陆服务器，然后使用vim来查看日志，我觉得很麻烦，而且只能进行简单的问题跟踪，对日志数据的进一步分析相当困难。

通过搜索了解到ELK是比较流行的日志监控系统，所以就尝试着使用起来。

## ELK stack

ELK stack ：

- **L**ogstash: 作为数据的收集引擎，从不同的地方收集数据并进行相应的处理，然后输送到存储引擎，比如ElasticSearch。

- **E**lasticSearch： 提供数据的存储和查询。ES提供全文搜索和分析，底层给予Lucene。默认就是分布式，易扩展。

- **K**ibana： 数据可视化工具，可以对ElasticSearch中的数据进行查询可视化。还可以进行分析和生成一些图表。

## 作用

从上面的介绍可知，使用ELK，我们可以通过一个Web界面来搜索分布式系统的日志数据。但更重要的是，我们还可以通过对数据的处理来对系统进行监控和调查分析。比如：

- 统计某个类型的错误出现的次数

- 统计某个类型的事件数，以确定受问题影响的用户数量

- 升级系统之后产生的影响

等等。


## Logstash

Logstash 作为中心的控制器和日志数据路由。

它可以从不同的数据源（使用**input**）对数据进行收集，然后使用**filters**对数据进行格式化等处理，最后数据流向各种输出端点（使用**output**）。如下图所示：

![Logstash](https://raw.githubusercontent.com/rason/rason.github.io/master/image/logstash.png)

因此，使用Logstash其实非常简单，就是通过一个配置文件（比如logstash.conf）来对数据管道中的三个部分（**input - filter - output**）进行配置即可。

### input

Logstash 可以通过不同的外部插件来从众多的数据源中接收数据，一些通用的比如“file”,"tcp/udp"，或者是一些其它的比如Kafka topic等。下面就是一个**input**配置的例子：

```
input {  
  tcp {
   port => 4560
   codec => json_lines
  }
}
```

通过上面的配置，应用系统就可以通过TCP连接Logstash服务器4560端口发送json格式的数据到Logstash中了。

<!-- more -->

### filter

Filters 可以对每条日志信息进行实时处理。比如将非结构化的数据进行结构化等。举个例子，下面是一条日志信息：

```
"55.3.244.1 GET /index.html 15824 0.043"
```

配置filter

```
filter {  
  mutate {
    remove_field => [ "@version" ]
  }

  if [type] == "nginx-access" {
    grok {
      match => { "message" => "%{IP:clientip} %{WORD:method} %{URIPATHPARAM:request}
                 %{NUMBER:bytes} %{NUMBER:duration}" }
    }
    geoip {
      source => "clientip"
      target => "geoip"
      database => "/etc/logstash/GeoLiteCity.dat"
      add_field => [ "[geoip][coordinates]", "%{[geoip][longitude]}" ]
      add_field => [ "[geoip][coordinates]", "%{[geoip][latitude]}"  ]
    }
    mutate {
      convert => [ "[geoip][coordinates]", "float"]
    }
  }
}
```

这里的grok，geoip就是一些filter处理插件，Logstash支持很多内置或者外部的插件对日志数据进行实时处理，这些就不展开了。

## output

指定Logstash 处理后的数据输出地点。同样也支持不同的插件将数据输送到不同的地方。通常我们会使用“elasticsearch”插件，但是也经常会输送到标准输出（stdout）或者HadoopFS hdfs等地方。


```
output {  
    if ([environment] == "qa" {
        stdout {}
    } else {
        elasticsearch {
            hosts => [ "elasticsearch-host:port" ]
            index => "elk-%{+YYYY.MM.dd}"
            document_type => "log"
        }
    }
}
```

## 将日志数据推送到Logstash中

有多种方式可以将日志数据推送到Logstash中：

- 一种方式是通过**外部日志传输系统**，比如说[Beats](https://github.com/elastic/beats)。

- 使用Docker容器运行应用程序的可以利用Docker“日志驱动程序”（[FluentD Docker Logging](https://www.fluentd.org/guides/recipes/docker-logging)）的功能捕获stdout输出——[EFK stach](https://geowarin.github.io/spring-boot-logs-in-elastic-search.html)。你可能还需要看一下[Logspout](https://github.com/gliderlabs/logspout)。

- 通过自定义的[Logback Appender](https://github.com/logstash/logstash-logback-encoder) 直接在应用中将日志信息传输到Logstash中。Java 日志中一个比较常用的日志方案是slf4j+logback，因此在网上找了一个开源项目[logstash-logback-encoder](https://github.com/logstash/logstash-logback-encoder)来提供自定义的Logback Appender。

## 使用logstash-logback-encoder

在我们使用自定义的Logback Appender之前，可能会有一些疑问，比如说“当我们将日志数据传输到Logstash中时，Logstash服务器挂了会怎么样？”，“日志传输的调用会不会阻塞？”，“会不会对系统有性能的影响？”答案是否定的，因为数据是通过异步的方式传输到Logstash中的。

[LogstashTcpSocketAppender](https://github.com/logstash/logstash-logback-encoder/blob/master/src/main/java/net/logstash/logback/appender/AbstractLogstashTcpSocketAppender.java) 实际上将数据写到[LMAX RingBuffer](https://github.com/logstash/logstash-logback-encoder/blob/master/src/main/java/net/logstash/logback/appender/AsyncDisruptorAppender.java)，RingBuffer 被独立的线程消费将数据推送到Logstash中。所以，即使Logstash服务挂了也不会阻塞我们的应用代码。


下面是一个logback.xml的配置例子：

```
<appender name="stash" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
	<destination>10.16.2.30:4560</destination>

	<encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
		<providers>
			<mdc/> <!-- MDC variables on the Thread will be written as JSON fields-->
			<context/> <!--Outputs entries from logback's context -->
			<version/> <!-- Logstash json format version, the @version field in the output-->
			<logLevel/>
			<loggerName/>

			<pattern>
				<pattern>
					<!-- we can add some custom fields to be sent with all the log entries.      -->
					<!--make filtering easier in Logstash-->
					<!--or searching with Kibana-->
                    {
					"appName": "qhee-webapp-net",
					"appVersion": "1.0"
					}
				</pattern>
			</pattern>

			<threadName/>
			<message/>

			<logstashMarkers/> <!-- Useful so we can add extra information for specific log lines as Markers-->
			<arguments/> <!--or through StructuredArguments-->

			<stackTrace/>
		</providers>
	</encoder>
</appender>

<root level="INFO">
	<appender-ref ref="stash" />
</root>
```

更多关于logstash-logback-encoder的详细内容可以见其[Github](https://github.com/logstash/logstash-logback-encoder)。

## ElasticSearch

ElasticSearch 可能是ELK stack 中最关键的因素，它充当数据库部分。关于其详细用法由于篇幅关系就不展开了，这里就直接从实际应用出发，修改其配置文件`elasticsearch.yml`：

```
network.host: 0.0.0.0
http.port: 9200

```

配置其host和port监听请求，注意这个端口就是Logstash output 中配置的elasticsearch端口，这样Logstash中的日志数据就可以传输到ElasticSearch中了。

然后就可以通过`bin/elasticsearch` 命令启动ElasticSearch，启动过程中可能会遇到一些问题，比如：

```
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
[2]: max number of threads [1024] for user [laixs] is too low, increase to at least [2048]
[3]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[4]: system call filters failed to install; check the logs and fix your configuration or disable system call filters at your own risk
```

问题的描述应该很清晰，按照提示修改其相应的系统配置既可。

## Kibana

Kibana 是一个可视化工具，使用起来也是相当的简单，主要就是配置读取的ElasticSearch 地址，修改其配置文件`kibana.yml`：

```
server.host: "0.0.0.0"
elasticsearch.url: "http://elasticsearch-host:9200"

```

然后通过`bin/kibana`命令即可启动Kibana。

![Kibana](https://raw.githubusercontent.com/rason/rason.github.io/master/image/kibana.png)

## 总结

本文只是简单地介绍了一下ELK和Java 日志中如何通过自定义的Logback Appender将日志数据发送到Logstash中。
