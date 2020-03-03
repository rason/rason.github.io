title: 符号表
categories: Algorithm
date: 2016-07-21 15:56:47
tags: 符号表
description: 符号表
---

## 前言

前段时间学习了排序算法，接下来的一段时间将学习查找算法。**符号表**描述一张抽象的表格，我们会将信息（值）存储在其中，然后按照指定的**键**来搜索并获取这些信息。怎么才能高效地实现符号表的**插入**和**查找**,就是我们学习的重点。

## 无序符号表API

当然，在实现插入和查找功能之前，我们先来认识一下符号表的定义和设计其API。作为一位有经验的程序员，在实际编码之前，我们要先对问题进行仔细的分析和设计，然后才是编码测试。将问题思考分析清楚，将会避免在编码上耗费太多的时间。

> **定义**：符号表是一种存储键值对的数据结构，支持两种操作：插入（put），即将一组新的键值对存入表中；查找（get），即根据给定的键得到相应的值。

根据定义，我们设计出简单的泛型符号表API如下：

| public class   | ST<Key, Value\>             |    |
| :------------: |:--------------------------:| :-----:|
|                | ST()                       | 创建一张符号表 |
| void           | put(Key key, Value value)  | 将键值对存入表中（若值为空则将键key从表中删除） |
| Value          | get(Key key)               | 获取键key对应的值（若键key不存在则返回null） |
| void           | delete(Key key)            | 从表中删去键key（及其对应的值） |
| boolean        | contains(Key key)          | 键key在表中是否有对应的值 |
| boolean        | isEmpty()                  | 表是否为空 |
| int            | size()                     | 表中的键值对数量 |
| Iterable<Key\>  | keys()                     | 表中的所有键的集合 |

为了简洁实用，我们做了一些简单的设计规定：

- 每个键只对应一个只（表中不允许存在重复的键）
- 当插入一个已经存在的键，将覆盖旧的值
- 不允许空键和空值

先花一些时间自己想一下用什么样的数据结构可以实现这样的功能。首先能想到的就是最基础的数据结构——数组或者是链表，假如我们用数组来实现，保存键和值是不是需要两个数组？好像有点麻烦，用链表的话是不是一个结点就可以包含键和值？那就先用链表来实现这些API吧。

<!-- more -->

...
...
...

```
package com.rason.algs4.map;

import edu.princeton.cs.algs4.Queue;

public class SequentialSerachST<Key,Value> {
	private Node first;
	private int n;
	
	public SequentialSerachST() {
	}

	public void put(Key k, Value v){
		if(k == null) throw new NullPointerException("key is null");
		if(v == null) {
			delete(k);
			return;
		}
		
		for (Node node = first; node != null; node = node.next) {
			if(node.k.equals(k)){
				node.v = v;
				return;
			}
		}
		first = new Node(k, v, first);
		n++;
	}
	
	public Value get(Key k){
		if(k == null) throw new NullPointerException("key is null");
		
		for (Node node = first; node != null; node = node.next) {
			if(node.k.equals(k))
				return node.v;
		}
		return null;
	}
	
	public void delete(Key k){
		if(k == null) throw new NullPointerException("key is null");
		first = delete(first, k);
	}
	
	private Node delete(Node node, Key k){
		if(node == null)
			return null;
		if(k.equals(node.k)){
			n--;
			return node.next;
		}
		node.next = delete(node.next, k);
		return node;
	}
	
	public boolean contains(Key k){
		if(k == null) throw new NullPointerException("key is null");
		return get(k) != null;
	}
	
	public boolean isEmpty(){
		return n == 0;
	}
	
	public int size(){
		return n;
	}
	
	public Iterable<Key> keys(){
		Queue<Key> queue = new Queue<Key>();
        for (Node node = first; node != null; node = node.next)
            queue.enqueue(node.k);
        return queue;
	}
	
	private class Node{
		private Key k;
		private Value v;
		private Node next;
		
		public Node(Key k, Value v, Node next) {
			super();
			this.k = k;
			this.v = v;
			this.next = next;
		}
		
	}
}

```
上述符号表的实现使用了一个私有内部类Node来在链表中保存键和值。get()的实现会顺序地搜索链表查找给定的键。put()的实现也会顺序地搜索链表查找给定的键，如果找到则更新相关联的值，否则创建新的结点插入到链表的开头。

> 在含有N对键值的基于（无序）链表的符号表中，未命中的查找和插入操作都需要N次比较。命中的查找在最坏情况下需要N次比较。特别地，向一个空表中插入N个不同的键需要～N*N/2次比较。

很显然，基于链表的实现以及顺序查找是非常低效的，在庞大的数据情况下就不适用了。

## 有序符号表API

在很多情况下，键都是Compareable的对象，因此可以使用a.compareTo(b)来比较a和b两个键。这样可以实现符号表键的有序性，可以更好地实现put()和get()方法，还可以增加一下更使用的操作。比如说键是时间，你可能对最早或最晚的键或者是给定时间段内的所有键感兴趣。

因此，我们也设计一下有序符号表的API如下：

| public class   | ST<Key extends Comparable<Key\>, Value\>   |    |
| :------------: |:----------------------------------------:| :-----:|
|                | ST()                                     | 创建一张符号表 |
| void           | put(Key key, Value value)                | 将键值对存入表中（若值为空则将键key从表中删除） |
| Value          | get(Key key)                             | 获取键key对应的值（若键key不存在则返回null） |
| void           | delete(Key key)                          | 从表中删去键key（及其对应的值） |
| boolean        | contains(Key key)                        | 键key在表中是否有对应的值 |
| boolean        | isEmpty()                                | 表是否为空 |
| int            | size()                                   | 表中的键值对数量 |
| Key            | min()                                    | 最小的键 |
| Key            | max()                                    | 最大的键 |
| Key            | floor(Key key)                           | 小于等于key的最大键 |
| Key            | ceiling(Key key)                         | 大于等于key的最小键 |
| int            | rank(Key key)                            | 小于key的键数量 |
| Key            | select(Key key)                          | 排名为k的键 |
| void           | deleteMin()                              | 删除最小的键 |
| void           | deleteMax()                              | 删除最大的键 |
| int            | size(Key lo, Key hi)                     | [lo..hi]之间键的数量 |
| Iterable<Key\>  | keys(Key lo, Key hi)                     | [lo..hi]之间的所有键，已排序 |
| Iterable<Key\>  | keys()                                   | 表中的所有键的集合,已排序 |

对于有序的符号表，链表就不适合了，因为我们找到指定键的排名等需要用到索引的操作。这时就可以使用两个平衡数组来分别保存Key和Value了，数组的索引可以高效地实现get()和其他操作。

分析一下上面的API设计，发现与无序的API最大的核心区别就是rank()方法，它返回表中小于给定键的键的数量。因此，对于get()方法，只要给定的键存在表中，rank()方法就能够精确地告诉我们在哪里能够找到它（如果找不到，那它肯定就不在表中了）。

对于put()方法，只要给定的键存在于表中，rank()方法就能够精确地告诉我们到哪里去更新它的值，以及当键不在表中时将键存储到表的何处。我们将所有更大的键往后移动一格来腾出位置（从后向前移动）并将给定的键值分别插入到各自数组中合适的位置。

所以，**对于有序符号表实现的关键点就在于如何高效地实现rank()方法，即找到指定键的合适位置**。二分查找算法就能很好地解决这个问题。

**基于有序数组的二分查找**

```
private int rank(Key k){
	int lo = 0, hi = N - 1;
	while(lo <= hi){
		int mid = (hi - lo)/2 + lo;
		int comp = k.compareTo(keys[mid]);
		if(comp > 0){
			lo = mid + 1;
		}else if(comp < 0){
			hi = mid - 1;
		}else {
			return mid;
		}
	}
	return lo;
}
```
该方法达到的效果：

- 如果表中存在该键，返回键的位置，也就是表中小于它的键的数量；
- 如果表中不存在该键，还是返回表中小于它的键的数量。

**基于二分查找的有序符号表实现**

```
package com.rason.algs4.map;

import edu.princeton.cs.algs4.Queue;

public class BinarySearchST<Key extends Comparable<Key>, Value> {
	private static final int INIT_CAPACITY = 2;
	private Key[] keys;
	private Value[] values;
	private int n;

	public BinarySearchST() {
		this(INIT_CAPACITY);
	}

	public BinarySearchST(int capacity) {
		keys = (Key[]) new Comparable[capacity];
		values = (Value[]) new Object[capacity];
	}

	public Value get(Key k) {
		if (k == null)
			throw new NullPointerException("key is null");
		if (isEmpty())
			return null;
		int i = rank(k);
		if (i < n && keys[i].compareTo(k) == 0)
			return values[i];
		else
			return null;
	}

	public void put(Key k, Value v) {
		if (k == null)
			throw new NullPointerException("key is null");
		if (v == null) {
			delete(k);
			return;
		}

		// already in table
		int i = rank(k);
		if (i < n && keys[i].compareTo(k) == 0) {
			values[i] = v;
			return;
		}

		// resize
		if (n == keys.length)
			resize(2 * keys.length);

		for (int j = n; j > i; j--) {
			values[j] = values[j - 1];
			keys[j] = keys[j - 1];
		}
		values[i] = v;
		keys[i] = k;
		n++;
	}

	public void delete(Key k) {
		if (k == null)
			throw new NullPointerException("key is null");
		if (isEmpty())
			return;
		int i = rank(k);

		// key not in table
		if (i == n || k.compareTo(keys[i]) != 0)
			return;

		for (int j = i; j < n - 1; j++) {
			values[j] = values[j + 1];
			keys[j] = keys[j + 1];
		}
		n--;
		keys[n] = null;
		values[n] = null;

		if (n > 0 && n == keys.length / 4)
			resize(keys.length / 2);
	}

	public boolean contains(Key k) {
		if (k == null)
			throw new NullPointerException("key is null");
		return get(k) != null;
	}

	public Key ceiling(Key k) {
		if (k == null)
			throw new NullPointerException("key is null");
		int i = rank(k);
		if (i == n)
			return null;
		else
			return keys[i];
	}

	public Key floor(Key k) {
		if (k == null)
			throw new NullPointerException("key is null");

		int i = rank(k);
		if (i < n && keys[i].equals(k)) {
			return keys[i];
		}
		if (i == 0) {
			return null;
		} else {
			return keys[i - 1];
		}
	}

	public int rank(Key k) {
		if (k == null)
			throw new NullPointerException("key is null");

		int lo = 0, hi = n - 1;
		while (lo <= hi) {
			int mid = (hi - lo) / 2 + lo;
			int comp = k.compareTo(keys[mid]);
			if (comp > 0) {
				lo = mid + 1;
			} else if (comp < 0) {
				hi = mid - 1;
			} else {
				return mid;
			}
		}
		return lo;
	}

	public boolean isEmpty() {
		return n == 0;
	}

	public int size() {
		return n;
	}

	public Key min() {
		if (isEmpty())
			return null;
		return keys[0];
	}

	public Key max() {
		if (isEmpty())
			return null;
		return keys[n - 1];
	}

	public Key select(int k) {
		if (k < 0 || k >= n)
			return null;
		return keys[k];
	}

	public int size(Key lo, Key hi) {
		if (lo == null)
			throw new NullPointerException("first argument to size() is null");
		if (hi == null)
			throw new NullPointerException("second argument to size() is null");

		if (lo.compareTo(hi) > 0)
			return 0;
		if (contains(hi))
			return rank(hi) - rank(lo) + 1;
		else
			return rank(hi) - rank(lo);
	}

	public Iterable<Key> keys() {
		return keys(min(), max());
	}
	  
	public Iterable<Key> keys(Key lo, Key hi) {
		if (lo == null)
			throw new NullPointerException("first argument to keys() is null");
		if (hi == null)
			throw new NullPointerException("second argument to keys() is null");

		Queue<Key> queue = new Queue<Key>();
		for (int i = rank(lo); i < rank(hi); i++) {
			queue.enqueue(keys[i]);
		}
		if (contains(hi))
			queue.enqueue(keys[rank(hi)]);
		return queue;
	}

	private void resize(int capacity) {
		Key[] tempk = (Key[]) new Comparable[capacity];
		Value[] tempv = (Value[]) new Object[capacity];
		for (int i = 0; i < n; i++) {
			tempk[i] = keys[i];
			tempv[i] = values[i];
		}
		values = tempv;
		keys = tempk;
	}
}

```

> 在N个键的有序数组中进行二分查找最多需要（lgN+1）次比较（无论是否成功）。

尽管查找所需的时间是对数级别，但是对于庞大的数据put()方法还是太慢，因为它虽然比较少了，但是访问移动数组的次数多了。而且，在键是随机排序的情况下，构造一个基于有序数组的符号表所需要访问数组的次数是数组长度的平方级别。

> 向大小为N的有序数组中插入一个新的元素在最坏情况下需要访问～2N次数组，因此向一个空符号表中插入N个元素最坏情况下需要访问～N*N/2次数组。

## 性能分析

下图是简单的符号表实现的成本总结：

![简单的符号表实现的成本总结](/image/symboltable)

可以看出，两种方式都不能高效地实现查找和插入操作。因此，**符号表的核心问题就在于我们能否找到能够同时保证查找和插入操作都是对数级别的算法和数据结构**。

那么什么样的数据结构可以同时满足这两个条件？我们试想一下，要支持高效的插入操作，似乎需要一种链式结构。但是链式结构又无法使用二分查找，因为二分查找的高效来自于能够快速通过索引取得任何子数组的中间元素（但得到一条链表的中间元素的唯一方法只能是沿链表遍历）。

为了将二分查找的效率和链表的灵活性结合起来，我们需要更加复杂的数据结构，能够同时拥有两者的就是**二叉查找树**。下一篇文章再研究这个。

## 总结

本篇文章学习了符号表的定义，无序、有序符号表的API设计和实现，两种实现的性能情况。结论是这两种简单的数据结构都不能同时保证高效的查找和插入操作，能够将二分查找的效率和链表的灵活性结合起来的数据结构**二叉查找树**将在下一篇文章中学习。
