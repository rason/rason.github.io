title: Linux内存问题
categories: Linux
date: 2016-11-25 15:56:11
tags: Linux内存
description: Linux内存问题
---

## 前言

今天学习一些关于内存管理方面的知识。

## 一些处理器架构的历史

- 8080 (late-1970s) 16-bit address (64-KB)
- 8086 (early-1980s) 20-bit address (1-MB)
- 80286 (mid-’80s) 24-bit address (16-MB)
- 80386 (late-’80s) 32-bit address (4-GB)
- 80686 (late-’90s) 36-bit address (64-GB)
- Core2 (mid-2000s) 40-bit address (1-TB)

## 向后兼容

- 新的处理器需要运行旧的程序
- 比如80x86依然可以运行很多8086的程序
- 当然也有一些不兼容的地方存在

## Linux必须适配这些遗留问题

- 遗留因素：硬件和固件
- CPU：复位地址和中断向量
- ROM-BIOS：数据区和boot位置
- DMA：24位内存地址总线
- 其他

## 其他处理器架构

- 除了IA-32，Linux还可以运行在其他CPU中（比如IBM360，MC68000等）
- 因此，Linux还需适配他们的不同点
 - Memory-Mapped I/O
 - Wider address-buses
 - Non-Uniform Memory Access (NUMA)

 ## Nodes, Zones, 和 Pages

 - Nodes：为了适配NUMA系统
 - 然而80x86系统不支持UNMA
 - 所以在80x86中只用了一个`node`
 - Zones：为了适配不同的区域
 - 80x86中的三个`zones`
  - ZONE_DMA		 (memory below 16-MB)
  - ZONE_NORMAL	     (from 16-MB to 896-MB)
  - ZONE_HIGHMEM	 (memory above 896-MB)

## Zones被切分为Pages

- 80x86支持4KB的页框
- Linux使用页描述符数组`mem_map[]`跟踪物理内存页
- 物理内存是由CPU来映射匹配的

<!-- more -->

## 80x86如何寻址

- 两个阶段：分段和分页
 - 首先：逻辑地址->线性地址
 - 然后：线性地址->物理地址

## 逻辑地址->线性地址

![逻辑地址->线性地址](/image/Logical-to-Linear.png)

## 线性地址->物理地址

![线性地址->物理地址](/image/Linear-to-Physical.png)

- CR3寄存器保存的是当前任务的页目录表的物理地址

## 页表项格式

![页表项](/image/PageTable-Entry.png)

## 分配任务内存

- 每个Linux进程都有一个进程描述符，里面有一个名为mm的指针

```
struct task_struct {
	pid_t				pid;
	char 				comm[16];
	struct mm_struct		*mm;
	...
};
```
## mm_struct结构体

- 用于描述任务的内存映射
- 代码段、数据段和堆栈段的位置
- 共享库的位置
- 每个内存区的属性
- 有多少个不同的内存区
- 任务的页目录位置

## 编写一个内核模块mm.c查看mm_struct内部属性

- 用来创建一个伪文件`/proc/mm`
- 让用来看到任务的mm_struct结构体的属性

```
#include <linux/module.h>	// for init_module() 
#include <linux/proc_fs.h>	// for create_proc_info_entry() 
#include <linux/sched.h>	// for 'struct task_struct'
#include <linux/mm.h>		// for 'struct mm_struct'
#include <linux/seq_file.h>

char modname[] = "mm";

static int proc_show(struct seq_file *m, void *v)
{
	struct task_struct	*tsk = current;
	struct mm_struct	*mm = tsk->mm;
	unsigned long		stack_size = (mm->stack_vm << PAGE_SHIFT);
	unsigned long		down_to = mm->start_stack - stack_size;
	

	seq_printf(m, "\nInfo from the Memory Management structure " );
	seq_printf(m, "for task \'%s\' ", tsk->comm );
	seq_printf(m, "(pid=%d) \n", tsk->pid );
	seq_printf(m, "   pgd=%08lX  ", (unsigned long)mm->pgd );
	seq_printf(m, "mmap=%08lX  ", (unsigned long)mm->mmap );
	seq_printf(m, "map_count=%d  ", mm->map_count );
	seq_printf(m, "mm_users=%d  ", mm->mm_users.counter );
	seq_printf(m, "mm_count=%d  ", mm->mm_count.counter );
	seq_printf(m, "\n" );
	seq_printf(m, "    start_code=%08lX  ", mm->start_code );
	seq_printf(m, " end_code=%08lX\n", mm->end_code );
	seq_printf(m, "    start_data=%08lX  ", mm->start_data );
	seq_printf(m, " end_data=%08lX\n", mm->end_data );
	seq_printf(m, "     start_brk=%08lX  ", mm->start_brk );
	seq_printf(m, "      brk=%08lX\n", mm->brk );
	seq_printf(m, "     arg_start=%08lX  ", mm->arg_start );
	seq_printf(m, "  arg_end=%08lX\n", mm->arg_end );
	seq_printf(m, "     env_start=%08lX  ", mm->env_start );
	seq_printf(m, "  env_end=%08lX\n", mm->env_end );
	seq_printf(m, "   start_stack=%08lX  ", mm->start_stack );
	seq_printf(m, "  down_to=%08lX ", down_to );
	seq_printf(m, " <--- stack grows downward \n" );
	seq_printf(m, "\n" );
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
	return	0;  //SUCCESS
}


static void __exit my_exit(void )
{
	remove_proc_entry( modname, NULL );

	printk( "<1>Removing \'%s\' module\n", modname );
}

module_init( my_init );
module_exit( my_exit );
MODULE_LICENSE("GPL"); 

```

## Virtual Memory Areas

- 在mm_struct中指向一个列表的指针
- 这个指针的名字叫`mmap`

```
struct mm_struct {
	struct vm_area_struct		*mmap;	
	...	
};
```

## VMAs链表

```
struct vm_area_struct {
	unsigned long		vm_start;
	unsigned long		vm_end;
	unsigned long		vm_flags;
	struct vm_area_struct	*vm_next;
	...		
}
```

## 结构关系

![任务内存结构关系](/image/mm_struct.png)

## 编写一个内核模块vma.c查看虚拟内存区链表属性

```
#include <linux/module.h>	// for init_module() 
#include <linux/proc_fs.h>	// for create_proc_read_entry() 
#include <linux/mm.h>		// for 'struct vm_area_struct'
#include <linux/sched.h>	// for 'struct task_struct'
#include <linux/seq_file.h>

char modname[] = "vma";

static int proc_show(struct seq_file *m, void *v)
{
	struct task_struct	*tsk = current;
	struct vm_area_struct	*vma;
	unsigned long 		ptdb;
	int			i = 0;
	
	// display title
	seq_printf(m, "\n\nList of the Virtual Memory Areas " );
	seq_printf(m, "for task \'%s\' ", tsk->comm );
	seq_printf(m, "(pid=%d)\n", tsk->pid );

	// loop to traverse the list of the task's vm_area_structs
	vma = tsk->mm->mmap;
	while ( vma )
		{
		char	ch;	
		seq_printf(m, "\n%3d ", ++i );
		seq_printf(m, " vm_start=%08lX ", vma->vm_start );
		seq_printf(m, " vm_end=%08lX  ", vma->vm_end );

		ch = ( vma->vm_flags & VM_READ ) ? 'r' : '-';
		seq_printf(m, "%c", ch );
		ch = ( vma->vm_flags & VM_WRITE ) ? 'w' : '-';
		seq_printf(m, "%c", ch );
		ch = ( vma->vm_flags & VM_EXEC ) ? 'x' : '-';
		seq_printf(m, "%c", ch );
		ch = ( vma->vm_flags & VM_SHARED ) ? 's' : 'p';
		seq_printf(m, "%c", ch );
		
		vma = vma->vm_next;
		}
	seq_printf(m, "\n" );

	// display additional information about tsk->mm
	ptdb = read_cr3();  
	seq_printf(m, "\nCR3=%08lX ", ptdb );
	seq_printf(m, " mm->pgd=%p ", tsk->mm->pgd );
	seq_printf(m, " mm->map_count=%d ", tsk->mm->map_count );
	seq_printf(m, "\n\n" );

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
	return	0;  //SUCCESS
}


static void __exit my_exit(void )
{
	remove_proc_entry( modname, NULL );

	printk( "<1>Removing \'%s\' module\n", modname );
}

module_init( my_init );
module_exit( my_exit );
MODULE_LICENSE("GPL"); 
```

## 总结

本文主要学习了物理内存的分区和内存寻址，并了解了进程的内存分配数据结构中的内容。

### 更多学习资料

- [哥伦比亚大学课程](https://www.cs.columbia.edu/~junfeng/13fa-w4118/lectures/l20-adv-mm.pdf)
- [逻辑地址、线性地址和物理地址](https://vosamo.github.io/2016/01/VA2PA/)
- [Linux进程地址管理之mm_struct](http://www.cnblogs.com/Rofael/archive/2013/04/13/3019153.html)
