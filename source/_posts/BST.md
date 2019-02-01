title: 二叉查找树
categories: Algorithm
date: 2016-07-27 09:40:09
tags: 二叉查找树
description: 二叉查找树
---

## 前言

上篇文章[符号表](http://rason.me/2016/07/21/SymbolTable/)中谈到了符号表的两种基础实现方式——链表和数组，但都无法同时满足高效的插入和查找操作。**二叉查找树**结合了二分查找的效率和链表的灵活性，今天就学习一下如何实现基于二叉查找树的符号表。

## 二叉查找树

### 什么是二叉查找树？

> 一棵二叉查找树（BST）是一棵二叉树，其中每个结点都含有一个Comparable的键（以及相关联的值）且每个结点的键都大于其左子树中的任意结点的键而小于右子树的任意结点的键。

所以，当我们**中序遍历**一棵二叉查找树的键时结果应该是**有序**的。即如果我们将一棵二叉查找树的所有键投影到一条直线上，那么我们一定可以得到一条有序的键列。如下图所示：

![二叉查找树](https://raw.githubusercontent.com/rason/rason.github.io/master/image/BST)

### 数据表示

和链表一样，我们用一个私有类来表示二叉查找树上的一个结点。每个结点都含有一个键、一个值、一条左链接、一条右链接和一个结点计数器。左链接指向一棵由小于该结点的所有键组成的二叉查找树，右链接指向一棵由大于该结点的所有键组成的二叉查找树。变量n给出了以该结点为根的子树的结点总数。数据结构如下：

```
private class Node{
	Key k;
	Value v;
	Node left;
	Node right;
	int n;
	public Node(Key k, Value v, int n) {
		super();
		this.k = k;
		this.v = v;
		this.n = n;
	}
	
}
```

<!-- more -->

## 基于二叉查找树的符号表实现

### 查找

根据二叉查找树的定义，我们很容易想到实现查找的方法和二分查找算法应该是差不多的。先与根结点对比，相等则直接返回，小于则递归左子数，大于则递归右子树，没有命中则返回空。代码如下：

```
public Value get(Key k){
	return get(root, k);
}

private Value get(Node node, Key k){
	if(node == null) return null;
	int comp = k.compareTo(node.k);
	if(comp == 0){
		return node.v;
	}else if(comp > 0){
		return get(node.right, k);
	}else{
		return get(node.left, k);
	}
}
```

### 插入

如何选择合适的位置插入？细想应该应该和查找的方式是差不多的，因为它实际上就相当于查找一个不存在于树的结点并结束于一条空链接，我们只需要做的就是将链接指向一个含有被查找的键的新结点即可。因此，过程也是先与根结点比较，相等则替换根结点的值，如果小于则递归左子树插入，大于则递归右子树插入，如果结点为空则代表我们需要在此位置插入该结点。代码如下：

```
	
public void put(Key k, Value v){
	if(k == null) throw new NullPointerException("key is null");
	if(v == null) {
		delete(k);
		return;
	}
	
	root = put(root, k, v);
}

private Node put(Node node, Key k,Value v){
	if(node == null){
		return new Node(k, v, 1);
	}
	int comp = k.compareTo(node.k);
	if(comp > 0){
		node.right = put(node.right, k, v);
	}else if(comp < 0){
		node.left = put(node.left, k, v);
	}else{
		node.v = v;
	}
	node.n = size(node.left) + size(node.right) + 1;
	return node;
}
```
**注意：**添加了新结点需要沿搜索路径向上增加结点计算器的值。

### 递归思想

在查找和插入方法中我们都使用了递归的方式，递归的思想可能会有点复杂。要写出递归的代码，首先要化繁为简，不要将一棵树想得太复杂，否则思维会越陷越深；我们可以先以三个结点的树为例来思考，这样就不难实现单个非递归的代码了，然后再左右子树扩展递归即可。

关于二叉查找树的查找和插入的递归代码，我们可以将递归前的代码想象成**沿着树向下走**：它会将给定的键和每个结点的键相比较并根据结果向左或向右移动到下一个结点。然后可以将递归调用后的代码想象成**沿着树向上爬**。对于get()方法，这对应着一系列的返回指令（return），但是对于put()方法，这意味着重置搜索路径上每个父结点指向子结点的链接，并增加路径上每个结点中的计算器的值。

### 分析

使用二叉查找树的算法的运行时间取决于树的形状，而树的形状又取决于键被插入的先后顺序。最好的情况下，一棵含有N个结点的数是完全平衡的，每条空链接和根结点的距离都为～lgN。在最坏情况下，搜索路径上可能有N个结点。但在一般情况下树的形状和最好情况更接近。如下图所示：

![最好和一般情况](https://raw.githubusercontent.com/rason/rason.github.io/master/image/BST2)
![最坏情况](https://raw.githubusercontent.com/rason/rason.github.io/master/image/BST3)

在大多数情况下我们可以假设键的分布是随机的，或者说它们的插入顺序是随机的。这样的模型，二叉查找树和快速排序几乎就是"双胞胎"。树的根结点就是快速排序中的第一个切分元素，这和快速排序中对子数组的递归排序完全对应。这使我们能够分析得到二叉查找树的一些性质。

> 在由N个随机键构造的二叉查找树中，查找命中平均所需的比较次数为～2lnN(约1.39lgN)。
> 在由N个随机键构造的二叉查找树中插入操作和查找未命中平均所需的比较次数为～2lnN(约1.39lgN)。

说明在二叉查找树中查找随机键的成本比二分查找高约39%，但这些成本是值得的，因为插入一个新键的成本是对数级别的——这是基于二分查找的有序数组所不具备的灵活性。

## 有序性相关方法和删除操作

二叉查找树得以广泛应用的一个重要原因就是它能够**保持键的有序性**，因此它可以作为实现有序符号表API中的众多方法的基础。

### 最大键和最小键

很显然，最左端的结点键最小，最右端的结点键最大。代码如下：

```
public Key min(){
	if(root == null) return null;
	return min(root).k;
}

private Node min(Node node){
	if(node.left == null)
		return node;
	return min(node.left);
}
```

### 向上取整和向下取整

向下取整时：

- 如果给定的键key小于二叉查找树的根结点的键，那么小于等于key的最大键（floor）一定在根结点的左子树中；
- 如果给定的键key大于二叉查找树的根结点，那么只有当根结点右子树中存在小于等于key的结点时，小于等于key的最大键才会出现在右子树中，否则根结点就是小于等于key的最大键。

代码如下：

```
public Key floor(Key k) {
	if(k == null) throw new NullPointerException("key is null");
	Node floor = floor(root, k);
	if(floor == null) return null;
	return floor.k;
}

private Node floor(Node node, Key k){
	if(node == null) return null;
	int comp = k.compareTo(node.k);
	if(comp == 0) return node;
	if(comp < 0) return floor(node.left, k);
	Node t = floor(node.right, k);
	if(t != null) return t;
	else return node;
}
```

向上取整则相反，左变右，小于变大于。

### 选择操作

在选择操作中，结点中的n就发挥重大作用了。假设一棵有15个结点的平衡二叉查找树，则根结点的n为15，根结点的左结点n为7，根结点的右结点n也为7。当我们选择排序为9的结点时，判断根结点的左结点的n为7，比9小，所以还要加上两个结点，根结点一个和右子树的一个。代码如下：

```
public Key select(int k){
	Node node = select(root, k);
	if(node == null) return null;
	return node.k;
}

private Node select(Node node, int k){
	if(node == null) return null;
	int t = size(node.left);
	if(t > k) return select(node.left, k);
	else if(t < k) return select(node.right, k-t-1);
	else return node;
}
```

### 排名

rank()是select()的逆方法，它会返回给定键的排名。它的实现和select()类似：如果给定的键和根结点的键相等，我们返回左子树中的结点总数t;如果给定的键小于根结点，我们会反hi该键在左子树中的排名（递归计算）；如果给定的键大于根结点，我们会返回t+1（根结点）加上它在右子树中的排名（递归计算）。

```
public int rank(Key k){
	if(k == null) throw new NullPointerException("key is null");
	return rank(root, k);
}

private int rank(Node node, Key k){
	if(node == null) return 0;
	int comp = k.compareTo(node.k);
	if(comp < 0) return rank(node.left, k);
	else if(comp > 0) return  1 + size(node.left) + rank(node.right, k);
	else return size(node.left);
}
```

### 删除最大键最小键

前面说过，最小键就是最左边的结点，因此一直递归到结点的左结点为空，那么这个结点就是最小结点，将其删除。需要注意的是需要将指向该结点的链接指向该结点的右子树。

```
public void deleteMin(){
	root = deleteMin(root);
}

private Node deleteMin(Node node){
	if(node == null) return null;
	if(node.left == null){
		return node.right;
	}
	node.left = deleteMin(node.left);
	node.n = size(node.left) + size(node.right) + 1;
	return node;
}
```

### 删除操作

删除操作比较复杂，需要将删除元素的右子树的最小键结点移动到删除元素的位置，并重新链接左右子树。试想一下，删除元素的右子树的最小键是不是比删除元素左子树的所有结点的键都大，因此将其替换删除元素的位置保持了有序。

```
public void delete(Key k){
	root = delete(root, k);
}

private Node delete(Node node, Key k){
	if(node == null) return null;
	int comp = k.compareTo(node.k);
	if(comp > 0){
		node.right = delete(node.right, k);
	}else if(comp < 0){
		node.left = delete(node.left, k);
	}else{
		if(node.right == null) return node.left;
		if(node.left == null) return node.right;
		Node t = node;
		node = min(t.right);
		node.right = deleteMin(t.right);
		node.left = t.left;
	}
	node.n = size(node.left) + size(node.right) + 1;
	return node;
}
```

### 范围查找

范围查找其实就是二叉查找树的**中序遍历**,只不过加上了范围的限制，因此将中序遍历的方法稍微改造一下即可。

```
public Iterable<Key> keys(){
	return keys(min(), max());
}

public Iterable<Key> keys(Key lo, Key hi){
	Queue<Key> queue = new Queue<Key>();
	keys(root, queue, lo, hi);
	return queue;
}

public void keys(Node node, Queue<Key> queue, Key lo, Key hi){
	if(node == null) return;
	int complo = lo.compareTo(node.k);
	int comphi = hi.compareTo(node.k);
	if(complo < 0) keys(node.left, queue, lo, hi);
	if(complo <= 0 && comphi >= 0) queue.enqueue(node.k);
	if(comphi > 0) keys(node.right, queue, lo, hi);
}
```

### 性能分析

二叉查找树中和有序性相关的操作效率如何？要研究这个问题，我们首先要知道**树的高度**(即树中任意结点的最大深度)。给定一棵树，树的高度决定了所有操作在最坏情况下的性能（范围查找除外，因为它的额外成本和返回的键的数量成正比）。

> 在一棵二叉查找树中，所有操作在最坏情况下所需的时间都和树的高度成正比。

二叉查找树在键随机分布时各种操作性能都还可以，只是在最坏的情况下（比如所有键按顺序插入或逆序插入）性能就较差，这也是我们需要寻找更好的算法和数据结构的原因。

## 总结

本文学习了二叉查找树数据结构，并基于二叉查找树实现了符号表相关操作。二叉查找树结合了二分查找的效率和链表的灵活性，是一种比较高效的实现方式。符号表的三种实现性能对比如下图：

![简单的符号表实现的成本总结](https://raw.githubusercontent.com/rason/rason.github.io/master/image/BST4)
