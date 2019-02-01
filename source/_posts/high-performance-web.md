title: 网站的高性能
categories: Architecture
date: 2016-03-25 16:21:39
tags: Architecture
description: 高性能网站
---

## 前言

继续读书，继续笔记。

## 性能测试指标

- 响应时间：可以利用多次请求得出平均值
- 并发数：利用多线程模拟用户并发访问测试
- 吞吐量：TPS（每秒事务数）、HPS（每秒HTTP请求数）、QPS（每秒查询数）
- 性能计数器：System Load、对象与线程数、内存使用、CPU使用、磁盘与网络IO指标

## 性能测试方法

- 性能测试
- 负载测试
- 压力测试
- 稳定性测试

关于这四点，可以用两个图表来表示，就一目了然。

<!-- more -->

![性能测试曲线](https://raw.githubusercontent.com/rason/rason.github.io/master/image/architecture-high-performance-1.png)
![并发用户访问响应时间曲线](https://raw.githubusercontent.com/rason/rason.github.io/master/image/architecture-high-performance-2.png)

## 性能测试报告

![性能测试结果报告](https://raw.githubusercontent.com/rason/rason.github.io/master/image/architecture-high-performance-3.png)

## 性能优化策略

### Web前段性能优化

**1.浏览器优化**

- 减少HTTP请求：合并CSS、JS、图片
- 使用浏览器缓存
- 启用压缩：HTML、CSS、JS文件Gzip压缩
- CSS放最上面、JS放最下面
- 减少Cookie传输：减少数据量，静态资源使用独立域名避免发送Cookie

**2.CDN加速**
**3.反向代理**

### 应用服务器优化

**1.分布式缓存**

- Memcached

**2.异步操作**

- 消息队列消峰
- 能晚点做的事情就晚点做

**3.使用集群**

**4.代码优化**

- 多线程
- 资源复用（池的概念：数据库连接池、线程池之类）

### 存储性能优化

略。

## 总结

懂的一些概念，懂的一些研究方向。
