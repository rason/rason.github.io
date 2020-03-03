title: 散列表
categories: Algorithm
date: 2016-08-17 15:17:40
tags: 散列表
description: 散列表
---

## 前言

如果的键都是小整数，那么我们是不是可以直接用数组实现符号表，键作为数组的索引，数组中的键i索引保存它对应的值。这样，我们就可以快速访问任何键的值了。那么，对于复杂类型的键，我们能够达到这样的效果吗？答案是可以的，我们可以假设复杂类型的键也一样拥有一个值，这个值可以是根据它的属性等方式计算出来的，这样是不是在数组中也有了对应的位置？这就是今天要学习的**散列表**。

## 散列表

上面说到每个复杂类型的键都可以计算出一个值，在Java中就可以是我们的hashCode()方法返回的值了。然而，这个hashCode值可能是很大的，我们不可能创建一个很大的数组啊。因此我们需要一个**散列函数**将这个hashCode转化为数组中的索引，当然转化之后的值不能太大。（注：其实hashCode()方法本身就是一个散列函数，只是返回的散列值范围比较大）

可是，如果这个散列函数对不同的hashCode计算出来的索引是一样的呢？我们并不希望见到这样的情况，因为不同的键我们要放到不同的位置。因此，散列表的第二个关键问题就是我们如何**处理碰撞冲突**问题了。

因此，散列表有两个关键：

- 一是需要有一个高效且能够均匀分布所有键的散列函数
- 二是需要有一个高效处理碰撞冲突的方案

本文的重点就是从**散列函数**和**处理碰撞冲突**两方面展开学习。

散列表是算法在**时间和空间做出权衡**的经典例子。如果没有内存限制，我们可以直接将键作为（可能是一个超大的）数组的索引，那么所有查找操作只需访问内存一次即可完成。另一方面，如果没有时间限制，我们可以使用无序数组并进行顺序查找，这样即只需要很少的内存。而散列表则使用了适度的空间和时间并在这两个极端之间找到了一种平衡。

<!-- more -->

## 散列函数

散列表的第一个关键问题就是散列函数的计算，这个过程会将键转化为数组的索引。如果我们有一个能够保存M个键值对的数组，那么我们需要一个能够将任意键转化为该数组范围内的所有([0,M-1]范围内的整数)的**散列函数**。

我们要找的散列函数应该易于计算并且能够均匀分布所有的键，即对于任意键，0到M-1之间的每个整数都有相等的可能性与之对应（与键无关）。所以，我们该如何去实现这样一个函数？

### 正整数

将正整数散列最常用的方法是**除留余数法**。我们选择大小为**素数**M的数组，对于任意正整数k，计算k除以M的余数。如果M不是素数，可能会导致无法均匀地散列散列值。

### 浮点数

如果键是0到1之间的实数，可以将它乘以M并四舍五入得到一个0到M-1之间的索引值。但是有点缺陷，因为这种情况下键的高位起的作用更大，最低位对散列的结果没有影响。修正这个问题的办法是将键表示为二进制然后再使用除留余数法（Java就是这么做的）。

### 字符串

除留余数法也可以处理较长的键，例如字符串，我们只需将它们当作大整数即可。例如，下面的代码就能够用除留余数法计算String s的散列值：

```
int hash = 0;
for(int i = 0; i < s.length; i++){
	hash = (R*hash + s.charAt(i))%M;
}
```
Java的charAt()函数能够返回一个char值，即一个非负16位整数。上述代码可能一下子不能理解为什么要这么写，但是我们只要知道散列函数的目的：获得分布均匀的索引值，就知道它就是为了达到这样的效果。Java的String的默认实现使用了一个类似的方法。

###　组合键

如果键的类型含有多个整形变量，我们可以和String类型一样将它们混合起来。假设是Date类型，我们可以像下面这样计算散列值：

```
int hash = (((day*R + month) % M) * R + year) % M;
```

### Java的约定

每种类型都需要相应的散列函数，于是Java令所有数据类型都继承了一个能够返回一个32位整数的hashCode()方法。每一种数据类型的hashCode()方法都必须和equals()方法**一致**。也就是说，如果a.equals(b)返回true，那么a.hashCode()的返回值和b.hashCode()的返回值相同。

因此，**为自定义的数据类型定义散列函数，需要同时重写hashCode()和equals()方法。**默认散列函数会返回对象的内存地址，但这只适用于很少的情况。Java中的String,Integer,Double,File和URL等常用数据类型重写了hashCode()方法。

### 将hashCode()的返回值转化为一个数组索引

因为我们需要的是数组的索引而不是一个32位的整数，我们在实现中会将默认的hashCode()方法和除留余数法结合起来产生一个0到M-1的整数，方法如下：

```
private int hash(Key x){
	return (x.hashCode() & 0x7fffffff) % M;
}
```
这段代码将符号位屏蔽（将一个32位整数变为一个31为非负整数），然后用除留余数法计算它除以M的余数。

### 自定义的hashCode()方法

**散列表的用例希望hashCode()方法能够将键平均地散布为所有可能的32位整数**。也就是说，对于任意对象x，你可以调用x.hashCode()并认为有均等机会得到2的32次方中的任意一个32位整数值。Java中的String，Integer，Double，File和URL对象的hashCode()方法都能实现这一点。

在Java中，所有的数据类型都继承了hashCode(),所以我们可以使用对象中每个变量的hashCode()计算散列值。例如：

```
public class Tranaction{
	private final String who;
	private final Date when;
	private final double amount;

	public int hashCode(){
		int hash = 17;
		hash = 31 * hash + who.hashCode();
		hash = 31 * hash + when.hashCode();
		hash = 31 * hash + ((Double)amount).hashCode();
		return hash;
	}
}
```

### 软缓存

如果散列值的计算很耗时，那么我们或许可以将**每个键的散列值缓存起来**，即在每个键中使用一个hash变量来保存它的hashCode()的返回值。

总的来说，要为一个数据类型实现一个优秀的散列方法需要满足三个条件：

- 一致性——等价的键必然产生相等的散列值；
- 高效性——计算简便；
- 均匀性——均匀地散列所有的键。

设计同时满足这三个条件的散列函数是专家们的事。有了各种内置函数，Java程序员在使用散列时只需要调用hashCode()方法即可。但是，在有性能要求时应该谨慎使用散列，因为糟糕的散列函数经常是性能问题的罪魁祸首。

## 基于拉链法的散列表

上面说的都是散列表的第一步工作，先利用散列函数计算出索引位置，但是如果不同的键计算出的索引位置是相同的情况我们应该如何处理？一般来说有两种处理方法:**拉链法**和**线性探测**。

**将大小为M的数组中的每个元素指向一条链表，链表中每个结点都存储了散列值为该元素的索引的键值对，这种方法就叫拉链法。**这种方法的基本思想就是选择足够大的M，使得所有链表都尽可能短以保证高效的查找。如下图所示：

![标准索引用例使用基于拉链法的散列表](/image/separate-chaining.png)

在学习符号表的一开始，我们尝试过用链表的数据结构实现符号表API，现在我们就可以将链表实现的符号表作为M条链保存键值对了。代码如下：

```
package com.rason.algs4.map;

import edu.princeton.cs.algs4.SequentialSearchST;

public class SeparateChainingHashST<Key,Value> {
	private static final int INIT_CAPACITY = 4;
	private int n;// 键值对数量
	private int m;// 散列表的大小
	private SequentialSearchST<Key, Value>[] st;// 链表实现的符号表数组
	
	public SeparateChainingHashST() {
	    this(INIT_CAPACITY);
    } 
	
	public SeparateChainingHashST(int m) {
		this.m = m;
		st = (SequentialSearchST<Key, Value>[]) new Object[m];
		for (int i = 0; i < m; i++) {
			st[i] = new SequentialSearchST<Key,Value>();
		}
	}
	
	public int hash(Key key){
		return (key.hashCode() & 0x7fffffff) % m;
	}
	
	public void put(Key key, Value val){
		if(key == null) throw new NullPointerException("key is null");
		if(val == null){
			delete(key);
			return;
		}
		
		// TODO扩容
		
		int i = hash(key);
		if(!st[i].contains(key))
			n++;
		st[i].put(key, val);
	}
	
	public void delete(Key key){
		if(key == null) throw new NullPointerException("key is null");
		
		int i = hash(key);
		if (st[i].contains(key)) n--;
		st[i].delete(key);
		
		// TODO缩容
	}
	
	public Value get(Key key){
		if(key == null) throw new NullPointerException("key is null");
		
		int i = hash(key);
		return st[i].get(key);
	}
	
}

```
这段简单的符号表实现维护着一条链表的数组，用散列函数来为每个键选择一条链表。当你能够预知所需的符号表的大小时，这段短小精悍的方案能够得到不错的性能。当然更加可靠的方法是动态调整链表数组的大小，这样就可以保证链表较端。动态调整时需要遍历之前所有的键进行重新散列。

> 在一张含有M条链表和N个键的散列表中，未命中查找和插入操作所需的比较次数为～N/M。

### 有序性相关操作

散列最主要的目的在于均匀地将键散布开来，因此在计算散列后键的顺序信息即丢失了。需要实现有序符号表API的方法，散列表不是合适的选择。
**基于拉链法的散列表实现简单，在键的顺序并不重要的应用中，它可能是最快的（也是使用最广泛的）符号表实现。**

## 基于线性探测法的散列表

实现散列表的另一种方式就是用大小为M的数组保存N个键值对，其中**M>N**。我们需要依靠数组中的**空位**解决碰撞冲突。基于这种策略的所有方法被统称为**开放地址**散列表。

开放地址散列表中最简单的方法叫做**线性探测法**：当碰撞发生时（当一个键的散列值已经被另一个不同的键占用），我们直接检查散列表的下一个问题（将索引值加1）。这样的线性探测可能会产生三种结果：

- 命中，该位置的键和被查找的键相同；
- 未命中，键为空（该位置没有键）；
- 继续查找，该位置的键和被查找的键不同。

![标准索引用例使用的基于线性探测的符号表的轨迹](/image/linear-probing.png)

**开放地址类的散列表的核心思想是与其将内存用作链表，不如将它们作为散列表的空元素。**我们可以使用并行数组，一条保存键，一条保存值，并像前面讨论的那样使用散列函数产生访问数据所需的数组索引来实现。代码如下：

```
package com.rason.algs4.map;

public class LinearProbingHashST<Key,Value> {
	private static final int INIT_CAPACITY = 4;
	
	private int n;// 键值对数量
	private int m;// 散列表的大小
	private Key[] keys;// 保存键的数组
	private Value[] vals;// 保存值的数组
	
	public LinearProbingHashST() {
		this(INIT_CAPACITY);
	}

	public LinearProbingHashST(int capacity) {
		this.m = capacity;
		keys = (Key[]) new Object[capacity];
		vals = (Value[]) new Object[capacity];
	}
	
	public int hash(Key key){
		return (key.hashCode() & 0x7fffffff) % m;
	}
	
	public void put(Key key, Value val){
		if(key == null) throw new NullPointerException("key is null");
		if(val == null){
			delete(key);
			return;
		}
		
		// TODO扩容
		
		int i = hash(key);
		for (; keys[i] != null; i = (i + 1) % m) {
			if(key.equals(keys[i])){
				vals[i] = val;
				return;
			}
		}
		
		keys[i] = key;
		vals[i] = val;
		n++;
	}

	public void delete(Key key){
		if(key == null) throw new NullPointerException("key is null");
		if (!contains(key)) return;
		
		// 找到删除的位置
		int i = hash(key);
	    while (!key.equals(keys[i])) {
	    	i = (i + 1) % m;
	    }

	    keys[i] = null;
		vals[i] = null;
		
		// 将删除位置后面的值重新散列
		i = (i + 1) % m;
		for (; keys[i] != null; i = (i + 1) % m) {
			Key keyToRehash = keys[i];
			Value valToRehash = vals[i];
			keys[i] = null;
			vals[i] = null;
			n--;
			put(keyToRehash, valToRehash);
		}
		
		n--;
		// TODO缩容
	}
	
	public Value get(Key key){
		if(key == null) throw new NullPointerException("key is null");
		
		for (int i = hash(key); keys[i] != null; i = (i + 1) % m) {
			if(key.equals(keys[i])){
				return vals[i];
			}
		}
		return null;
	}
	
	public boolean contains(Key key){
		if(key == null) throw new NullPointerException("key is null");
		
		return get(key) != null;
	}
	
}

```

和拉链法一样，开放地址类的散列表的性能也依赖于a=N/M的比例，但意义有所不同。我们将a成为散列表的**使用率**。对于基于拉链法的散列表，a是每条链表的长度，因此一般大于1；对于基于线性探测的散列表，a是表中已被占用的空间的比例，它是不可能大于1的。事实上，我们也不会允许a达到1，因为这样的话未命中的查找会导致无限循环。**为了保证性能，我们会动态调整数组的大小来保证使用率在1/8到1/2之间**。调整大小一样需要遍历所有元素重新散列。

注：拉链法调整数组大小不是必须的，只需要根据查找耗时和（1+N/M）成正比来选择一个合适的M即可。而对于线性探测法，调整数组大小是必需的，防止效率过低和无限循环。

## 总结

本文从**散列函数和处理碰撞冲突**两方面展开学习了散列表。散列函数的目的就是为了获得均匀分布的索引值，当然编写一个这样的散列函数并不容易，我们Java程序员可以信任hashCode()方法。处理碰撞冲突学习了**拉链法和线性探测法**，拉链法使用的是链表数组方式而线性探测法则为整张表使用了两个很大的数组。

散列表的性能有可能支持和数组大小无关的常数级别的查找和插入操作，对于任意的符号表实现，理论上是性能最优的。当然，它也不是万能的，因为：

- 每种类型的键都需要一个优秀的散列函数
- 性能保证来自于散列函数的质量；
- 散列函数的计算可能复杂而且昂贵；
- 难以支持有序性相关的符号表操作。

符号表的内容到现在为止就算是大概地学了一遍，那么我们应该使用符号表的哪种实现？下图总结了各种符号表的性能特点：

![各种符号表实现的渐进性能总结](/image/st-conclusion)

从上表可以看出，对于典型的应用程序，应该在散列表和二叉查找树之间进行选择。相对二叉查找树，散列表的有点在于代码更简单，且查找时间最优。二叉查找树相对于散列表的有点在于抽象结构更简单（不需要设计散列函数），红黑树可以保证最坏情况下的性能。大多数情况下，我们的第一选择都是散列表，在其它因素更重要的情况下才会选择红黑树。

