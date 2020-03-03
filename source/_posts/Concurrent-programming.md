title: 并发编程
categories: Linux
date: 2016-11-22 10:02:41
tags: 并发编程
description: 并发编程
---

## 前言

今天来简单地学习Linux中的编发编程，介绍一下互斥和线程同步。

## stash.c的问题

- 上一篇博客我们写了一个简单的公共剪切板设备驱动来演示进程的睡眠和唤醒
- 细心的同学应该能够发现，当多个进程同时读或者写可能会出现问题

## 竞争条件

- 当两个或多个进程试图在同一时刻访问共享内存，或读写某 些共享数据，而最后的结果取决于线程执行的顺序(线程运行时序)，就称为竞争条件
- 如果没有同步机制，多道程序设计很容易因为竞争条件而出现不是我们所期待或者是错误的结果

## 临界区

- 对共享变量，共享内存，共享资源进行访问的程序片度叫做临界区

## 互斥

在某些提供`test-and-set`指令的机器可以用于解决临界区互斥的问题，这个指令能检查并设置内存的值，而不会被打断。工作原理大致是这样的：

```
boolean TestAndSet(boolean *lock){
	boolean rv = *lock;
	*lock = TRUE;
	return rv;
}
```

**注意：**这个函数中三条指令是`原子`执行的，也就是说不会被打断。

使用方法：

```
boolean lock = false;

while(TestAndSet(&lock)){
	;// 啥也不干
}

临界区；

lock = false;

其他代码；
```

<!-- more -->

## 信号量

- 除了使用`test-and-set`指令来解决互斥问题，还可以使用更加强大的信号量。

![信号量数据结构](/image/semaphore-struct.png)

如上图所示，对于一个互斥的信号量（比如互斥锁），count属性会被初始化为1，代表最多一个任务可以获得这个信号量。

当一个任务调用`down（&sem）`方法——这个方法会对减少count值,如果新的count值大于0(获得信号量)则会立即返回执行接下来的代码，否则，这个任务会被进入睡眠，即放到信号量的`task_list`等待队列。之后，当一个已经获得信号量的任务完成之后，调用`up(&sem)`方法，count值就会增加，等待队列中的所有任务都会被唤醒。

从上面的描述可知，使用信号量不仅可以解决互斥的问题，还可以解决`生产者-消费者`同步的问题。

使用信号量解决互斥的伪代码：

```
sem->count=1;

down(&sem);

临界区

up(&sem);

```

使用信号量解决生产者-消费者同步问题伪代码：

```
sem_lock->count = 1; // 用于访问队列时互斥
sem_empty->count = 5; // 生产者信号量，大于0可生产
sem_full->count = 0; // 消费者信号量，大于0可消费

生产者：
while(true){
	down(&sem_empty); // 如果队列中没有空余的位置，生产者只能等待

	down(&sem_lock); // 加锁，准备操作队列

	把新产生的内容加入队列；

	up(&sem_lock); // 释放锁

	up(&sem_full); // 通知消费者有内容可以消费了
}

消费者：
while(true){
	down(&sem_full); // 如果队列中没有内容，消费者只能等待

	down(&sem_lock); // 加锁，准备操作队列

	将队列中的内容取出进行处理；

	up(&sem_lock); // 释放锁

	up(&sem_empty); // 通知生产者进行生产
}
```

- **注意：**当然对信号量的down和up操作是在内核中实现的，并且屏蔽了中断，否则count本身也会出现竞争条件。

## newstash.c

- 了解信号量之后，我们修改一下stash.c用两个信号量来限制在一个时间点只能有一个读进程和一个写进程来访问设备文件
- 其他进程如果想尝试打开设备文件会进入睡眠，一旦信号量被释放则会被唤醒

```
#include <linux/module.h>	// for init_module() 
#include <linux/fs.h>		// for register_chrdev()
#include <linux/sched.h>	// for wait_event_interruptible() 
#include <asm/uaccess.h>	// for get_user(), put_user()

#define RINGSIZE 512

char modname[] = "stash";	// keeps former pseudo-file name
int my_major = 40;		// and same device-file major ID
unsigned char ring[ RINGSIZE ];
volatile int head = 0, tail = 0;
wait_queue_head_t  wq_rd, wq_wr;

// here two semaphores are added
struct semaphore sem_wr, sem_rd;  


int my_open( struct inode *inode, struct file *file )
{
	// if task wants to write, obtain 'write' semaphore (or sleep)
	if ( file->f_mode & FMODE_WRITE ) down_interruptible( &sem_wr );

	// if task wants to read, obtain 'read' semaphore (or sleep)
	if ( file->f_mode & FMODE_READ ) down_interruptible( &sem_rd );	

	return	0;  // SUCCESS
}


int my_release( struct inode *inode, struct file *file )
{
	// if task was a writer, then release the 'write' semaphore
	if ( file->f_mode & FMODE_WRITE ) up( &sem_wr );

	// if task was a reader, then release the 'read' semaphore
	if ( file->f_mode & FMODE_READ ) up( &sem_rd );	

	return	0;  // SUCCESS
}


ssize_t
my_read( struct file *file, char *buf, size_t count, loff_t *pos )
{
	// sleep if necessary until the ringbuffer has some data 
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
	// sleep if necessary until the ringbuffer has room for more data
	int	next = (1 + tail) % RINGSIZE;
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
		open:		my_open,
		release:	my_release,
		};

int __init my_init( void )
{
	printk( "<1>\nInstalling \'%s\' module ", modname );
	printk( "(major=%d) \n", my_major );

	// initialize our semaphores and wait-queue structures
	init_MUTEX( &sem_wr );
	init_MUTEX( &sem_rd );
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

## 总结

本文不但学习了`test-and-set`指令实现互斥，还学习了更加强大的信号量来解决`生产者-消费者`同步问题。
