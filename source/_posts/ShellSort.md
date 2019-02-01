title: 希尔排序
categories: Algorithm
date: 2016-06-07 16:43:51
tags: 排序
description: 希尔排序
---

## 前言

上一篇文章讲了一下[插入排序](http://rason.me/2016/06/02/Insertion/),试想一下，假设有一个很小的值排在很后面，那么这个值是不是要交换很多次才能排到前面？因为插入排序每次都是交换相邻的两个元素。那么，有没有办法优化一下插入排序，令那些小的元素很快地交换到了前面？这就是今天要讲的*希尔排序*。

## 希尔排序

**什么是希尔排序？**前言中谈到，插入排序每次只是交换相邻（步伐为1）两个元素，所以才会交换很多次。那么，是不是只要我们将步伐扩大，是不是很快就交换到前面了？比如说一个数组：16,15,14,13,12,11,10,9,8,7,6,5,4,3,2,1。我们以4为步伐，即16,12,8,4为一组、15,11,7,3为一组、14,10,6,2为一组、13,9,5,1为一组,将四组分别进行插入排序，这样就可以将较少的元素尽快地交换到较前面的地方。但是对这几个组分别进行一次的插入排序，整个数组还是无法达到最终的排序结果。那现在该怎么办？这里先留个悬念，我们先把步伐为4的排序用代码实现出来。

先不要往下看，按照上述的描述，自己先动手写一写，写错了或者写的不那么完美没关系，我自己也没写正确哈哈哈。下面是我第一时间想到的方式：

```
package com.rason.algs4.sort;

public class ShellSort extends Sort {

	@Override
	void sort(Comparable[] a) {
		int h =4;
		for (int k = 0; k < h; k++) {
			for (int i = h + k; i < a.length; i += h) {
				for (int j = i; j > k && less(a[j], a[j - h]); j -= h) {
					exch(a, j, j - h);
				}
			}
		}
	}

	public static void main(String[] args){
		ShellSort insertSort = new ShellSort();
		Integer[] a = {16,15,14,13,12,11,10,9,8,7,6,5,4,3,2,1};
		insertSort.sort(a);
		insertSort.show(a);
	}
}

```

<!-- more -->

我个人比较笨，先从特殊情况做起，我就假设这个数组是固定的16个元素，步伐就是固定为4。那么我就将插入排序修改一下，将i++改成i+h，这里的h就是我们的步伐，然后a[j]与a[j-h]比较，如果小的话就交换位置。我做了三个循环，最里边两个就是我们的插入排序，最外面一层循环就是循环上面提到的分成步伐为4的四个数组。我跑了一下代码，输出结果如下：

```
a[0]=4
a[1]=3
a[2]=2
a[3]=1
a[4]=8
a[5]=7
a[6]=6
a[7]=5
a[8]=12
a[9]=11
a[10]=10
a[11]=9
a[12]=16
a[13]=15
a[14]=14
a[15]=13

```

发现输入结果是正确的，较小的值虽然没达到自己最终正确的位置，但是已经达到了比较前面的位置，而且发现以4为步伐的几个元素都已经实现了排序。后来我对比了一下标准的希尔排序算法，发现自己的代码还是有点多余，为什么呢？因为我用了三层循环，其实不需要，我们可以对比一下下面的代码：

```
void sort(Comparable[] a) {
	int h =4;
	for (int i = h; i < a.length; i++) {
		for (int j = i; j >= h && less(a[j], a[j - h]); j -= h) {
			exch(a, j, j - h);
		}
	}
}
```
通过简单的修改之后，少了一层循环，输出结果一样。我自己写的代码是简单地先将四个数组之中的一个排序完，再进行第二个数组的排序，而标准的解法是并行进行的。虽然我的代码比较烂，但是还是达到了最终的效果哈哈。

**然而，这并不是我们最终的排序效果啊。**怎么办？先回想一下我们将步伐调为4的出发点，是不是因为较少的数组排在较后面？既然我们现在通过一次步伐为4的排序后，较少的值已经在相对较前面的地方了，那么我们是不是直接对现在这个数组在进行一次标准的（步伐为1）插入排序就行了？好，那么我们试一下：

```
void sort(Comparable[] a) {
	int[] jump = {4,1};
	for (int k = 0; k < jump.length; k++) {
		int h = jump[k];
		for (int i = h; i < a.length; i++) {
			for (int j = i; j >= h && less(a[j], a[j - h]); j -= h) {
				exch(a, j, j - h);
			}
		}
	}
}
```
上述的代码我们设置了两个步伐，一个为4，一个为1，先对数组进行步伐为4的插入排序，然后再对数组进行步伐为1的插入排序。步伐为1的插入排序是不是就是我们上篇文章说到的[插入排序](http://rason.me/2016/06/02/Insertion/)？所以肯定能对数组进行最终的排序。输出结果如下：

```
a[0]=1
a[1]=2
a[2]=3
a[3]=4
a[4]=5
a[5]=6
a[6]=7
a[7]=8
a[8]=9
a[9]=10
a[10]=11
a[11]=12
a[12]=13
a[13]=14
a[14]=15
a[15]=16

```

奇迹出现了，排序完成了哈哈哈。

...
...
...

等等，好像总感觉缺了点什么，感觉有点不对，代码那么奇葩怎么能称之为希尔排序呢。而且认真想一下，如果数组很大呢，步伐为4是不是也要交换很多次，那么数组很大我们是不是可以将步伐设置得更大一点。另外，从一个更大的步伐排序完之后，还是有挺多小的元素排在较后的位置呢，此时直接将步伐变成1再排序一次是不是有点欠妥。我们是不是可以类似地将步伐设置为类似8,4,2，1这样排序达到最终的排序，就是先设置步伐为8，排序完再以步伐为4排序，然后再以步伐为2排序，最终以步伐为1达到最终的排序。

对的，对的，那么这个8,4,2,1这样的数列怎么得出来才合理呢？我不可以设置为9,3,1这样子吗？实际上，要回答这个问题并不简单。算法的性能不仅取决于h，还取决于h之间的数学性质，比如它们的公因子等。有很多论文研究了各种不同的数列，但都无法证明某个数列是“最好的”。所以，我们可以设计一个简单的数列，比如下面这样：

```
void sort(Comparable[] a) {
	int N = a.length;
	int h = 1;
	while(h < N/3)
		h = h * 3 + 1; // 1,4,13,,40,121,...
	
	while(h >= 1){
		for (int i = h; i < a.length; i++) {
			for (int j = i; j >= h && less(a[j], a[j - h]); j -= h) {
				exch(a, j, j - h);
			}
		}
		h = h/3;
	}
}
```

上面的算法就是按照 1,4,13,,40,121,...这样的步伐从大到小进行插入排序，到最终的步伐为1时排序完成。代码中第一个while循环就是根据数组长度得出我们开始循环插入排序的最大步伐，第二个while循环就是不断地将步伐/3进行插入排序，最终步伐h为1时排序完成结束循环。

实践证明上面的简单步伐数列和复杂的数列性能接近，但复杂的数列在最坏的情况下性能要好于我们使用的简单数列，更多优秀的数列有待我们去发现。

## 希尔排序的性能

继续沿用我们上篇文章的SortCompare代码进行比较，如下：

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
		Sort shellSort = new ShellSort();
		int N = 1000,T = 100;
		double t1 =  timeRandomInput(selectionSort, N, T);
		double t2 =  timeRandomInput(insertSort, N, T);
		double t3 =  timeRandomInput(shellSort, N, T);
		System.out.println("selectionSort time:" + t1);
		System.out.println("insertSort time:" + t2);
		System.out.println("shellSort time:" + t3);
	}

}

```

多次运行输入结果如下：

```
selectionSort time:133.0
insertSort time:155.0
shellSort time:34.0

selectionSort time:194.0
insertSort time:266.0
shellSort time:40.0

selectionSort time:135.0
insertSort time:171.0
shellSort time:31.0

```

发现**希尔排序比选择排序和插入排序都要快的多,并且数组越大，优势越大。**对于上述希尔排序的性能，目前最重要的结论是**它的运行实践达不到平方级别。**由插入排序到希尔排序，一个小小的改变就突破了平方级别的运行时间的屏障。这正是许多算法设计问题想要达到的目标。

在输入随机排序数组的情况下，我们在数学上还不知道希尔排序所需要的平均比较次数。人们发明了很多步伐数列来渐进式地改进最坏情况下所需的比较次数（N的4/3次方，N的5/4次方，N的6/5次方...），但这些结论大多只有学术意义，因为对于实际应用中的N来说它们的步伐数列生成函数之间的区别并不明显。所以，我们上述的简单步伐数列方式已经基本够用了。

对于中等大小的数组可以选择希尔排序，它的运行实际还可以接受，代码量也少，而且不需要使用额外的内存空间。

## 总结

**一句话，希尔排序其实就是不断地改变步伐的插入排序，知道步伐为1排序就完成了。**

借用网络上了一张动图，展示希尔排序的过程。
![希尔排序](https://raw.githubusercontent.com/rason/rason.github.io/master/image/shell-sort-animation.gif)
