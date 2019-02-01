title: 编写一个简单的Linux内核模块
categories: Linux
date: 2016-11-03 15:51:49
tags: Linux
description: Linux内核模块
---

## 前言

Linux的可扩展性体现在两个方面：

- 开源
- 可加载的内核模块（LKMs, Loadable kernel module）

今天我们来认识一下LKM并编写一个简单的内核模块。

## 可加载的内核模块

- 是一项方便扩展操作系统的技术
- 同时让我们了解到内核是如何工作的
- 内核运行时也可以进行修改
- 不需要重新编译和重启
- 因而它也是不安全的，有bug的话容易造成系统崩溃

## 超级用户权限

- 修改运行中的内核是很危险的
- 只有授权的系统管理员才可以安装内核模块

## ‘insmod’ and ‘rmmod’

安装内核模块：`$  /sbin/insmod  myLKM.ko`

移除内核模块：`$  /sbin/rmmod  myLKM`

列出已安装的内核模块：`$  /sbin/lsmod`

## 创建一个新的内核模块

- 可以用任何文本编辑器编写Linux内核源码
- 内核模块和普通的C程序有点不一样，比如没有main()方法
- 内核模块不能调用标准C运行库的方法
- 对于任何的LKM，必须要有两个切入点，一个用于initialization，一个用于cleanup

<!-- more -->

## 跟C程序的另外一下不同点

- 内核模块用printk()代替printf()
- 包含<linux/module.h>头文件
- 指定一个合法的软件License（GPL）
- 编译需要一个专用的Makefile
- 没有权限控制

## 用C语言编写一个内核模块

```
#include <linux/module.h>		// for printk()

int init( void )
{
	printk( "\n Hello, everybody! \n" );

	return	0;
}

void exit( void )
{
	printk( "\n   Goodbye now... \n\n" );
}

MODULE_LICENSE("GPL");
module_init(init);
module_exit(exit);
```

## Makefile文件

```
ifneq	($(KERNELRELEASE),)
obj-m	:= kello.o 

else
KDIR	:= /lib/modules/$(shell uname -r)/build
PWD	:= $(shell pwd)
default:	
	$(MAKE) -C $(KDIR) SUBDIRS=$(PWD) modules 
	rm -r -f .tmp_versions *.mod.c .*.cmd *.o *.symvers 

endif

```

## 编译、安装、移除

编译：

- `$ make -f Makefile`
- `$ make`,前提是kello.c和Makefile在同一目录下。

安装：

- `$ sudo /sbin/insmod kello.ko`
- `$ /sbin/insmod` 查看是否安装成功
- `$ dmesg` 查看内核日志

删除：

- `$ sudo /sbin/rmmod kello`



完。
