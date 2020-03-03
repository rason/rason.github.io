title: 一个查看所有内存的字符设备驱动
categories: Linux
date: 2016-11-14 15:52:02
tags: 字符设备驱动
description: 字符设备驱动
---

## 前言

用户程序是无法访问直接访问物理内存的，使用的地址也只是逻辑地址，而不是物理地址。如果我们想直接通过物理地址来访问内存，可以通过内核模块来实现。

## 物理内存分区

假设一台32位的计算机，2G的内存，会分为如下三个区域：

![物理内存分区](/image/ram-zone.png)

### 遗留DMA

- 许多旧的设备依赖PC/AT的DMA控制器来进行数据传输
- 这些芯片只能使用24位的地址
- 所以这些设备只能访问到16MB的物理内存
- 因此Linxu将ZONE_LOW区域保留使用

### Normal zone

- 该区域的范围从16MB到896MB
- 该区域的低位部分保存了一个重要的数据结构用来跟踪物理内存的使用情况
- 该数据结构就是**mem_map[]**
- 该区域的剩余空间可以让操作系统内核动态地开辟空间

### High zone

- 32位计算机虚拟地址空间为4G，一般低3G为用户空间，高1G位内核空间
- 如果物理内存为1G，则内核可以直接映射
- 如果物理内存大于1G，则无法进行直接映射
- 所以内核要访问物理内存的高位区域，需要进行临时的映射，如下图

![物理内存高位区域映射](/image/kmap.png)

<!-- more -->

## 字符设备驱动

```
dram.c

#include <linux/module.h>	// for module_init() 
#include <linux/highmem.h>	// for kmap(), kunmap()
#include <linux/mm.h>		// for get_num_physpages()
#include <asm/uaccess.h>	// for copy_to_user() 

char modname[] = "dram";	// for displaying driver's name
int my_major = 85;		// note static major assignment 
loff_t 	dram_size;		// total bytes of system memory

loff_t my_llseek( struct file *file, loff_t offset, int whence );
ssize_t my_read( struct file *file, char *buf, size_t count, loff_t *pos );

struct file_operations 
my_fops =	{
		owner:		THIS_MODULE,
		llseek:		my_llseek,
		read:		my_read,
		};

static int __init dram_init( void )
{
	printk( "<1>\nInstalling \'%s\' module ", modname );
	printk( "(major=%d)\n", my_major );
	
	dram_size = (loff_t)get_num_physpages() << PAGE_SHIFT;
	printk( "<1>  ramtop=%08llX (%llu MB)\n", dram_size, dram_size >> 20 );
	return 	register_chrdev( my_major, modname, &my_fops );
}

static void __exit dram_exit( void )
{
	unregister_chrdev( my_major, modname );
	printk( "<1>Removing \'%s\' module\n", modname );
}

ssize_t my_read( struct file *file, char *buf, size_t count, loff_t *pos )
{
	struct page	*pp;
	void		*from;
	int		page_number, page_indent, more;
	
	// we cannot read beyond the end-of-file
	if ( *pos >= dram_size ) return 0;

	// determine which physical page to temporarily map
	// and how far into that page to begin reading from 
	page_number = *pos / PAGE_SIZE;
	page_indent = *pos % PAGE_SIZE;
	
	// map the designated physical page into kernel space
	pp = pfn_to_page( page_number);
	from = kmap( pp ) + page_indent;
	
	// cannot reliably read beyond the end of this mapped page
	if ( page_indent + count > PAGE_SIZE ) count = PAGE_SIZE - page_indent;

	// now transfer count bytes from mapped page to user-supplied buffer 	
	more = copy_to_user( buf, from, count );
	
	// ok now to discard the temporary page mapping
	kunmap( pp );
	
	// an error occurred if less than count bytes got copied
	if ( more ) return -EFAULT;
	
	// otherwise advance file-pointer and report number of bytes read
	*pos += count;
	return	count;
}

loff_t my_llseek( struct file *file, loff_t offset, int whence )
{
	loff_t	newpos = -1;

	switch( whence )
		{
		case 0: newpos = offset; break;			// SEEK_SET
		case 1: newpos = file->f_pos + offset; break; 	// SEEK_CUR
		case 2: newpos = dram_size + offset; break; 	// SEEK_END
		}

	if (( newpos < 0 )||( newpos > dram_size )) return -EINVAL;
	file->f_pos = newpos;
	return	newpos;
}

MODULE_LICENSE("GPL");
module_init( dram_init );
module_exit( dram_exit );
```

- 编译：`$ make`
- 安装：`$ sudo /sbin/insmod dram.ko`
- 创建设备文件：`$ sudo mknod  /dev/dram  c  85  0`



完。
