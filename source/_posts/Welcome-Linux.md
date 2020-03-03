title: 欢迎来到Linux
categories: Linux
date: 2016-10-31 16:56:28
tags: Linux
description: Linux概述
---

## 前言

本文将快速学习一些Linux的设计理念和常用命令。

## 两种界面选择

### 图形界面

- 由彩色图标代表对象
- 用户主要通过图标来进行操作

### 传统命令行界面

- 通过文件名来访问对象
- 用户通过命令来进行操作

### 两种界面之间的切换

图形界面-->命令行：ctrl + alt + fn (n = 1, 2, 3, 4, 5, 6)

命令行-->图形界面：ctrl + alt + f7

## 一切皆文件

- 所有的普通文件都是字节序列
- 一切存储数据，其他的存储程序
- 也有一些特殊的文件，比如文件夹，设备文件和伪文件
- 每个文件都有唯一的名字（利用了目录层次结构）
- 每个文件都有具体的所有者
- 每个文件都有一系列的访问权限

![文件访问权限](/image/file-permissions.png)

比如：rw-r--r--

## 目录树

![目录树](/image/directory-tree.png)

Linux将成千上万的文件组织成树状的层次结构，顶层目录“/”称为根目录，通常包括大约二十来个子目录。

<!-- more -->

![home目录](/image/file-tree-home.png)

每个授权用户在home目录下都有一个子目录。

![boot目录](/image/file-tree-boot.png)

boot目录下包含一些Linux内核需要用到的文件和目录。

![etc目录](/image/file-tree-etc.png)

etc目录下保存的是系统配置项文件。

![bin目录](/image/file-tree-bin.png)

bin目录下包含上百个最基础的二进制可执行文件。

![usr目录](/image/file-tree-usr.png)

要用到的应用程序和文件几乎都在usr目录。

![var目录](/image/file-tree-var.png)

var目录下包含一些变量资源，比如日志文件。

## 在线手册

- Linux为所有命令提供了在线文档
- 可以通过`man <命令名>`来查看文档
- 比如`$ man ls`

## 常用命令

- cd：切换目录
- cp：复制文件
- mv：移动文件
- rm：删除文件
- rename：重命名文件
- who：查看登录用户
- mkdir：创建文件夹
- rmdir：删除文件夹
- scp：安全复制（远程文件复制）
- ssh：远程登录
- cat：将多个文件连接在一起打印到标准输出
- grep：根据匹配模式在文件中查找
- ln -s：软连接
- gcc：Gnu C 编译器
- g++：Gnu C++ 编译器
- make：编译和链接脚本
- tar：压缩加压
- diff：对比两个文本
- exit：结束进程
- chmod：修改权限
- more：按页来显示文本内容

举例：

解压：
`$ tar –xvf linux-2.6.16.6.tar`

压缩：
`$ tar –cvf linux-2.6.16.6.tar  *`



完。
