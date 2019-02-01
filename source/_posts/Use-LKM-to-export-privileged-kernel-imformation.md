title: 使用LKM将特权内核信息展示给用户
categories: Linux
date: 2016-11-07 10:45:22
tags: LKM
description: 使用LKM将受限的内核信息展示给普通用户
---

## 前言

今天的学习目标是编写一个内核模块，访问CMOS内存中的内容，读取出实时时钟的时间。

## 特权内核信息

- 用户通常是禁止看到运行中的内核内部正在发生什么
- 但是我们可以通过内核模块来重写内核数据的普通访问限制
- 最方便的机制就是实现所谓的“/proc”文件系统

## /proc目录

- 除了普通文件存储在硬盘之外，UNIX系统还支持几种特殊类型文件
 - 目录和子目录
 - 符号链接
 - 设备文件
 - 伪文件

- 这些伪文件通常都是在/proc下

## cat命令

- 标准的UNIX命令中提供用户的一个快捷查看/proc文件内容方式
- 这个命令可以将多个文件内容连接起来然后输送到标准输出
- 比如：$ cat /proc/version

## 自己编写一个cat命令

- 要想知道一个命令是如何工作的最好方式就是自己去重新实现一个

```
#include <fcntl.h>	// for open() 
#include <stdio.h>	// for perror() 
#include <unistd.h>	// for read(), write(), close() 

int main( int argc, char *argv[] )
{
	int	i, fd, ch;	// declare local variables

	for (i = 1; i < argc; i++)
		{
		fd = open( argv[i], O_RDONLY );
		if ( fd < 0 ) { perror( argv[i] ); continue; }
		while ( read( fd, &ch, 1 ) == 1 ) 
			write( STDOUT_FILENO, &ch, 1 );
		close( fd );
		}
}
```

- 编译：`$ gcc mycat.c -o mycat`
- 执行：`$ ./mycat <filename>`
-当然这只是一个最简单的实现，需要实现支持更多可选参数可以参考手册：`$ man cat`

<!-- more -->

## 创建我们自己的/proc文件

- 我们可以编写代码实现属于自己的伪文件，位于/proc目录下
- 今天要实现的功能就是读取CMOS内存中的实时时钟信息
- CMOS内存中的前十个字节保存的就是时间日期信息，如下图所示
- 通过索引端口0x70和数据端口0x71来进行读取

![十个时钟日期字节](https://raw.githubusercontent.com/rason/rason.github.io/master/image/clock-calendar-bytes.png)

## cmos.c

```
#include <linux/module.h>	// for init_module() 
#include <linux/proc_fs.h>	// for create_proc_info_entry() 
#include <asm/io.h>		// for inb(), outb()
#include <linux/seq_file.h>

char modname[] = "cmos";
unsigned char  cmos[10];
char *day[] = { "", "MON", "TUE", "WED", "THU", "FRI", "SAT", "SUN" };
char *month[] = { "", "JAN", "FEB", "MAR", "APR", "MAY", "JUN",
			"JUL", "AUG", "SEP", "OCT", "NOV", "DEC" };

static int proc_show(struct seq_file *m, void *v) {

	int	i;
	int	month_index;

	// input and store the first ten CMOS entries
	for (i = 0; i < 10; i++)
	{
		outb( i, 0x70 );
		cmos[i] = inb( 0x71 );
	}

	// show the current time and date
	seq_printf(m, "\n\t CMOS Real-Time Clock/Calendar:" );
	seq_printf(m, " %02X", cmos[4] );	// current hour
	seq_printf(m, ":%02X", cmos[2] );	// current minutes
	seq_printf(m, ":%02X", cmos[0] );	// current seconds
	seq_printf(m, " on" );
	seq_printf(m, " %s, ", day[ cmos[6] ] ); 	// day-name
	seq_printf(m, "%02X", cmos[7] );		// day-number

	// convert 'cmos[ 8 ]' from BCD-format to integer-format
	month_index = ((cmos[ 8 ] & 0xF0)>>4)*10 + (cmos[ 8 ] & 0x0F);
	seq_printf(m, " %s", month[ month_index ] );	// month-name

	seq_printf(m, " 20%02X ", cmos[9] );		// year-number
	seq_printf(m, "\n" );

	seq_printf(m, "\n" );
	
	return 0;
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

- 编译：需要上篇博客的Makefile文件，执行`$ make`
- 安装：`$ sudo /sbin/insmod cmos.ko`
- 查看：使用上面自己实现的mycat命令，`$ ./mycat /proc/cmos`
- 本机输出结果：`CMOS Real-Time Clock/Calendar: 05:53:11 on MON, 07 NOV 2016 `


完。
