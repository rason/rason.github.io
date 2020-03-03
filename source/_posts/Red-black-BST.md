title: 红黑树
categories: Algorithm
date: 2016-08-09 14:19:19
tags: Algorithm
description: 红黑树
---

## 前言

在[上一篇](http://rason.me/2016/08/08/2-3-search-tree/)文章中我们学习了2-3查找树，并且留下了一个悬念：直接用2-3查找树实现符号表API相当复杂，但是我们只需要将其简单变换一下，则可在二叉查找树实现的符号表API的基础上做一些简单的改动即可。这中简单变换之后的数据结构就是**红黑二叉查找树**，我们今天要研究一些它的一些性质及如何利用它实现符号表API。

## 红黑二叉树

那么，首先我们要明白的是，如何将一棵2-3查找树转换成红黑二叉树。

### 替换3-结点

> 红黑二叉查找树背后的基本思想是用标准的二叉查找树（完全由2-结点构成）和一些额外的信息（替换3-结点）来表示2-3树。

我们将树中的链接分成两种类型：

- **红链接**将两个2-结点连接起来构成一个3-结点。
- **黑链接**则是2-3树中的普通链接。

如下图所示：

![由一条红色链接相连的两个2-结点表示一个3-结点](/image/redblack-encoding.png)

这种表示法的一个优点是，我们**无需修改**就可以直接使用标准二叉查找树的get()方法。对于任意的2-3树，只要对结点进行转换，我们都可以立即派生出一棵对应的二叉查找树。

<!-- more -->

### 一种等价的定义

红黑树的另一种定义是含有红黑链接并满足下列条件的二叉查找树：

- 红链接均为左链接；
- 没有任何一个结点同时和两条红链接相连；
- 该树是**完美黑色平衡**的，即任意空链接到根结点的路径上的黑链接数量相同。

满足这样定义的红黑树和相应的2-3树是一一对应的。

### 一一对应

- 如果我们将一棵红黑树中的红链接画平，那么所有的空链接到根结点的距离都是相同的。
- 如果我们将由红链接相连的结点合并，得到的就是一棵2-3树。
- 相反，如果将一棵2-3树中的3-结点画作由红色左链接相连的两个2-结点，那么不会存在能够和两条红链接相连的结点，且树必然是完美黑色平衡的。

如下图所示：

![红黑树和2-3树的一一对应关系](/image/redblack-1-1.png)

所以，**红黑树既是二叉查找树，也是2-3树，能够满足二叉查找树中的简洁高效的查找算法和2-3树中的高效平衡插入算法。**

### 颜色表示

每个结点都智慧有一条指向自己的链接（从它的父结点指向它），我们将链接的颜色保存在表示结点的Node数据类型的布尔变量color中。如果指向它的链接是红色，那么该变量为true，黑色则为false。我们约定空链接为黑色，当我们提到一个结点的颜色时，我们指的是指向该结点的链接的颜色。代码如下：

![红黑树的结点表示](/image/redblack-color.png)

```
public static final boolean RED = true;
public static final boolean BLACK = false;

private class Node{
	Key k;
	Value v;
	Node left;
	Node right;
	int n;
	boolean color;
	public Node(Key k, Value v, int n, boolean color) {
		super();
		this.k = k;
		this.v = v;
		this.n = n;
		this.color = color;
	}
	
}

public boolean isRed(Node node){
	if(node == null) return false;
	return node.color == RED;
}

```

### 旋转

在我们实现的某些操作中可能会出现红色右链接或者两条连续的红链接，但在操作完成前这些情况都会被消息地**旋转**并修复。旋转操作会改变红链接的指向。

- 假设有一条红色的右链接需要被转换为左链接，称为**左旋转**；
- 假设有一条红色的左链接需要被转换为右链接，称为**右旋转**。

![左旋转h的右链接和右旋转h的左链接](/image/rotate.png)

```
public Node rotateLeft(Node h){
	if(h == null) return null;
	Node x = h.right;
	h.right = x.left;
	x.left = h;
	x.color = h.color;
	h.color = RED;
	x.n = h.n;
	h.n = size(h.left) + size(h.right) + 1
	return x;
}

public Node rotateRight(Node h){
	if(h == null) return null;
	Node x = h.left;
	h.left = x.right;
	x.right = h;
	x.color = h.color;
	h.color = RED;
	x.n = h.n;
	h.n = size(h.left) + size(h.right) + 1
	return x;
}
```

在插入新的键时我们可以使用旋转操作帮助我们保证2-3树和红黑树之间的一一对应关系，因为旋转操作可以保持红黑树的两个重要性质：**有序性和完美平衡性**。

那么，如何使用旋转来保持红黑树的另外两个重要性质（不存在两条连续的红链接和不存在红色的右链接）？这里会分几种情况：

### 向2-结点插入新键

![向2-结点插入新键](/image/insert2-node.png)

### 向3-结点插入新键

![向3-结点插入新键](/image/insert3-node.png)

### 颜色变换

上图中有涉及将一个结点的两个红色子结点转换成黑色，我们也将其用一个方法flipColor()来实现。除了将子结点的颜色由红变黑之外，我们同时还要将父结点的颜色由黑变红。这项操作最重要的性质在于和旋转操作一样是局部变换，不会影响**整棵树的黑色平衡性**。

![颜色变换](/image/color-flip.png)

为什么父结点要由黑变红？其实就是和2-3树一样，不断地分解提升到父结点中，所以就是红色的。

### 根结点总是黑色

在上面这种情况，颜色变换会使根结点变为红色。试想一下，如果根结点是红色的会是什么情况？因为红色的结点是一个3-结点的一部分，证明还有结点指向于它，那它就不是根结点了。因此我们在**每次插入后都会将根结点设为黑色**。注意，**每当根节点由红色变黑时树的黑链接高度就会加1**。因为需要颜色变换时根节点才会变成红色，插入后将根节点变成黑色，这就是由红变黑。这时其实就是一个分解根节点的情况，高度会加1。

### 向树底部的3-结点插入新键

![向树底部的3-结点插入新键](/image/insert3-node-bottom.png)

只要谨慎地使用左旋转、右旋转和颜色转换三种简单的操作，我们就能够保证插入操作后红黑树和2-3树的一一对应关系。在沿着插入点到根节点的路径向上移动时在所经过的每个结点中顺序完成一下操作，我们就能完成插入操作：

- 如果右子节点是红色的而左子节点是黑色的，进行左旋转；
- 如果左子节点是红色的且它的左子节点也是红色的，进行右旋转；
- 如果左右子节点均为红色，进行颜色变黑。

## 实现

因为保持树的平衡性所需的操作是由下向上在每个所经过的结点中进行的，将它们植入我们已有的实现中十分简单：只需要在递归调用之后完成这些操作即可。

```
public void put(Key k, Value v){
	if(k == null) throw new NullPointerException("key is null");
	if(v == null) {
		delete(k);
		return;
	}
	
	root = put(root, k, v);
	root.color = BLACK;
}

private Node put(Node node, Key k,Value v){
	if(node == null){
		return new Node(k, v, 1, RED);
	}
	int comp = k.compareTo(node.k);
	if(comp > 0){
		node.right = put(node.right, k, v);
	}else if(comp < 0){
		node.left = put(node.left, k, v);
	}else{
		node.v = v;
	}
	
	// 修正红黑树
	if(isRed(node.right) && !isRed(node.left)) node = rotateLeft(node);
	if(isRed(node.left) && isRed(node.left.left)) node = rotateRight(node);
	if(isRed(node.left) && isRed(node.right)) flipColors(node);
	
	node.n = size(node.left) + size(node.right) + 1;
	return node;
}
```

至于红黑树的删除操作，是比插入操作复杂得多的，由于篇幅关系此处就不做深入研究了。

## 红黑树的性质

> 所有基于红黑树的符号表实现都能保证操作的运行时间为对数级别（范围查找除外，它所需的额外时间和返回的键的数量成正比）。

> 一棵大小为N的红黑树的高度不会超过2lgN，根结点到任意结点的平均路径长度为～1.00lgN。

红黑树最吸引人的一点是它的实现中最复杂的代码仅限于插入和删除方法。二叉查找树中的查找最大和最小键、select()、rank()、floor()、ceiling()和范围查找方法**不作任何改变**即可继续使用，因为红黑树也是二叉查找树而这些操作也不会涉及结点的颜色。

![各种符号表实现性能总结](/image/st-performance)

## 总结

到目前为止，我们把红黑树的基本特性和操作都熟悉了一遍。将2-3树的3-结点转换一下表示方式即可变成红黑树，红黑树的基本操作就是左右旋转和颜色变换，这些操作都是局部操作，不会影响红黑树的有序性和完美平衡性。并且简单地学习了一下红黑树实现的符号表及其性能总结比较。
