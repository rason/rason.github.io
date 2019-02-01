title: 子数组的最大值
categories: LeetCode
date: 2016-06-20 13:19:46
tags: 子数组的最大值 Kadane算法
description: 子数组的最大值 Kadane算法
---

## 前言

在Leetcode刷题的时候遇到一个求连续子数组的最大值问题，发现了一个比较有趣的算法，写篇博客记录一下。原题目如下：

> Find the contiguous subarray within an array (containing at least one number) which has the largest sum.
For example, given the array [−2,1,−3,4,−1,2,1,−5,4],
the contiguous subarray [4,−1,2,1] has the largest sum = 6.

## 思考

拿到问题，先自己思考一下怎么做。

...
...
...

我大概思考了两分钟，并没有想出什么好的方式。就是正常人都能想到的，把所有情况都比较一下，元素为1个的时候，元素为2个的时候...，找出最大值。于是，我就花了几分钟时间，写出了以下代码：

```
package com.rason.leetcode.five;

public class Solution {
	public static int maxSubArray(int[] nums) {
		if(nums.length == 0)
			return 0;
		
		int max = nums[0];
		for(int count = 1; count <= nums.length; count ++){
			int range = nums.length - count + 1;
			for (int i = 0; i < range; i++) {
				int sum = sum(nums,i,count);
				if(sum > max){
					max = sum;
				}
			}
		}
		
		return max;
    }
	
	public static int sum(int[] nums,int index,int count){
		int sum = 0;
		for(int i = 0; i < count;i++){
			sum += nums[index + i]; 
		}
		return sum;
	}
	
	public static void main(String[] args){
		int[] nums = {-2,1,-3,4,-1,2,1,-5,4};
		System.out.println(maxSubArray(nums));
	}
}

```

运行一下，结果输出正确。我心里感觉挺愉快的，虽然方法有点笨，但是我能得出正确的结果，说明我笨得有出息哈哈哈。于是，我愉快地点了`Leetcode`上的`submit solution`,提示如下：

```
Time Limit Exceeded

Last executed input:
[-57,9,-72,-72,-62,45,-97,24,-39,35,-82,-4,-63,1,-93,42,44,1,-75,-25,-87,-16,9,-59,20,5,-95,-41,4,-30,47,4.....]
```

悲剧发生了，当输入一个特别大的数组的时候，由于运算时间太长被拒绝了。果然，算法题不可能那么暴力地得出结果，我又思考了几分钟，依然没有想出有什么更好的方法。于是，我就去看了一下讨论区别人的解法，发现一个相当了不起的算法——**Kadane算法**。

<!-- more -->

## Kadane算法

关于Kadane算法，我不想用一些定义性的文字来说明。我们试想一下，怎么能得到连续数组的最大值？继续沿用我之前的学习方法，**先做最简单的假设，然后再到普通的情况**。

举个例子，对于一个只有两个元素的数组，比如：[-2,1]。

对于这个简单的数组，我们是不是可以一眼看出它的连续子数组的最大值是1。为什么我们能一眼看出来？因为-2+1=-1比1少，然后就是-2和1了，当然最大值是1。这当中存在的道理是什么？

**当一个负数加上一个任何数，是不是都比这个任何数都小？**即-2+1是不是比1小，所以连续i个元素的子数组是最大值的话，那么该子数组的前i-1个元素之和不可能为负数。如果前i-1个元素之和为负数，加上第i个元素之和，肯定比第i个元素的值小，那么我们是不是就可以丢弃前i-1个元素了？**这就是Kadane算法的原理**。

如果你还是不明白，我们来看一下文章开头题目中的数组：[−2,1,−3,4,−1,2,1,−5,4]。

我们从左往右看，第一个元素是-2，第二个元素是1，所以-2可以抛弃掉，目前最大值是1；
下一个元素是-3,1+（-3）=-2，再下一个元素是4，所以1+（-3）=-2可以抛弃，现在的最大值是4；
下一个元素是-1,4+（-1）=3，现在是正数，所以暂时不抛弃，但是3比上一步中的4小，所以目前的最大值还是4；
再下一个元素是2，4+（-1）+2=5,还是正数，不抛弃，目前最大值是5；
再下一个元素是1，4+（-1）+2+1=6，还是正数，不抛弃，目前最大值是6；
再下一个元素是-5，4+（-1）+2+1+（-5）=1，正数，不抛弃，但是加起来是1，所以最大值还是上一步中的6；
最后一个元素是4，4+（-1）+2+1+（-5）+4=5，数组结束，并没有比6大，所以最大值是6；加入最后一个元素是大于5的数，那么就比6大了，所以加起来是正数的不要抛弃，还要看下一个数的大小，有可能加上下一个数就是最大值。

以上，就是Kadane算法的实现逻辑了，那么用代码怎么表示？先根据上述文字，自己试着写代码实现一下Kadane算法。

...
...
...

根据以上的描述，实现起来应该不困难，以下是实现代码：

```
public class Solution2 {
	public static int maxSubArray(int[] nums) {
		int maxSoFar = nums[0],maxEndingHere = nums[0];
		for (int i = 1; i < nums.length; i++) {
			maxEndingHere = Math.max(maxEndingHere + nums[i], nums[i]);
			maxSoFar = Math.max(maxEndingHere, maxSoFar);
		}
		return maxSoFar;
    }
	
	public static void main(String[] args){
		int[] nums = {-2,1,-3,4,-1,2,1,-5,4};
		System.out.println(maxSubArray(nums));
	}
}
```

这里主要理解两个变量，一个是maxSoFar代表到目前位置的最大值，另一个是maxEndingHere表示到这里结束的最大值，有什么不同呢？请看上面推导描述的第三步，4就是到目前位置的最大值（maxSoFar），3就是到这里结束的最大值（maxEndingHere），这里的maxEndingHere有可能比maxSoFar小的，因为你不知道加上下一个数之后是否会比maxSoFar大，所以也要保存起来，就比如上述推导的中最后一步的假设。

以上，就是用我自己语言表达的Kadane算法，不知道我是否表达清楚。如果你还是不理解，那就需要自己认真琢磨一下了，一开始会有点难以理解，但是理解之后会发现这个一个特别美妙的解法。

## 总结

本文介绍的是利用Kadane算法求解连续子数组的最大值，要理解其关键原理：**当一个负数加上一个任何数，都比这个任何数都小**，所以连续子数组前面几个元素加起来是负数的话那就得丢弃掉了。
