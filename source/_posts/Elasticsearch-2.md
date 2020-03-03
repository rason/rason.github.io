---
title: Elasticsearch基础入门 2
date: 2017-08-24 14:00:28
categories: BigData
tags: Elasticsearch
description: Elasticsearch基础入门
---

## 前言

接着上一篇文章来学习稍微复杂一点的查询。

## 使用查询表达式搜索

Elasticsearch 提供一个丰富灵活的查询语言叫做`查询表达式 `，它支持构建更加复杂和健壮的查询。

*领域特定语言（DSL）*，指定了使用一个 JSON 请求。我们可以像这样重写之前的查询所有 Smith 的搜索：

```
GET /megacorp/employee/_search
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
```

这个请求使用 JSON 构造，并使用了一个 match 查询（属于查询类型之一，后续将会了解）。

## 更复杂的搜索

搜索姓氏为Smith 的雇员，并且年龄大于30的：

```
GET /megacorp/employee/_search
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
```

目前无需太多担心语法问题，后续会更详细地介绍。只需明确我们添加了一个 过滤器 用于执行一个范围查询，并复用之前的 match 查询。由于现在的查询请求JSON 太长了，直接在终端使用curl 输入将十分麻烦，所以我们可以使用Kibana 的 Console来进行查询。如下图所示：

![复杂查询](/image/elasticsearch-search.png)

现在结果只返回了一个雇员，叫 Jane Smith，32 岁。

<!-- more -->

## 全文搜索

现在来尝试一下稍微高级一点的全文搜索——一项传统数据库很难搞定的任务。

搜索一下所有喜欢攀岩(rock climbing) 的雇员：

![全文搜索](/image/elasticsearch-complex-search.png)

返回结果中的`_score` 属性表示相关性得分。

Elasticsearch 默认按照相关性得分排序，即每个文档跟查询的匹配程度。第一个最高得分的结果很明显：John Smith 的 about 属性清楚地写着 “rock climbing” 。

但为什么 Jane Smith 也作为结果返回了呢？原因是她的 about 属性里提到了 “rock” 。因为只有 “rock” 而没有 “climbing” ，所以她的相关性得分低于 John 的。

Elasticsearch中的 相关性 概念非常重要，也是完全区别于传统关系型数据库的一个概念，数据库中的一条记录要么匹配要么不匹配。

## 短语搜索

上面的情况，如果我不想 Jane Smith 也被搜索出来该怎么办？

很简单，只需对`match`查询稍作调整，使用一个叫做`match_phrase`的查询：

![短语搜索](/image/elasticsearch-phrase-search.png)

这样，就只匹配含有“rock climbing” 这个短语的雇员。

## 高亮搜索

在实际的应用中，我们希望在搜索结果列表中高亮搜索关键字，使用Elasticsearch也很容易做到，只需增加一个新的`highlight` 参数：

![高亮搜索](/image/elasticsearch-highlight.png)

当执行该查询时，返回结果与之前一样，与此同时结果中还多了一个叫做 highlight 的部分。

## 分析

Elasticsearch 有一个功能叫聚合（aggregations），允许我们基于数据生成一些精细的分析结果。聚合与 SQL 中的 GROUP BY 类似但更强大。

比如，挖掘出雇员中最受欢迎的兴趣爱好：

```
GET /megacorp/employee/_search
{
  "aggs": {
    "all_interests": {
      "terms": { "field": "interests" }
    }
  }
}
```

上述的搜索可能会报出以下的错误：

```
"Fielddata is disabled on text fields by default. Set fielddata=true on [interests] in order to load fielddata in memory by uninverting the inverted index.
```

按照提示，需要将interests 属性的fielddata 设置为true：

![设置fielddata](/image/elasticsearch-mapping.png)

设置完之后再次运行上面的查询语句，返回结果：

```
{
   ...
   "hits": { ... },
   "aggregations": {
      "all_interests": {
         "buckets": [
            {
               "key":       "music",
               "doc_count": 2
            },
            {
               "key":       "forestry",
               "doc_count": 1
            },
            {
               "key":       "sports",
               "doc_count": 1
            }
         ]
      }
   }
}
```

可以看到，两位员工对音乐感兴趣，一位对林地感兴趣，一位对运动感兴趣。这些聚合并非预先统计，而是从匹配当前查询的文档中即时生成。如果想知道叫 Smith 的雇员中最受欢迎的兴趣爱好，可以直接添加适当的查询来组合查询：

```
GET /megacorp/employee/_search
{
  "query": {
    "match": {
      "last_name": "smith"
    }
  },
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests"
      }
    }
  }
}
```

聚合还支持分级汇总 。比如，查询特定兴趣爱好员工的平均年龄：

```
GET /megacorp/employee/_search
{
    "aggs" : {
        "all_interests" : {
            "terms" : { "field" : "interests" },
            "aggs" : {
                "avg_age" : {
                    "avg" : { "field" : "age" }
                }
            }
        }
    }
}
```

得到的聚合结果有点复杂，但理解起来还是很简单的：

```
...
  "all_interests": {
     "buckets": [
        {
           "key": "music",
           "doc_count": 2,
           "avg_age": {
              "value": 28.5
           }
        },
        {
           "key": "forestry",
           "doc_count": 1,
           "avg_age": {
              "value": 38
           }
        },
        {
           "key": "sports",
           "doc_count": 1,
           "avg_age": {
              "value": 25
           }
        }
     ]
  }
```

## 总结

这是Elasticsearch 基础教程，用于引导入门，更多诸如 suggestions、geolocation、percolation、fuzzy 与 partial matching 等特性没并有提及。

