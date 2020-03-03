title: 二叉树中序遍历
categories: LeetCode
date: 2016-06-13 10:00:35
tags: Binary Tree
description: 二叉树中序遍历
---

## 前言

偶然的知道了LeetCode，正好自己也在学习算法，于是上去随机选了一道题来做——二叉树的终须遍历。咋一看题目，由于太久没复习算法和数据结构，连二叉树的概念都模糊了，更别说中序遍历了。于是，先谷歌了一下概念，然后设法遍历……

## 二叉树

> 在计算机科学中，二叉树是每个节点最多有两个子树的树结构。通常子树被称作“左子树”（left subtree）和“右子树”（right subtree）。

### 数据结构

```
 public class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    TreeNode(int x) { val = x; }
 }
```

### 遍历

- 前序遍历：根节点，左子树，右子树
- 中序遍历：左子树，根节点，右子树
- 后序遍历：左子树，右子树，根节点

**注意：**这里要理解的是树，而不是孩子。举个例子：

![二叉树](/image/binaryTree)

<!-- more -->

上图中，根节点为1，{2,4,5，8}为左子树，{3,6,7,9,10}为右子树；再深入1的左子树，它也是一棵二叉树，根节点为2，左子树为{4}，右子树为{5、8}；依此类推。

因此，上图中的各种遍历结果如下：

- 前序遍历：1 2 4 5 8 3 6 7 9 10
- 中序遍历：4 2 8 5 1 6 3 9 7 10
- 后序遍历：4 8 5 2 6 9 10 7 3 1

## 中序遍历

好了，简单地复习完上述概念，可以开始着手编写代码实现中序遍历了。LeetCode给出的代码模板如下：

```
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
public class Solution {
    public List<Integer> inorderTraversal(TreeNode root) {
       
    }
}
```

要求很简单，就是给出一个根节点，返回中序遍历的节点值。先不要往下看答案，先自己想一下怎么实现。

...
...
...

由于本人智商比较拙计，看着上面那张图想了大半天没想出来怎么做。其中想过用栈的方式，但是想不出一个栈怎么整；其中又想了多个栈能不能实现，稀里糊涂地折腾了半天楞是没想出来。

...
...
...

后来，我干脆放弃上面那张图了，我为什么要按照上面那张那么复杂的图来想，我不会按照只有三个节点的二叉树的图来实现吗？这不就简单多了吗？

![简单的二叉树](/image/binaryTree2)

嘿嘿，我真的是太机智了，遍历上面只有三个节点的二叉树难道我不会？然后就吭哧吭哧的开始编码了。

```
public class Solution {
	public static List<Integer> inorderTraversal(TreeNode root) {
		
	}
	
	public void printNode(TreeNode root){
		System.out.println(root.left.val);
		System.out.println(root.val);
		System.out.println(root.right.val);
	}
}	
```

更简单地，我干脆就先直接把它们打印出来，先不返回遍历之后的数组，打印成功了就证明我遍历成功啦。嘿嘿，有了上面的基础，然后我就想，左子树也是一棵树呢，那是不是也可以这样打印？同理，右子树应该也是一样的啊。于是，我就想到了递归的方式：

```
public class Solution {
	public static List<Integer> inorderTraversal(TreeNode root) {
		
	}
	
	public void printNode(TreeNode root){
		if(root.left != null)
			printNode(root.left,list);
		System.out.println(root.val);
		if(root.right != null)
			printNode(root.right,list);
	}
}	
```

编写测试案例：

```
public static void main(String[] args){
	TreeNode one = new TreeNode(1);
	TreeNode two = new TreeNode(2);
	TreeNode three = new TreeNode(3);
	TreeNode four = new TreeNode(4);
	TreeNode five = new TreeNode(5);
	TreeNode six = new TreeNode(6);
	TreeNode seven = new TreeNode(7);
	TreeNode eight = new TreeNode(8);
	TreeNode nine = new TreeNode(9);
	TreeNode ten = new TreeNode(10);
	
	one.left = two;
	one.right = three;
	
	two.left = four;
	two.right = five;
	
	three.left = six;
	three.right = seven;
	
	five.left = eight;
	
	seven.left = nine;
	seven.right = ten;
	
	printNode(one);
}
```

发现输入结果和上面描述的中序排序结果一样，成功了！

既然已经成功地打印了中序遍历，那么只需要将程序简单地改造一下就可以返回中序遍历的数组了。如下：

```
public class Solution {
    public List<Integer> inorderTraversal(TreeNode root) {
        List<Integer> result = new ArrayList<Integer>();
		printNode(root, result);
		return result;
    }
    
	public void printNode(TreeNode root,List<Integer> list){
	    if (root == null)
			return;
		if(root.left != null)
			printNode(root.left,list);
		list.add(root.val);
		if(root.right != null)
			printNode(root.right,list);
	}
	
}
```

## 总结

这道题在LeetCode上的困难程度属于中等，由于太久时间没有温习数据结构和算法，所以思考了很久才想出解决方案。值得高兴的是，通过自己的思考找到了解决的方法，并且通过将问题化繁为简得出了解决问题的思路，这将成为一套自己的方法论。**遇到问题不要胡思乱想，先将问题简单化，或者先将问题特殊化，找出解决方案，然后在此基础上进行解决方案的一般化。**

另外，在提交自己的答案之后，看了一些别人的答案。还真有人用栈的方式来实现，十分巧妙，也证明了自己当时的思考思路并没有完全错，而是没有将问题简单化思考，从而导致了半天没思考出来。这里贴一下别人的巧妙解法：

```
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> list = new ArrayList<Integer>();

    Stack<TreeNode> stack = new Stack<TreeNode>();
    TreeNode cur = root;

    while(cur!=null || !stack.empty()){
        while(cur!=null){
            stack.add(cur);
            cur = cur.left;
        }
        cur = stack.pop();
        list.add(cur.val);
        cur = cur.right;
    }

    return list;
}
```

这个解法很巧妙地控制了入栈和出栈的时机，实现遍历。当初自己就是一条筋想把所有元素全部压入栈再遍历出来，怎么就没想到又压又出。
