title: Gitlab+Jenkins持续集成
categories: Git
date: 2016-12-28 10:25:11
tags: Gitlab Jenkins
description: Gitlab+Jenkins持续集成
---

## 前言

公司内部版本管理使用SVN，一直就觉得挺麻烦的，所以就自个折腾一下把版本管理转移到git。后续的问题当然就是要将发布系统搭建起来，大概搜索了一下情况，决定使用Gitlab+Jenkins来做持续集成。

## Gitlab安装

[Gitlab官网](https://about.gitlab.com/downloads/)上有教程，选择适合自己的操作系统按照步骤一步步安装就可以了。安装过程可能会遇到一两个小问题，谷歌一下应该可以解决。

安装完成之后，在浏览器中输入`127.0.0.1`或者`localhost`回车即可，Gitlab默认监听80端口。然后注册一个账户即可进入系统。

![首页](/image/gitlab-home.png)

## Gitlab使用

如果你有Github的使用经验，Gitlab使用起来应该就非常熟练了，不过没有经验的话也是相当简单的。

### New project

点击右上角的`New project`创建一个新的工程，输入工程名称、工程描述和选择访问权限点击`Create project`即可。

<!-- more -->

![New project](/image/new-project.png)

### 推送代码到Gitlab

两种方式：

- 创建一个新的git仓库

```
git clone http://rason-ThinkPad-T430/root/test.git
cd test
touch README.md
git add README.md
git commit -m "add README"
git push -u origin master
```
- 将现有的git仓库推送到Gitlab

```
cd existing_folder
git init
git remote add origin http://rason-ThinkPad-T430/root/test.git
git add .
git commit
git push -u origin master
```

关键点是建立起和远程分支的追踪关系，通过`git clone`是默认建立关系，而`git remote add`是自己手动建立追踪关系。

### 分支管理

通过上面推送代码的操作，Gitlab中只有一个`master`分支，我们可以建立一些其他分支，关于分支管理模型可以见[上一篇](http://rason.me/2016/12/22/git-branching-model/)博客。

## Jenkins安装

可以通过下载war包的方式，也可以通过命令行的方式来安装，我自己使用的是Ubuntu系统，所以就选择命令行安装了。

```
wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```

安装完成之后在浏览器输入`localhost:8080`即可访问，Jenkins默认监听8080端口。由于我们做Java开发的通常都会用到8080端口，所以可以通过`/etc/default/jenkins`文件中的`HTTP_PORT=8080`这一行来修改端口号。然后`/etc/init.d/jenkins restart`重启一下即可。

第一次进入Jenkins会让你选择安装一些插件，直接选择默认就行了，因为缺少的插件之后可以在Jenkins系统管理-管理插件中安装。

![Jenkins系统管理](/image/jenkins-sys-man.png)

## Jenkins新建项目

点击左上角的`新建`，可以创建一个新的构建项目，输入名字和选择类型点击确定即可。

### 项目配置

重点就是项目的配置了，每个配置的右边都有个问号的图标，如果不太清楚这项配置是什么作用，点击这个问号会有解析。

### 源码管理

即Jenkins从哪里拉取代码进行构建，这里我们可以选择Git或者SVN，如果没有Git选项，那就是没有安装git的插件，需要到管理插件中先安装Git的插件。

![源码管理](/image/git-source.png)

输入相应的仓库地址和构建的分支即可。

### Build

使用maven进行编译打包。

![Build](/image/build.png)

### Post Steps

编译打包完之后就是如何将产品上传到服务器进行发布了，可以在`Post Steps`配置编译打包完的后置操作。这里我选择通过SSH将代码上传到服务器，你也可以选择SCP或者是FTP的方式。每种方式都有相应的插件，通过SSH上传需要安装`publish over ssh`插件。

![Publish over ssh](/image/publish-over-ssh.png)

上面的配置如果看不懂的话，可以[点击这里](https://wiki.jenkins-ci.org/display/JENKINS/Publish+Over#PublishOver-promotions)查看使用示例。

其中`Exec command`配置项可以在上传完代码之后执行一些命令，比如进行Tomcat的重启，这里我就不举例了。

配置了以上几项基本上就可以将整个持续集成环境跑起来了，当然Jenkins还有还多其它丰富的构建配置项就不细说了。

## 总结

整个持续集成的环境大概流程就是这样：将代码提交到Gitlab--->Jenkins从Gitlab拉取代码进行编译打包--->通过Jenkins的Publish over ssh插件上传代码到服务器--->配置Publish over ssh插件中的`Exec command`选项执行上传代码之后的重启服务器脚本。
