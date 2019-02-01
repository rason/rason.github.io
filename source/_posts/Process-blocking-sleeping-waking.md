title: 简单介绍进程的阻塞和Linux内核对进程睡眠和唤醒的支持
categories: Linux
date: 2016-11-17 13:25:34
tags: Linux
description: 进程的阻塞 环形缓存
---

## 前言

今天简单学习一下进程的阻塞和Linux内核对进程睡眠和唤醒的支持，并编写一个环形缓存应用来演示睡眠和唤醒。

## 设备可能会空转

- 比如键盘，程序可能在等待键盘的输入
- 这时，我们希望程序等到输入再运行，而不是放弃等待

## 设备可能会繁忙

- 比如打印机，多个程序调用打印，打印机可能暂时没法接收更多的数据
- 这个时候，我们希望等到打印机可以接收更多的数据，而不是放弃打印

## 使用忙等待

- 针对上面的情况，我们可让设备驱动不断轮询一个状态位，直到数据准备好或者设备不再繁忙

```
do {
	status = inb( 0x64 );
}
while ( ( status & READY ) == 0 );
```

- 这种技术称为`忙等待`
- 但是这样做会浪费宝贵的CPU时间

## 避免忙等待

- 在多任务系统中我们希望尽可能地避免使用`忙等待`策略，因为这会成为性能的瓶颈
- 所以现代的操作系统支持一个可选择的策略，使得任务可以这样处理

### 空转时阻塞

- 如果一个任务尝试从一个暂时没有数据的设备文件读取，但又希望新的数据到来；这时操作系统通过将这个任务进入`睡眠`进行阻塞，避免在等待期间浪费宝贵CPU时间；但是会在数据一到来的时候就将任务`唤醒`。

### 繁忙时阻塞

- 同样地，如果一个任务尝试写一个设备文件，但是设备在忙于先去写的数据；操作系统会使这个任务睡眠，避免它浪费宝贵的CPU时间，以便其他任务可以做一些有用的工作；当设备可能接收更多数据时会第一时间将任务唤醒。

<!-- more -->

## 睡眠和唤醒是什么意思？

- Linux内核将一个任务进入睡眠只是简单地修改它的状态值：
  - TASK_RUNNING
  - TASK_STOPPED
  - TASK_UNINTERRUPTIBLE
  - TASK_INTERRUPTIBLE

- 睡眠的任务状态为TASK_INTERRUPTIBLE或者TASK_UNINTERRUPTIBLE
- 通过修改睡眠任务的状态为TASK_RUNNING将任务`唤醒`
- 当Linux调度程序发现任务的状态为TASK_RUNNING时，它才会被授予CPU时间来执行

## 运行队列和等待队列

- 为了让Linux更高效地管理众多任务的调度，分开了两个队列来维护运行中的任务和临时阻塞的任务
- 准备运行的任务放在运行队列中
- 阻塞的任务放在等待队列中

## 内核的支持程序

- Linux内核提供了支持程序使驱动很方便地执行睡眠和唤醒动作避免潜在的竞争条件
- 通过在你的驱动中导入<linux/sched.h>即可使用支持程序

```
wait_queue_head_t		wq;
init_waitqueue_head( &wq );
wait_event_interruptible( wq, <condition> );
wake_up_interruptible( &wq );
```

## 我们的"stash"设备

- 这次我们要编写一个类似公共剪切板的设备
- 使用内核内存来存储数据
- 允许任务之间进行通讯
- 一个任务写，另一个任务读

## 环形缓存

- 一个先进先出的数据结构
- 用来存储有限长度的数组
- 使用两个数组索引：head和tail
- 数据在tail索引处插入
- 数据在head索引处删除

![环形缓存](https://raw.githubusercontent.com/rason/rason.github.io/master/image/ringbuffer.png)

- 总有一个数组位置没有被使用
- 当head == tail表示没有数据
- 当tail == head-1表示装满了
- 计算：next = ( next+1 )%RINGSIZE

## stash.c

```
#include <linux/module.h>	// for init_module() 
#include <linux/fs.h>		// for register_chrdev()
#include <linux/sched.h>	// for TASK_INTERRUPTIBLE
#include <asm/uaccess.h>	// for get_user(), put_user()

#define RINGSIZE 512

char modname[] = "stash";
int my_major = 40;
unsigned char ring[ RINGSIZE ];
volatile int head = 0, tail = 0;
wait_queue_head_t  wq_rd, wq_wr;


ssize_t
my_read( struct file *file, char *buf, size_t count, loff_t *pos )
{
	// sleep if necessary until our ringbuffer has some data 
	if ( wait_event_interruptible( wq_rd, head != tail ) )
		return -ERESTARTSYS;

	// remove a byte of data from our ringbuffer
	if ( put_user( ring[ head ], buf ) ) return -EFAULT;
	head = ( 1 + head ) % RINGSIZE;

	// now awaken any sleeping writers
	wake_up_interruptible( &wq_wr );
	return	1;
}

ssize_t
my_write( struct file *file, const char *buf, size_t count, loff_t *pos )
{
	int	next = (1 + tail) % RINGSIZE;

	// sleep if necessary until our ringbuffer has room for more data
	if ( wait_event_interruptible( wq_wr, next != head ) ) 
		return -ERESTARTSYS; 

	// insert a new byte of data into our ringbuffer
	if ( get_user( ring[ tail ], buf ) ) return -EFAULT; 
	tail = (1 + tail) % RINGSIZE;

	// now awaken any sleeping readers 
	wake_up_interruptible( &wq_rd );
	return	1;
}


static struct file_operations 
my_fops =	{
		owner:		THIS_MODULE,
		write:		my_write,
		read:		my_read,
		};


int __init my_init( void )
{
	printk( "<1>\nInstalling \'%s\' module ", modname );
	printk( "(major=%d) \n", my_major );

	// initialize our wait-queue structures
	init_waitqueue_head( &wq_rd );
	init_waitqueue_head( &wq_wr );
	
	// register this device-driver with the kernel
	return	register_chrdev( my_major, modname, &my_fops );
}


void __exit my_exit( void )
{
	printk( "<1>Removing \'%s\' module\n", modname );

	// unregister this device-driver 
	unregister_chrdev( my_major, modname );
}


module_init(my_init);
module_exit(my_exit);
MODULE_LICENSE("GPL");
```

- 编译`$ make`
- 安装`$ sudo /sbin/insmod stash.ko`
- 创建设备文件`$ sudo mknod /dev/stash c 40 0`
- 授权读写权限`$ sudo chmod a+rw /dev/stash`
- 测试写`$ echo "hello">/dev/stash`
- 测试读`$ cat /dev/stash`

## 总结

本文学习了进程的睡眠和唤醒，并通过一个类剪贴板程序演示了任务的睡眠和唤醒。当环形缓存满了的时候，继续向其写数据的话，会将该任务进入写等待队列（即睡眠），当有空闲空间可以写时，则会从写等待队列中唤醒一个任务；读的时候也是类似，有数据就读，没数据就进入读等待队列，有数据写入时会被唤醒。
