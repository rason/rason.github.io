title: 插入排序
categories: Algorithm
date: 2016-06-02 10:29:25
tags: 排序
description: 插入排序
---

## 前言

趁热打铁，上篇写了[选择排序](http://rason.me/2016/06/02/Selection/)，继续把插入排序简单总结一下。

## 插入排序

**什么是插入排序？**通常我们在打牌的时候，整理牌的方法是一张一张来，将每一张牌插入到其它已经有序的牌中的适合位置。假设我们从左到右整理，第一张不用整理，然后看第二张，如果比第一张小，就会放到第一张前面；然后再看第三张，和第二张比较如果比它大就不用管了，如果比它小，就跟第二张交换一下位置，然后第二张再跟第一张比较，如果比第一张小，则还要和第一张交换位置；依次进行到最右边的，则排序完成。

乍一看，跟上篇选择排序有点相像，**选择排序是取出来跟后面剩下的元素比较，而插入排序是取出来后跟前面的元素比较**，但实际操作上又有所不同。因为选择排序每次外循环最多只交换一次，而插入排序中，如果是一张比较少的牌，可能会依次地和前面的好多张牌都交换位置，就相当于很多牌都往后移动了一个位置。

那么，从上面的描述之中，先自己动手写一个插入排序的代码实现，先不要看下面的标准答案。我自己根据定义，写出的代码如下：

```
package com.rason.algs4.sort;

public class InsertSort extends Sort {

	@Override
	void sort(Comparable[] a) {
		for (int i = 0; i < a.length; i++) {
			for (int j = i; j > 0; j--) {
				if(less(a[j], a[j - 1])){
					exch(a, j, j - 1);
				}
			}
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

<!-- more -->

对比了标准的答案，实现的方法是一样的，只是if的判断提到了for语句中，如下所示：

```
for (int j = i; j > 0 && less(a[j], a[j - 1]); j--) {
	exch(a, j, j - 1);
}
```

连我都能写出来，这说明这个算法的实现还是十分的简单的。

## 插入排序的性能分析

同样地，先自己思考一下，**对于一个长度为N的数组，进行插入排序需要进行多少次的比较和多少次的互换？**

...
...
...

我们假设，这个数组刚好是全部都倒过来的，就是说我们每次都要比较，每次都要交换。

对于第一个元素，不用管，从第二个元素开始。
第二个元素，比较一次，交换一次。
第三个元素，比较两次，交换两次。
...
第N个元素，比较N-1次，交换N-1次。

所以，最悲观的情况是，比较1+2+3+...+(N-1)，即（N-1）*N/2次，交换1+2+3+...+(N-1)，即（N-1）*N/2次。

那么，最乐观的情况当然是这个数组本来就是排好序的了，会比较N-1次，交换0次。

**结论一：对于一个随机排序的长度为N且主键不重复的数组，平均情况下插入排序需要（N-1）*N/4次比较以及（N-1）*N/4次交换。**

...
...
...

我们需要考虑的更一般的情况是*部分有序*的数组。首先，我们先理解一下什么叫做倒置。*倒置*指的是数组中的两个顺序颠倒的元素，比如3,1,4,7,5,8,6中有4对倒置：3-1,7-5,7-6,8-6。

**如果数组中倒置的数量小于数组大小的某个倍数，那么我们说这个数组是部分有序的。**下面是几种典型的部分有序的数组：

- 数组中每个元素距离它的最终位置都不远；
- 一个有序的大数组接一个小数组；
- 数组中只有几个元素的位置不正确。

插入排序对这样的数组很有效，而选择排序则不然。事实上，当倒置的数量很少时，插入排序很可能比其它算法都要快。

**结论二：插入排序需要的交换操作和数组中倒置的数量相同，需要的比较次数大于等于倒置的数量，小于等于倒置的数量加上数组的大小再减一。**

先思考一下，为什么能得出结论二。先不要看下面的答案。

...
...
...

是不是每次交换都改变了两个顺序颠倒的元素的位置，相当于减少了一对倒置？当倒置数量为0时，排序就完成了。
每次交换都对应着一次比较，且1到N-1之间的每个i都可能需要一次额外的比较（在a[i]没有达到数组的左端时）。
所以，就有了结论二。

总的来说，插入排序对于部分有序的数组十分高效，也很适合小规模数组。这很重要，因为这些类型的数组在实际应用中经常出现，而且它们也是高级排序算法的中间过程。

## 选择排序与插入排序性能比较

首先，我们先估算一下这两种方法的性能。

通过前一篇文章和上面的分析，对于随机排序数组，两者的运行实际都是平方级别的。也就是说，在这种输入下插入排序的运行时间和N的平方乘以一个小常数成正比，选择排序的运行时间和N的平方乘以另一个小常数成比例。这两个常数的值取决于所使用的计算机中比较和交换元素的成本。对于许多数据类型和一般的计算机，可以假设这些成本是相近的。因此，可以得出以下猜想：

**对于随机排序的无重复主键的数组，插入排序和选择排序的运行时间是平方级别的，两者之比应该是一个较少的常数。**

下面我们用一个测试案例来比较一下两者的性能：

```
package com.rason.algs4.sort;

import edu.princeton.cs.algs4.StdRandom;

public class SortCompare {
	
	public static double time(Sort alg,Double[] a){
		long start = System.currentTimeMillis();
		alg.sort(a);
		long end = System.currentTimeMillis();
		return end - start;
	}
	
	/**
	 * 排序T个长度为N的数组，返回排序时间
	 * @param alg 
	 * @param N
	 * @param T
	 * @return
	 */
	public static double timeRandomInput(Sort alg,int N,int T){
		double total = 0.0;
		Double[] a = new Double[N];
		for (int i = 0; i < T; i++) {
			for (int j = 0; j < N; j++) {
				a[j] = StdRandom.uniform();
			}
			total += time(alg, a);
		}
		return total;
	}
	
	public static void main(String[] args) {
		Sort selectionSort = new SelectionSort();
		Sort insertSort = new InsertionSort();
		int N = 1000,T = 100;
		double t1 =  timeRandomInput(selectionSort, N, T);
		double t2 =  timeRandomInput(insertSort, N, T);
		System.out.println("selectionSort time:" + t1);
		System.out.println("insertSort time:" + t2);
		System.out.println("selectionSort/insertSort :" + t1/t2);
	}

}

```

上面的测试案例分别测试了两种排序方法，排序T个长度为N的数组所需的总时间。在我的机器上跑了多次，结果如下

```
selectionSort time:128.0
insertSort time:156.0
selectionSort/insertSort :0.8205128205128205

selectionSort time:186.0
insertSort time:169.0
selectionSort/insertSort :1.1005917159763314

selectionSort time:137.0
insertSort time:163.0
selectionSort/insertSort :0.8404907975460123

selectionSort time:137.0
insertSort time:167.0
selectionSort/insertSort :0.8203592814371258

```
我们还可以修改N和T的值，跑了很多次，发现有时候是插入比选择快，有时候是选择比插入快，不知道你的机器是怎么样的，但是两者之比应该是一个较少的常数。

## 总结

和选择排序一样，插入排序也是一种比较简单的算法，先从简单的算法学起，掌握其分析思路才是重点。
