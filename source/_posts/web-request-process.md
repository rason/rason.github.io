title: Web请求过程
categories: Network
date: 2016-02-03 10:13:07
tags: Web请求
description: Web请求过程
---

## 前言

休了两个星期的假，半个多月没更新博客了。曾经有太多的事情没有坚持下去，希望这个博客能够成为一个成功的例子。继续自己的阅读，学习，记录之旅，今天阅读了一些关于Web请求的基础知识，简单地做些笔记。

## B/S网络架构

B/S网络架构有两个好处：

- 客户端使用统一的浏览器（Browser）。
- 服务端（Server）基础统一的HTTP协议。

与传统的C/S采用长连接的模式不同，B/S采用无状态的短连接方式，一次请求完成一次数据交互。采用这种模式能够同时服务更多的用户，因为现在的互联网每天都会处理上亿的请求，不可能用户访问一次就一直保持这个连接。
基于Http协议本身的特点，目前的B/S网络架构大都采用类似下图所示的设计：

![CDN架构](https://raw.githubusercontent.com/rason/rason.github.io/master/image/web-cdn-structure.png)

<!-- more -->

## 请求发起

发起一个Http请求的过程就是建立一个Socket连接的过程。当我们在浏览器键入一个连接并回车，浏览器会根据URL组装成一个get类型的Http请求头，通过outputStream.write()发送到目标服务器，服务器等待inputStream.read()返回数据，最后断开这个连接。因此我们可以模拟浏览器来发起Http请求，只要我们outputStream.write()写的二进制字节数据格式符合Http协议即可。当然Java中已经有一些开源项目可以直接供我们使用，比如HttpClient。

在Linux中也可以简单地使用`curl+URL`发起一个Http请求。

```
例如：curl rason.me，会返回我的个人博客首页的HTML数据
```

也可以通过`curl+URL -I`方式获取此次请求的相应头信息。

## Http协议

要理解Http协议，最重要的是要熟悉Http Header，常见的Http请求头和响应头如下：

**常见的Http请求头**

| 请求头             | 说明                                                    | 
| ----------------  |:------------------------------------------------------:| 
| Accept-Charset    | 指定客户端接受的字符集 | 
| Accept-Encoding   | 指定可接受的内容编码，如Accept-Encoding:gzip-.deflate       | 
| Accept-Language   | 指定一种自然语言，如Accept-Language:zh-cn                  | 
| Host              | 指定被请求资源的Internet主机和端口号，如Host:www.baidu.com   | 
| User-Agent        | 客户端将它的操作系统，浏览器和其它熟悉告诉服务器                | 
| Connection        | 当前连接是否保持，如Connection:Keep-Alive                  | 

**常见的Http响应头**

| 响应头             | 说明                                                    | 
| ----------------  |:------------------------------------------------------:| 
| Server            | 使用的服务器名称，如Server:Apache/1.3.6(Unix) | 
| Content-Type      | 指明发送给接收者的正文媒体类型，如Content-Type:text/html;charset=GBK   | 
| Content-Encoding  | 与请求报文Accept-Encoding对应，告诉浏览器服务器采用的是什么压缩编码       | 
| Content-Language  | 描述资源使用的自然语言，与Accept-Language对应                         | 
| Content-Length    | 指明实体正文的长度，用以字节方式存储的十进制数字来表示                     | 
| Keep-Alive        | 保持连接的时间，如 Keep-Alive:timeout=5,max=120                     | 

**常见的Http状态码**

| 状态码  | 说明                                                    | 
| ------ |:-----------------------------------:| 
| 200    | 客户端请求成功 | 
| 302    | 临时跳转，跳转的地址由Location指定       | 
| 400    | 客户端请求语法有错误，不能被服务器识别     | 
| 403    | 服务器接收到请求，但是拒绝提供服务        | 
| 404    | 请求的资源不存在                       | 
| 500    | 服务器发送不可预期的错误                | 
| 504    | 网关超时                             |

## DNS域名解析

DNS解析的工作就是将域名解析成IP地址，过程如下：

- 浏览器检查缓冲中有没有该域名对应的解析过的IP地址，缓冲时间通过TTL属性设置
- 浏览器检查操作系统中是否有这个域名对应的DNS解析结果。操作系统也会有域名解析的过程，在windows中通过才C:\Windows\System32\drivers\etc\hosts文件设置；Linux是/etc/named.conf
- 操作系统将域名发送到本地区域名服务器LDNS进行解析。这个DNS服务器可在网络配置中“DNS服务器地址”中设置，如果是小区网络，这个DNS就是电信或者联通，通常这个服务器不会在很远的地方。在windows中可以通过`ipconfig`查看，Linux中可以通过`cat /etc/resolv.conf`查看。大约80%的域名解析工作都到这里就已经完成了，所以LDNS主要承担了域名的解析工作。
- 如果LDNS还没有命中，就直接到Root Server域名服务器解析，目前全世界只有几台根域名解析服务器
- 根域名服务器返回给本地域名服务器一个所查询域的主域名服务器（gTLD Server）。gTLD是国际顶级域名服务器，如.com,.cn,.org等，全球只有13台
- 本地域名服务器（LDNS）再向上一步返回的gTLD服务器发送请求
- 接收请求的gTLD服务器查询并返回此域名对应的Name Server域名服务器的地址，这个Name Server通常就是你注册的域名服务器
- Name Server域名服务器会查询存储的域名和IP的映射关系表，正常情况下会根据域名得到目标IP，连同一个TTL值返回给Name Server域名服务器
- 返回域名对应的IP和TTL值，LDNS会缓存这个IP对应关系，缓存时间有TTL值决定
- 把解析结果返回给用户，用户根据TTL值缓存在本地系统中，域名解析过程结束

![DNS解析](https://raw.githubusercontent.com/rason/rason.github.io/master/image/web-dns-resolv.png)

### 跟踪域名解析过程

- Linux和windows都可以用`nslookup`
- Linux还可以用`dig 域名`
- Linux还可以用`dig 域名 +trace`跟踪整个解析过程

### 清除域名缓存

清除本机缓存：
- windows：ipconfig /flushdns
- Linux: /etc/init.d/nscd restart

### 几种域名解析方式

- A记录，A代表address，用来指定域名对应的IP。可以将多个域名解析到一个IP，但不能将一个域名解析到多个IP。
- MX记录，表示的是Mail Exchange，可以将某域名下的邮件服务器指向自己的Mail Server。
- CNAME记录，即别名解析，可以为一个域名设置一个或多个别名。
- NS记录，为某个域名指定DNS解析服务器，也就是这个域名由指定的IP地址的DNS服务器去解析。
- TXT记录，为某个主机名或域名设置说明。

## CDN工作机制

CDN即内容分布网络（Content Delivery Network），它是构建在现有Internet上的一种先进的流量分配网络，使用户可以就近取得所需内容，提高网站响应速度。目前CDN都以缓存网站中的静态数据为主，如CSS，JS，图片和静态页面等数据。

当一个用户访问一个静态资源，这个静态资源的域名假设是cdn.taobao.com，一般会CNAME到另一个域名，而这个域名最终会指向CDN全局中的DNS负载均衡服务器。再由这个GTM来最终分配是哪个地方的的访问用户，返回给离这个用户最近的CDN节点。拿到DNS解析结果，用户就直接去这个CDN节点访问这个静态文件，如果所请求的文件不存在，就会再回源站获取，然后返回给用户。

### 负载均衡

负载均衡（Load Balance）就是对工作任务进行平衡，分摊到多个操作单元上执行，如图片服务器，应用服务器等，共同完成任务。提高服务器的响应速度和利用效率，避免软件或者硬件模块出现单点失效，解决网络拥塞问题等。

通常有三种负载均衡架构：

- 链路负载均衡：通过DNS负载均衡服务器解析成不同的IP，然后用户根据这个IP来访问不同的目标服务器。优点：无代理访问快，缺点：本地和LDNS都有缓存，一台服务器挂了缓存没更新会导致访问失败。

![链路负载均衡](https://raw.githubusercontent.com/rason/rason.github.io/master/image/web-link-lb.png)

- 集群负载均衡：分为硬件负载均衡和软件负载均衡。
	 硬件负载均衡：使用一台专门的转发设备来转发请求，如F5。优点：性能好，缺点：贵。
	 软件负载均衡：使用最普遍，使用廉价的PC即可搭建。优点：成本低，缺点：多次代理，网络延时。

	 ![硬件负载均衡](https://raw.githubusercontent.com/rason/rason.github.io/master/image/web-hard-lb.png)

	 ![软件负载均衡](https://raw.githubusercontent.com/rason/rason.github.io/master/image/web-soft-lb.png)

	 上图中，上面两台是LVS，使用四层负载均衡，也就是通过网络层利用IP地址进行地址转发。下面三台使用HAProxy进行七层负载，也就是可以通过根据访问用户的Http请求头来进行负载均衡，如可以根据不同的URL来请求转发到特定机器或者根据用户的Cookie信息来指定访问的机器。

- 操作系统负载均衡：利用操作系统级别的软中断或者硬件中断来达到负载均衡，如可以设置多队列网卡来实现。

## 总结

本文介绍了一些Web请求相关的基本知识，包括网络请求，DNS解析和CDN相关知识点，使我们对B/S网络架构有了基本的了解。
