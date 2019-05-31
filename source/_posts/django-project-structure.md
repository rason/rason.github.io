title: Django工程结构
categories: Django
date: 2016-03-22 15:04:45
tags: Django
description: Django工程结构
---

## 前言

在前两节课程中，我们在阿里云上搭建了一个Django的HelloWorld项目，并且将项目的代码托管在github上。这样，基本的条件都差不多算是准备好了，正常来说可以开始着手写业务代码了。可是，当我在了解Django的`project`和`app`区别的过程中，我发现我们有必要先把工程的结构先捋清楚。这样才能使我们的工程结构更加一目了然，也便于以后工程的分离，这一点，就是在《大型网站技术架构》一书中谈到的分离和分割。

**注意：**

- 由于本人是现学现卖，所以教程可能存在不正确的内容，请大家指正
- 工程代码的托管地址是：[点击这里](https://github.com/rason/HelloPython.git)

## 背景

在[教程一](http://rason.me/2016/03/03/Deploy-django-on-aliyun/)中我们创建了一个叫做`ebookloader`的Django项目，并且创建了一个`booksystem`应用。那么Django中的工程和应用究竟有什么不同？我举个例子，比如说我要做一个电子书下载的网站，这个网站包含一个电子书下载的功能和一个浏览科技新闻的功能。那么这个网站就是一个工程，这两个功能可以分别作为一个应用。

所以，我们要管理两个或者更多的应用，工程的结构应该是怎么样会更好呢？据我搜索的结果来说，没有固定唯一的结构，只有相对合理的结构。所以，这里我把我**目前水平认为**相对合理的结构讲解一下。

<!-- more -->

## Django缺省工程结构

当我们新创建一个Django工程，它的结构是这样的：

```
rason@rason-ThinkPad-T430:~/Documents/python-learn/projectdemo$ tree
.
├── manage.py
└── projectdemo
    ├── __init__.py
    ├── settings.py
    ├── urls.py
    └── wsgi.py

```
这是一个还没有创建应用是的工程结构，其文件分别代表的意思为：

- **项目目录**：projectdemo
- **manage.py**：用于管理Django站点
- **settings.py**: 包含项目的所有配置参数
- **urls.py**: URL根配置
- **wsgi.py**: 内置runserver命令的WSGI应用配置
- **__init__.py**: 用来告诉python，当前目录是python模块

## 当前教程的工程结构

根据分离和分割的原则，在默认工程的基础上，我将当前教程的工程结构改造成了如下结构：

```
root@iZ94hd1izoaZ:/mysite# tree
.
├── ebookloader
│   ├── apps
│   │   ├── book
│   │   │   ├── admin.py
│   │   │   ├── apps.py
│   │   │   ├── __init__.py
│   │   │   ├── migrations
│   │   │   │   └── __init__.py
│   │   │   ├── models.py
│   │   │   ├── templates
│   │   │   │   └── book
│   │   │   │       └── base.html
│   │   │   ├── tests.py
│   │   │   └── views.py
│   │   └── __init__.py
│   ├── __init__.py
│   ├── __init__.pyc
│   ├── settings
│   │   ├── base.py
│   │   ├── dev.py
│   │   ├── __init__.py
│   │   └── local.py
│   ├── static
│   │   ├── css
│   │   │   └── site.min.css
│   │   ├── img
│   │   │   └── favicon.png
│   │   └── js
│   │       └── site.min.js
│   ├── templates
│   │   ├── 404.html
│   │   ├── 500.html
│   │   └── base.html
│   ├── urls.py
│   └── wsgi.py
└── manage.py

```

大致的改造地方有：

- **apps**:放置各个应用，此处只有一个名为`book`的应用，当我们想新增一个`news`应用时，直接进入apps文件夹中执行`python manage.py startapp news`即可
- **settings**:默认工程只有一个settrings.py文件用于项目配置，此处将测试环境，开发环境，生产环境不同的配置放置在settings文件夹中
- **static**:静态资源配置
- **templates**:全局通用模板

以上改造只是一个大致的结构，可以继续细分或者优化，并没有唯一的标准。需要注意的是，修改了settings文件的位置需要修改其引用的地方。

## 总结

像我这样的初学者，只需大致的按照模块划分工程的结构即可，不必追求完美。可以在我们学习的过程中，不断地对工程进行结构优化。
