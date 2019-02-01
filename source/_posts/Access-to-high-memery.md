title: 访问高区内存
categories: Linux
date: 2016-12-05 13:55:57
tags: HighMemery
description: Linux HighMemery
---

## 前言

32位系统最大的寻址范围是4GB，通常会将这4GB虚拟内存地址空间分成3GB的用户空间和1GB的内核空间。也就是说内核能直接映射到物理地址的最大范围是1GB的地址（实际上只有896MB直接映射），那么内核怎么访问超过1GB的物理地址？今天我们来学习Linux内核的解决方案。

## Linux内核的解决方案

- 高区内存页框只有在真正需要的时候才会被映射
- 这些页框用完之后会被释放以便其它页框可以重用相同的虚拟地址
- 当然这些共享只适用于高区内存

## 内存映射函数

- `kmap()`函数可以创建一个新的页映射
- `kunmap()`函数可以释放映射

```
#include <linux/highmem.h>
void * kmap( struct page * page );
void kunmap( struct page * page );
```

## 内核空间可视化

- 896MB以下的内存直接映射
- `phys_to_virt()`物理地址转虚拟地址直接加上0xC0000000
- `virt_to_phys()`虚拟地址转物理地址直接减去0xC0000000

## kmap()如何工作

```
void * kmap( struct page * page )
{
	if ( page < highmem_start_page )
		return  page->virtual;
	return  kmap_high( page );
}
```

<!-- more -->

## kmap_high()如何工作

- 内核首先执行一个快速的检查
- 判断高区页是否已经映射
- 如果是则直接返回存在的虚拟地址
- 如果没有则找到一个没有使用的虚拟地址
- 设置映射该页的页表项
- 但是，也有可能找不到未被使用的项

```
void kmap_high( struct page * page )
{
	unsigned long	vaddr;
	spin_lock( &kmap_lock );
	vaddr = (unsigned long)page->virtual;
	if ( !vaddr ) vaddr = map_new_virtual( page );
	++pkmap_count[ (vaddr - PKMAP_BASE)>>12 ];
	spin_unlock( &kmap_lock );
	return	(void *)vaddr;
}
```

- 如果新的内核映射没成功（因为所有的地址已经被使用），那么`map_new_virtual()`函数会阻塞这个任务（进入睡眠）

## kunmap()的目的

- 程序流程

```
void  * vaddr = kmap( page );	  // get address
	… 
/* use this virtual address to access the page */ 
	…
kunmap( page );	       // release the address
```

- 然后内核就可以唤醒被阻塞的任务了

## 引用计数器

- 内核维护着一个计数器数组`static int pkmap_count[ LAST_PKMAP ]`
- 高区内存每一页都有一个计数器
- 计数器的值可以为0,1或者比1大
- 等于0代表页没有被映射
- 等于1表示现在没有被映射，但是曾经被映射
- 大于1表示被映射n-1次

## 刷新TLB

- `kunmap( page )`会减少引用计数
- 当`refcount == 1`，映射就不需要了
- 但是CPU还缓存着映射
- 所以要刷新缓存使其失效
- 但是在多CPU系统中，每个CPU都需要刷新

## 中断程序如何处理

- 中断处理器不能冒险使用`kmap()`，因为其性能低，使用锁会阻塞
- 但是还是需要访问高区内存的数据
- 所以需要一个备用方案在需要时映射数据

## 临时映射kmap_atomic()

- 保留了少数备用的页表项
- 每个CPU只有五六个页表项
- 中断处理器可以安全地使用
- 临时的页映射速度很快，因为它没有锁
- 不保证映射会持久
- 不需要unmapping
- 也不会和其他CPU产生TLB冲突

## 扩展阅读
1. [Wikipedia Highmemery](https://en.wikipedia.org/wiki/High_memory)
2. [HighMemery](https://linux-mm.org/HighMemory)
