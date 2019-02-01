---
title: 使用Pig结构化日志然后导入到Hive数据仓库
date: 2017-07-07 13:20:19
categories: BigData
tags: Pig
description: Pig structure format and import to hive dw
---

## 前言

接着上篇文章，我们已经将日志通过Flume收集到了HDFS中，那么接下来就是使用Pig将日志内容结构化，然后保存到Hive数据仓库中。

## Pig安装

1.下载最近稳定版的Pig,[点这里](http://hadoop.apache.org/pig/releases.html).

2.解压，修改`/etc/profile`文件配置环境变量`$ export PATH=/<my-path-to-pig>/pig-n.n.n/bin:$PATH`

3.`$ source /etc/profile`使环境变量生效

4.测试安装是否成功`$ pig -help`

## 运行Pig

本例子接着之前的教程，前提需要已经部署好的Hadoop并且配置好其环境变量。

1.运行Pig

```
$ pig
```

2.进入HDFS文件系统根目录

```
grunt> cd hdfs://10.16.1.24/
```

该目录是我前面教程中搭建的HDFS文件系统根目录。

3.查看根目录有哪些文件

```
grunt> ls
hdfs://10.16.1.24/flume	<dir>
hdfs://10.16.1.24/output	<dir>
hdfs://10.16.1.24/test	<dir>
hdfs://10.16.1.24/tmp	<dir>
hdfs://10.16.1.24/user	<dir>
```

上篇文章中通过Flume收集的日志在`/flume`文件夹中。

4.结构化日志文件

```
grunt> content = LOAD '/flume/qhee/net/*' USING PigStorage('\t') AS(domain:chararray,host:chararray,user:chararray,time:chararray,method:chararray,path:chararray,protocol:chararray,status:chararray,size:chararray,refer:chararray,agent:chararray,response_time:chararray,cookie:chararray,set_cookie:chararray,upstream_addr:chararray,upstream_cache_status:chararray,upstream_reponse_time:chararray);

grunt> DUMP content;

grunt> STORE content INTO 'output' USING PigStorage (',');

```

相关命令解析就不展开了，可以自行查看官方文档。

结构化之后的日志保存到了`output`目录中，所以上面的HDFS根目录中会有`output`这个文件夹，因为我之前已经执行过了。

可以将结构化的文件下载下来看看结果，发现其实就是将每个属性用逗号隔开了而已。做这项工作的主要目的就是为了方便导入到Hive仓库中，后面将会看到我们会以逗号为分隔符导入。

接下来该轮到Hive登场了。

<!-- more -->

## 安装Hive

1.下载解压[Hive](https://hive.apache.org/downloads.html)

2.类似Pig配置环境变量`$ export PATH=$HIVE_HOME/bin:$PATH`

3.创建`$HIVE_HOME/conf/hive-site.xml`文件，内容如下

```
<configuration>

<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://10.16.2.28:3306/metastore?createDatabaseIfNotExist=true</value>
</property>


<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>
</property>

<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>root</value>
</property>

<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>123456</value>
</property>

<property>
  <name>datanucleus.autoCreateSchema</name>
  <value>true</value>
</property>

<property>
  <name>datanucleus.fixedDatastore</name>
  <value>true</value>
</property>

<property>
 <name>datanucleus.autoCreateTables</name>
 <value>True</value>
 </property>

</configuration>

```

此文件的作用是配置hive元数据保存的地方。

## 运行Hive

1.运行Hive

```
$ $HIVE_HOME/bin/hive
```

这个操作可能会遇到一下异常，可以通过`hive -hiveconf hive.root.logger=DEBUG,console`将日志输出到控制台方便查看。

我在启动过程中遇到了些小问题，是由于保存元数据的metastore数据库某些表创建失败导致，可以自己手动创建。

```
cd $HIVE_HOME/scripts/metastore/upgrade/mysql/

< Login into MySQL >

mysql> drop database IF EXISTS metastore;
mysql> create database metastore;
mysql> use metastore;
mysql> source hive-schema-2.1.0.mysql.sql;
```

2.创建元数据数据库

```
hive> create database qhee_net;
hive> use qhee_net;
```

3.创建表

```
hive> Create table access_log
    > (
    >  host String,
    >  remote_addr String,
    >  remote_user String,
    >  time_local String,
    >  request_method String,
    >  request_uri String,
    >  server_protocol String,
    >  status String,
    >  body_bytes_sent String,
    >  http_referer String,
    >  http_user_agent String,
    >  request_time String,
    >  http_cookie String,
    >  sent_http_set_cookie String,
    >  upstream_addr String,
    >  upstream_cache_status String,
    >  upstream_response_time String                       
    > )
    > ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

```

重点是最后一行，指定了逗号分隔符（Pig结构化时所做的工作），这样就可以将数据对应导入。

4.加载数据

```
hive> LOAD DATA INPATH 'hdfs://10.16.1.24/output/part-m-00000' OVERWRITE INTO TABLE access_log;
Loading data to table qhee_net.access_log
OK
Time taken: 1.251 seconds
```

5.测试数据查询

```
hive> select remote_addr from access_log;
OK
10.8.30.19
10.8.30.19
10.8.30.19
10.8.30.19
10.8.30.19
10.8.30.19
10.8.30.19
...

```

## 总结

本文简单地演示了使用Pig对数据进行结构化，然后导入到Hive数据仓库中，并没有深入更多的细节，仅起到领入门的作用。
