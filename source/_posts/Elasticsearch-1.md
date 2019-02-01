---
title: Elasticsearch基础入门 1
date: 2017-08-24 10:52:45
categories: BigData
tags: Elasticsearch
description: Elasticsearch基础入门
---

## 前言

之前使用ELK将日志系统初步搭建了起来，但是对于Elasticsearch的查询语法还是有点陌生，所以就要针对性地学习一下。

## 与Elasticsearch 进行交互

Elasticsearch 的安装和运行之前已经介绍过了，现在就直接学习怎么通过HTTP和Elasticsearch 进行交互。

Elasticsearch 请求格式如下：

```
curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'
```
| 部件 | 解析 |
| ---------| --------- |
| VERB | 适当的HTTP方法：GET,POST,PUT,HEAD或者DELETE |
| PROTOCOL | HTTP或者HTTPS |
| HOST | Elasticsearch 集群中任意节点的主机名 |
| PORT | 运行 Elasticsearch HTTP 服务的端口号，默认是 9200 |
| PATH | API 的终端路径（例如 _count 将返回集群中文档数量）。Path 可能包含多个组件，例如：_cluster/stats 和 _nodes/stats/jvm  |
| QUERY_STRING | 任意可选的查询字符串参数 (例如 ?pretty 将格式化地输出 JSON 返回值，使其更容易阅读) |
| BODY | 一个 JSON 格式的请求体 (如果请求需要的话) |

例如，计算集群中文档的数量：

```
curl -XGET 'http://localhost:9200/_count?pretty' -d '
{
    "query": {
        "match_all": {}
    }
}
'
```

返回结果：

```
{
    "count" : 0,
    "_shards" : {
        "total" : 5,
        "successful" : 5,
        "failed" : 0
    }
}
```

在后面的内容中，我们将用缩写格式来展示这些 curl 示例，所谓的缩写格式就是省略请求中所有相同的部分，例如主机名、端口号以及 curl 命令本身。

<!-- more -->

## 实际案例

Elasticsearch 使用的是JSON格式将数据存储为文档，与平时我们使用的关系型数据库有点不一样，但是我们可以类比地学习。


### 创建一个雇员

假设我们需要保存一个公司的雇员信息，通过下面的请求即可保存一个雇员信息：

```
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```

返回结果：

```
 {"_index":"megacorp","_type":"employee","_id":"1","_version":1,"result":"created","_shards":{"total":2,"successful":1,"failed":0},"created":true}
```

 从返回结果可以看出：

 - 索引名：megacorp
 - 索引类型：employee
 - 特定雇员ID：1

存储数据到Elasticsearch 的行为叫做**索引**，所以上面的索引名，索引类型我们可以类比为数据库的数据库名和表名。 

为了演示后面的查询，我们再添加多两个雇员：

```
PUT /megacorp/employee/2
{
    "first_name" :  "Jane",
    "last_name" :   "Smith",
    "age" :         32,
    "about" :       "I like to collect rock albums",
    "interests":  [ "music" ]
}

PUT /megacorp/employee/3
{
    "first_name" :  "Douglas",
    "last_name" :   "Fir",
    "age" :         38,
    "about":        "I like to build cabinets",
    "interests":  [ "forestry" ]
}
```

### 检索文档

目前数据已经保存进去了，现在我们可以试一下简单的查询，比如根据ID来获取某个雇员的信息：

```
GET /megacorp/employee/1
```

返回结果不但包含了雇员的信息还包含了一些元数据：

```
{
  "_index" :   "megacorp",
  "_type" :    "employee",
  "_id" :      "1",
  "_version" : 1,
  "found" :    true,
  "_source" :  {
      "first_name" :  "John",
      "last_name" :   "Smith",
      "age" :         25,
      "about" :       "I love to go rock climbing",
      "interests":  [ "sports", "music" ]
  }
}
```

将 HTTP 命令由 PUT 改为 GET 可以用来检索文档，同样的，可以使用 DELETE 命令来删除文档，以及使用 HEAD 指令来检查文档是否存在。如果想更新已存在的文档，只需再次 PUT 。

### 轻量搜索

搜索所有雇员：

```
GET /megacorp/employee/_search
```

可以看到，我们仍然使用索引库 `megacorp` 以及类型 `employee`，但与指定一个文档 ID 不同，这次使用 `_search` 。返回结果包括了所有三个文档，放在数组 hits 中。一个搜索默认返回十条结果。

```
{
  "took" : 3,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 3,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "first_name" : "Douglas",
          "last_name" : "Fir",
          "age" : 38,
          "about" : "I like to build cabinets",
          "interests" : [
            "forestry"
          ]
        }
      }
    ]
  }
}
```

搜索姓氏为`Smith`的雇员：

```
GET /megacorp/employee/_search?q=last_name:Smith
```

返回结果如下：

```
{
  "took" : 4,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "2",
        "_score" : 0.2876821,
        "_source" : {
          "first_name" : "Jane",
          "last_name" : "Smith",
          "age" : 32,
          "about" : "I like to collect rock albums",
          "interests" : [
            "music"
          ]
        }
      },
      {
        "_index" : "megacorp",
        "_type" : "employee",
        "_id" : "1",
        "_score" : 0.2876821,
        "_source" : {
          "first_name" : "John",
          "last_name" : "Smith",
          "age" : 25,
          "about" : "I love to go rock climbing",
          "interests" : [
            "sports",
            "music"
          ]
        }
      }
    ]
  }
}
```

## 总结

这篇文章并没有过多地讲解语法，撸起衣袖就是干，让我们初步熟悉Elasticsearch 的使用，下一篇文章中将会学习更加复杂的搜索。
