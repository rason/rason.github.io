title: 归并排序
categories: Algorithm
date: 2016-06-29 13:22:39
tags: Algorithm
description: 归并排序
---

## 前言

前面学习了选择排序，插入排序和插入排序的改进版希尔排序，但这都属于比较简单的排序方法。今天学习一个比较高级的排序方式——归并排序。曾经听说过一句话：

> 在计算机领域，任何问题都可以通过增加一个虚拟层或者分治的方法来解决。

归并排序就是采用分治法的一个非常典型的应用。

## 归并排序

什么是归并排序？**通俗地描述就是将一个序列分成两份，分别对其进行排序，排序完之后再进行归并。**

不知道这样是否能够理解，反正一开始我是有疑问的。为什么要分开排序再归并，分开排序需要比较，归并的时候又要比较，这不是更加复杂吗？然而，实际上并不是这样的，这只是我们主观臆测的结果，学术上的问题需要有逻辑证明。关于归并排序的性能将在后文中展开讨论。

那么，通过上面概念的描述，我想一般人都无法写出归并排序的代码，甚至连归并排序的过程是怎么样的也不一定能在脑海中浮现。正所谓一图胜千言，归并排序过程如下图所示：

![归并排序](/image/Merge-sort-example-300px.gif)

上图的效果应该是十分直观了，就是将序列不断地分成两份，到最后变成了单个元素与单个元素相比、归并；再利用归并完的序列比较、归并，依次类推。

实现归并排序有两种方式，一种是**递归**，另一种是**迭代**。

<!-- more -->

## 原地归并的抽象方法

在学习递归和迭代实现归并排序之前，我们先思考一下怎么将两个已经有序的子数组原地归并到原来的数组。按照上面的描述，我们可以利用一个辅助的数组来实现，代码如下：

```
private void merge(Comparable[] a,int lo,int mid,int hi){
	
	for (int k = lo; k <= hi; k++) {
		aux[k] = a[k];
	}
	
	int i = lo,j = mid + 1;
	for (int k = lo; k <= hi; k++) {
		if(i > mid)
			a[k] = aux[j++];
		else if(j > hi)
			a[k] = aux[i++];
		else if(less(aux[j], aux[i]))
			a[k] = aux[j++];
		else
			a[k] = aux[i++];
	}
}
```
先将原数组对应位置的元素复制到辅助数组，然后判断进行归并。左边数组用完就直接把右边剩下的按顺序放回原数组中，右边先用完的话就左边按顺序放，两边都没用完就比较谁小谁放。

有了原地归并的抽象方法，进行归并排序就简单了。

## 递归方式实现归并排序

```
public class MergeSort extends Sort {
	private Comparable[] aux;//辅助数组
	private void merge(Comparable[] a,int lo,int mid,int hi){
		//见上方
	}
	
	private void recursiveSort(Comparable[] a,int lo,int hi){
		if(lo >= hi)
			return;
		int mid = lo + (hi - lo)/2;
		recursiveSort(a, lo, mid);
		recursiveSort(a, mid + 1, hi);
		merge(a, lo, mid, hi);
	}
	
	@Override
	void sort(Comparable[] a) {
		aux = new Comparable[a.length];//一次性分配空间
		recursiveSort(a, 0, a.length - 1);
	}

}
```

递归方式的归并排序是一种**自顶向下**的方式，其轨迹如下图所示：

![自顶向下的归并排序中归并结果轨迹](/image/mergesort1)

根据上图和递归的代码可以看出，排序的过程是先递归排序左边部分，然后再递归排序后边部分，再进行总体的归并操作。

## 迭代方式实现归并排序

```
private void iteraterSort(Comparable[] a){
	aux = new Comparable[a.length];
	int len = a.length;
	for (int block = 1; block < len; block *= 2) {
		for (int lo = 0; lo < len - block; lo += 2*block) {
			merge(a, lo, lo + block - 1, Math.min(len - 1, lo + 2*block - 1));
		}
	}
}

```

迭代方式的归并排序是一种**自底向上**的方式，其轨迹如下图所示：

![自底向上的归并排序中归并结果轨迹](/image/mergesort2)

代码中定义了一个block，可以理解为子序列中元素的个数，为两倍递增。也就是先相邻的1个元素进行排序，排序完之后两个为一组和相邻的两个元素进行比较排序；以此类推。

## 归并排序性能比较

在[希尔排序](http://rason.me/2016/06/07/ShellSort/)一文中，我们已经知道希尔排序是比选择排序和插入排序快很多的，已经突破了平方级别。所以这里就不再比较选择排序和插入排序了，为了节约执行时间我们只进行希尔排序与归并排序的比较。

```
public static void main(String[] args) {
	Sort shellSort = new ShellSort();
	Sort mergeSort = new MergeSort();
	int N = 100000,T = 1;
	double t3 =  timeRandomInput(shellSort, N, T);
	double t4 =  timeRandomInput(mergeSort, N, T);
	System.out.println("shellSort time:" + t3);
	System.out.println("mergeSort time:" + t4);
}
```

比较代码继续沿用之前的代码，这里就省略掉了。上面比较的是排序一个长度为10万的数组，多跑几次结果如下：

```
N = 100000

shellSort time:111.0
mergeSort time:80.0

shellSort time:78.0
mergeSort time:113.0

shellSort time:85.0
mergeSort time:71.0

```

结果发现，**当排序的元素在10万个的时候，归并排序与希尔排序的所需时间相差无几，甚至有时候希尔排序还快一点**。但是，当我将N改成100万时，结果就大不相同了：

```
N = 1000000

shellSort time:1297.0
mergeSort time:559.0

shellSort time:1188.0
mergeSort time:549.0

shellSort time:1273.0
mergeSort time:545.0

```

这时候，归并排序显然是比希尔排序要快的多的，并且经我测试，**数据越多的时候，归并排序的优势越明显**。

这是我们实际运行得出来的结论，那么理论上怎么证明？

## 归并排序性能推导

结论1：**对于长度为N的任意数组，归并排序需要1/2NlgN至NlgN次比较。**

我们可以通过下图的树状图来理解上面的结论，这棵树正好有n层(2的n次方等于N)。对于0到n-1之间的任意k，自顶向下的第k层有2的k次方个子数组，每个子数组的长度为2的n-k次方，归并最多需要2的n-k次方比较。因此每层的比较次数为2的k次方乘2的n-k次方=2的n次方，n层总共为n乘2的n次方=NlgN。

![N=16时归并排序中子数组的依赖树](/image/mergesort3)

通过上面的结论我们知道归并排序所需的实际和NlgN成正比。这跟选择排序、插入排序不可同日而语，它表明我们只需要比表里整个数据多个对数因子的实际就能将一个庞大的数组排序。**可以用归并排序处理数百万甚至更大规模的数组，这是插入排序或者选择排序做不到的**。**归并排序的主要缺点是辅助数组所使用的额外空间和N的大小成正比。**

归并排序的改进方式：

- 对小规模数组使用插入排序
- 测试数组是否已经有序，如果a[mid]小于等于a[mid+1]，就已经有序

结论2：**没有任何基于比较的算法能够保证使用少于lg(N!)~NlgN次比较将长度为N的数组排序。**

结论3：**归并排序是一种渐进最优的基于比较排序的算法。**也就是说归并排序在最坏情况下的比较次数和任意基于比较的排序算法所需的最少比较次数都是～NlgN。

但是归并排序的最优性并不代表在实际应用中我们不会考虑其他方式，因为其还有一些局限性：

- 归并排序的空间复杂度不是最优的；
- 在实践中不一定会遇到最坏的情况；
- 除了比较，算法的其他操作（例如访问数组）也可能很重要；
- 不进行比较也能将某些数据排序。

## 总结

本文学习递归和迭代两种方式实现归并排序，一种是自顶向下的方式，一种是自底向上的方式。另外知道了归并排序的最多比较次数是NlgN，是一种渐进最优的基于比较排序的算法。
