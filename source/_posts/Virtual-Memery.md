title: 虚拟内存
categories: OS
date: 2016-09-30 09:08:57
tags: 虚拟内存
description: 虚拟内存
---

## 前言

在[地址空间](http://rason.me/2016/09/27/Address-Space/)这篇文章中，我们知道利用基址寄存器和界限寄存器可以用于创建地址空间的抽象。但是，我们还有一个问题需要解决：管理软件的膨胀。虽然存储器容量增长快速，但是软件的大小增长更快。有些复杂的程序需要的内存比计算机的内存还要大，这显然就无法满足了。即使内存可以满足运行一个程序，可是我们需要的是多道程序并行，之前谈到的使用交换技术其实并不是一种很好的解决方案。因为将一个比如1GB大的程序从磁盘中加载到内存，可能需要10秒甚至更多的时间，这显然是无法接受的。因此，才有了**虚拟内存**技术。

## 虚拟内存

**虚拟内存的基本思想**是：每个程序拥有自己的地址空间，这个空间被分割成多个快，每一个块称作一页或**页面**。每一页有连续的地址范围。这些页被映射到物理内存，但并不是所有的页都必须在内存中才能运行程序。当程序引用到一部分在物理内存中的地址空间时，由硬件立即执行必要的映射。当程序引用到一部分不在物理内存中的地址空间时，由操作系统负责将缺失的部分装入物理内存并重新执行失败的指令。

举个例子，假设有一台可以产生16位地址的计算机，地址范围从0到64K，且这些地址是虚拟地址。然而，这台计算机只有32KB的物理内存，因此，虽然可以编写64KB的程序，但它们却不能完全调入内存运行。所以，在磁盘上必须有一个可以大到64KB的程序核心映像的完整副本，以保证程序片段在需要时能被调入内存。如下图所示：

![页表给出虚拟地址与物理内存地址之间的映射关系](/image/address-mapping)

<!-- more -->

### 分页

上图中利用**分页**技术将64K的虚拟地址分成了16个**页面**,每个页面的地址范围大小都是4K；同样地，物理内存也分成同样大小的单元，称为**页框**。显然，64KB大小的程序不可能全部都映射到物理内存中，最多也只能将一半内存映射到物理内存中。所以，上图中虚拟地址空间中用*X*表示没有存在物理内存中的虚拟地址空间。

首先，我们来看下地址的映射，比如执行下面这条指令：

```
MOV REG, 0
```

指令访问地址0时，从上图可以看到虚拟地址0在虚拟空间索引为0的页面（索引从0开始），这个页面映射到物理内存中索引为2的页框，这个页框地物理地址范围是8K～12K。因此，虚拟地址0就被映射到了物理地址8K，即8192。

再比如，虚拟地址20500，它在虚拟地址空间中索引为5的页面（20K～40K），距离起始范围20K即20480有20的偏移；再看一下虚拟空间索引为5的页面映射到物理内存中索引为3的页框，其起始地址为12K，即12288；因此,虚拟地址20500就会被映射为12288+20=12308。

至此，我想你应该明白了地址是如何映射了，可你应该至少会有以下三个问题：

- **计算机是通过什么将虚拟地址映射为物理内存地址的？**

- **为什么虚拟地址空间中的页面映射到物理内存中的页框不是按顺序的，好像也没有什么规律？**

- **当程序访问不在物理内存的地址空间时，会发送什么情况呢？**

对于第一个问题，计算机是通过一个称为**内存管理单元（Memery Management Unit, MMU）**的模块进行映射的。在没有虚拟内存的计算机上，系统直接将虚拟地址送到内存总线上，读写操作使用具有同样地址的物理内存；而在使用虚拟内存的情况下，虚拟地址不是被直接送到内存总线上，而是被送到MMU中，MMU把虚拟地址映射为物理内存地址。如下图所示：

![MMU的位置和功能](/image/MMU)

对于第二个问题，确实是没按顺序也没什么规律。因为虚拟内存技术并不是将整个程序一次性加载到内存，是分页加载的，即RAM和磁盘之间的交换总是以页面为单元进行的。而且，在加载一个页面的时候，你不能确定内存中哪块页框可用，这都是由CPU决定的，所以并没有顺序和规律。或许CPU有其复杂的规律，这个得看CPU是如何分配内存了。

那么，当程序访问一个未映射的页面时会怎么样，例如执行指令

```
MOV REG, 32780
```

首先，虚拟地址32780位于虚拟地址空间索引为8的页面，将这个地址发送给MMU之后，MMU发现该页面并没有被映射（在图中用X表示，实际硬件中用一个“在/不在”位记录页面在内存中实际存在情况）。于是，CPU陷入到操作系统，这个陷阱称为**缺页中断（page fault）**。操作系统找到一个很少使用的页框且把它的内容写入磁盘（如果它不在磁盘上）。随后把需要访问的页面读到刚才回收的页框中，修改映射关系，然后重新启动引起陷阱的指令。

可见，虚拟内存很适合在多道程序设计系统中使用，**许多程序的片段同时保存在内存中**。当一个程序等待它的一部分读入内存时，可以把CPU交给另一个进程使用。

### 页表

在操作系统中，其实有专门的地方记录进程的地址映射关系，被称为**页表**。从上面的分析可知，页表中的每一项（即**页表项**,记录页面的映射信息）至少得包含两项信息：**页框号和“在/不在”位信息**。即MMU查询页表项，起码要知道映射到物理内存中的哪个页框和是否在内存中有映射。

那么，一个典型的页表项除了上面两项信息还会有什么信息？请看下图：

![一个典型的页表项](/image/page-table-item)

- **保护位**：指出一个页允许什么类型的访问。最简单形式只有一位，0表示读/写，1表示只读。更先进的是使用三位，分别对应是否启用读、写、执行该页面。

- **修改位和访问位**：记录页面的使用情况。在操作系统重新分配页框时非常有用。如果一个页面被修改过，则必须回写磁盘。如果没有，可以简单丢弃，因为磁盘上的副本仍有效。页面在被访问时会设置访问位，在页面置换是不在访问的页面比正在访问的页面更适合淘汰。

- **高速缓存禁止位**：用于禁止该页面被高速缓存。

### MMU工作原理

在了解了页表和页表项之后，我们可以来看一下MMU的内部结构以便了解它是怎么工作的。如下图所示：

![在16个4KB页面情况下MMU的内部操作](/image/MMU-WORK)

上图中输入一个虚拟地址8196（二进制是0010000000000100），输入的16位虚拟地址高4位被用作虚拟页号，低12位作为页面内的偏移量。4位页号可以表示16个页面，12位偏移可以为一页内的全部4096个字节编址。

MMU根据虚拟页号作为索引，找到该虚拟页面的页框号。如果“在/不在”为是0，则将引起一个操作系统陷阱。如果该位是1，则将找到的页表项中的页框号复制到输出寄存器的高3位中，再加上输入虚拟地址中的低12位偏移量。这样就构成了15位的物理地址，输出寄存器的内容随即被作为物理地址送到内存总线。

## 总结

本文简单介绍了虚拟内存技术的出现背景和实现原理，重点分析了利用**分页**技术来实现虚拟内存，MMU通过查询**页表**找到映射关系完成地址转换。
