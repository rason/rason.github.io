title: Linux任务列表
categories: Linux
date: 2016-11-15 16:01:11
tags: 任务列表
description: Linux任务列表
---

## 前言

今天简单地学习一下Linux的进程描述符及其相关的数据结构，并编写一个内核模块将所有进程的简单信息用伪文件读取出来。

## 多任务

- 现代操作系统允许多用户共享计算机资源
- 每个用户都允许运行多个任务
- 操作系统内核必须保证每个任务不受其他任务干扰

## 栈和任务描述符

- 为了管理多任务，操作系统需要一个数据结构来跟踪每个任务的进展和对计算机资源的使用情况（物理内存、打开的文件、挂起信号等）
- 这样的一个数据结构称为**进程描述符**——每个活动中的任务都有一个
- 每个任务都需要有自己私有的栈

## 一个程序的栈里面有什么？

### 进入main()函数时

- 程序的退出地址在用户栈中
- 命令行参数在用户栈中
- 环境变量在用户栈中

### 执行main()期间

- 方法参数和返回地址
- 本地变量

## 进入内核

用户进程进入内核模式的几种情况：

- 执行系统调用
- 发生中断
- 发生异常

## 切换到一个不同的栈

- 进入内核模式不仅会引起权限等级的变化，还会进行栈的切换
- 这对于健壮性很有必要，比如用户栈可能会被耗尽
- 也是出于安全的原因，因为有些权限数据可能会被访问

## 内核栈中有什么？

进入内核模式时：

- 任务的寄存器会保存在内核栈中（比如任务的用户栈地址）

执行内核方法期间：

- 方法参数和返回地址
- 本地变量

## 支撑结构

- 所以每个任务，除了有自己的代码和数据，还会有一个在用户空间的栈加上另一个在内核空间的栈
- 每个任务都有一个只能在内核空间访问的进程描述符

## 任务在虚拟内存中的布局

![任务在虚拟内存中的布局](/image/task-layout-in-vm.png)

<!-- more -->

## Linux进程描述符

![Linux进程描述符](/image/process-descriptor.png)

## 任务状态

定义在内核头文件<linux/sched.h>

- #define TASK_RUNNING 			0
- #define TASK_INTERRUPTIBLE	    1
- #define TASK_UNINTERRUPTIBLE	2
- #define TASK_STOPPED			4
- #define TASK_TRACED			    8
- #define TASK_NONINTERACTIVE  	64
- #define TASK_DEAD			    128

## 任务的thread-info

- Linux使用任务内核栈页框的一部分来存储`thread-info`
- 这个`thread-info`包含一个指向进程描述符数据结构的指针

![thread-info](/image/thread-info.png)

## 如何找到任务的thread-info？

- 任务在内核运行期间，可以很快速地找到任务的thread-info
- 只需要两条汇编指令

```
movl		$0xFFFFF000, %eax
andl		%esp, %eax
```

解析：上面的指令是针对4KB的内核栈，esp寄存器指向的是内核栈的栈顶，因为thread-info位于内核栈的最低地址，所以将esp寄存器的地址值低12位屏蔽（按位与）即可得到该内核栈的最低地址，即thread-info。

- 内核中有现成的宏实现了这个计算

```
struct thread_info  *info = task_thread_info( task );

struct task_struct  *task = info->task;
```

## 内核任务列表

- 内核使用了双向环形链表来维持一系列的进程描述符
- `init_task`服务作为一个固定的表头
- 其他任务动态地进行插入/删除
- 任务有向前和向后的指针
- 向前：task = next_task( task )
- 向后：task = prev_task( task )

![任务双向环形链表](/image/doubly-linked-cricular-list.png)

## Demo

- 我们可以编写一个内核模块来建立一个伪文件查看当前活动任务列表（类似`ps -ef`）

```
#include <linux/module.h>	
#include <linux/proc_fs.h>	
#include <linux/sched.h>	// for init_task
#include <linux/seq_file.h>

char modname[] = "tasklist";
struct task_struct  *task;
int  n_elt;	


static int proc_show(struct seq_file *m, void *v) {

	
	task = &init_task;
	n_elt = 0;	
	
	seq_printf(m, "#%-3d ", ++n_elt );
	seq_printf(m, "%5d ", task->pid );
	seq_printf(m, "%lu ", task->state );
	seq_printf(m, "%-15s ", task->comm );
	seq_printf(m, "\n" );
	task = next_task( task );

	while( task != &init_task) {
		seq_printf(m, "#%-3d ", ++n_elt );
		seq_printf(m, "%5d ", task->pid );
		seq_printf(m, "%lu ", task->state );
		seq_printf(m, "%-15s ", task->comm );
		seq_printf(m, "\n" );
		task = next_task( task );
	}
	
	return	0;	
}

static int proc_open(struct inode *inode, struct  file *file) {
  return single_open(file, proc_show, NULL);
}

static const struct file_operations proc_file_fops = {
	.owner = THIS_MODULE,
	.open = proc_open,
	.read = seq_read,
	.llseek = seq_lseek,
	.release = single_release,
};

static int __init my_init( void )
{
	printk( "<1>\nInstalling \'%s\' module\n", modname );
    proc_create(modname, 0, NULL, &proc_file_fops);
  	return 0;
}


static void __exit my_exit(void )
{
	remove_proc_entry(modname, NULL);

	printk( "<1>Removing \'%s\' module\n", modname );
}

module_init( my_init );
module_exit( my_exit );
MODULE_LICENSE("GPL"); 
```

使用`$ cat /proc/tasklist`来查看，我的机器部分输出如下：

![当前活动任务](/image/current-active-task.png)

## 总结

本文学习了一下Linux中进程机器相关数据结构，知道进程在内核中是以双向环形链表的形式维持的，并且编写了一个小Demo将当前的进程简单信息输出到伪文件中查看。
