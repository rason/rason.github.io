title: '选择排序'
categories: Algorithm
date: 2016-06-02 08:40:53
tags: 排序
description: 选择排序
---

## 前言

最近这段时间工作有点忙，博客的更新频率放缓了一点，以后还是大约每月更新四篇比较符合节奏。接下来的时间打算简单地学习一下算法，说实话，由于大学时学习的专业是信息工程，属于计算机相关专业。在算法这方面的知识还是十分的薄弱，而且在工作上也并没有刻意地去学习或用到。鉴于成为一名优秀的工程师，算法是必然要掌握的，所以还是补补这方面的知识。

## 选择排序

**什么是选择排序？**就是依次地从后面剩下的元素中取出最小的值，替换第一个、第二个、……，直到元素末尾，排序就完成了。

看到这个概念，你会怎么编码实现这个功能？先不要看下面的答案。下面是我第一时间想出来的方案：

```
package com.rason.algs4.sort;

public class SelectSort {

	public static int[] sort(int[] a){
		for (int i = 0; i < a.length; i++) {
			int min = a[i];
			for (int j = i + 1; j < a.length; j++) {
				if(a[j] < min){
					int tmp = a[j];
					a[i] = tmp;
					a[j] = min;
					min = a[i];
				}
			}
		}
		return a;
	}
	
	public static void main(String[] args) {
		int[] a = {3,1,4,7,5,8,6};
		int[] sortA = sort(a);
		for (int i = 0; i < sortA.length; i++) {
			System.out.println(sortA[i]);
		}
		
	}

}

```

我的思路是，外循环取出第一个数，然后内循环剩下的数，然后和第一个数比较，如果比第一个数小，则交换它们的位置。编写了测试代码，成功地实现了排序。下面是标准解决方案：

<!-- more -->

```
package com.rason.algs4.sort;

public class SelectionSort extends Sort {

	@Override
	void sort(Comparable[] a) {
		int N = a.length;
		for (int i = 0; i < N; i++) {
			int min = i;
			for (int j = i + 1; j < N; j++) {
				if(less(a[j], a[min]))
					min = j;
			}
			exch(a, i, min);
		}
	}
	
	public static void main(String[] args){
		InsertSort insertSort = new InsertSort();
		Integer[] a = {3,1,4,7,5,8,6};
		insertSort.sort(a);
		insertSort.show(a);
	}

}

```

第一眼看标准的解法，会有一点疑惑，因为思维已经被自己的解法困住了一下。然后再看一遍，发现标准的解法比我的解法优秀很多。标准的解法虽然思路好像是一致的，但又存在不同的地方，是因为它使用的是数组的索引。循环剩下的元素，找出最小值的索引，循环完毕之后最小值的索引就保存在min中了，最后才和第i个元素互换。并不像我那样，找出了一个比第i个小的就立即互换位置，这样做了很多无用的互换操作，其实直接互换最小那个就行了。

上面标准的解法继承了Sort父类，这是为了各种算法公用某些方法而抽象出来的抽象类，代码如下：

```
package com.rason.algs4.sort;

public abstract class Sort {
	
	abstract void sort(Comparable[] a);
	
	
	boolean less(Comparable v,Comparable w){
		return v.compareTo(w) < 0;
	}
	
	void exch(Comparable[] a,int i,int j){
		Comparable t = a[i];
		a[i] = a[j];
		a[j] = t;
	}
	
	void show(Comparable[] a){
		for (int i = 0; i < a.length; i++) {
			System.out.println("a["+ i +"]=" + a[i]);
		}
	}
	
	boolean isSort(Comparable[] a){
		for (int i = 1; i < a.length; i++) {
			if(less(a[i], a[i - 1]))
				return false;
		}
		return true;
	}

}

```

## 选择排序性能分析

先自己思考一下，**对于一个长度为N的数组，进行选择排序需要进行多少次的比较和多少次的互换？**

...
...
...

对于第一个元素，是不是要比较剩下的N-1个元素？
对于第二个元素，是不是要比较剩下的N-2个元素？
...
对于第N-1个元素，是不是要比较剩下的1个元素？

因此，总比较的次数是1+2+3+....+(N-1)次，即（N-1）*N/2次，互换的次数假如每次都要互换的话是不是就是N次？

所以，得出结论：**对于长度为N的数组，选择排序需要大约N*N/2次比较和N次交换。**

## 选择排序的特点

总的来说，选择排序是一种容易理解和实现的简单排序算法，它有两个很鲜明的特点。

**1.运行时间和输入无关。**这里的输入无关指的是，无论你的输入是一个已经有序的数组或者主键全部相同的数组，甚至是一个随机的数组，只要它们的长度一样，你会发现排序的时间是一样长的。在以后的学习中，你会发现其它的算法会更善于利用输入的初始状态。

**2.数据移动是最少的。**每次交换都会改变两个数组元素的值，因此选择排序用了N次交换——交换次数和数组的大小是*线性关系*。在以后研究的其它算法都不具备这个特征（大部分的增长数量级都是线性对数或是平方级别）。

## 总结

由于篇幅关系，原打算一起讨论的插入排序就放到下一篇文章了，在下一篇文章中会进行选择排序和插入排序的性能比较。
