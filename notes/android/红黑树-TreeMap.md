# 红黑树——TreeMap源码分析

## 导语

最早知道红黑树应用场景并想一窥源码是在研究HashMap源码的时候，习惯性的认为HashMap采用的数据结构是数组+链表的方式，但在Java1.8版本，HashMap做了很多优化，其中一个优化点就是当链表的节点大于8的时候，存储方式变为红黑树。

## 内容概要


## 一. 概念
了解红黑树之前，我们先重温一下二叉查找树

二叉查找树特点

一棵空树，或者是具有下列性质的二叉树：
- (1) 若左子树不空，则左子树上所有节点的值均小于它的根节点的值；
- (2) 若右子树不空，则右子树上所有节点的值均大于它的根节点的值；
- (3) 左、右子树也分别为二叉排序树；
- (4)没有键值相等的节点。

![二叉查找树](https://github.com/Demo-H/Android-Notes/raw/master/assets/tree_base.png)

我们假设有图上这样一颗树，它属于典型的二叉查找树。

这样的树在查找的时候优势很明显，比如我们查找值为9的节点
- 查找到根节点12，由于12比9大，所以查找左孩子6；
- 因为6比9小，所以查找右孩子8；
- 因为8比9小，所以查找右孩子，发现恰好是9，正是查询的值。

![二叉查找树查询](https://github.com/Demo-H/Android-Notes/raw/master/assets/query.gif)

这种查询类同二分法查找的思想，查询所需要的最大次数等于二叉树的高度。

当然我们在插入数据的时候，也是这种思想，一层一层的找到合适的位置然后插入，二叉查找树有一个比较大的缺陷，这个缺陷直接影响到他查询的性能。比如我们来演示一种情况的数据插入操作。
假设只有初始的三个节点的二叉树，我们需要依次插入5个节点数据， 6,5,4,3,2。

![二叉查找树插入](https://github.com/Demo-H/Android-Notes/raw/master/assets/insert.gif)

是不是会觉得很别扭，强迫症的我做这个gif图的时候都抓狂。要是节点足够多的话，就会出现树的一条腿会变的非常长，它虽然满足二叉查找树的特性，但查询的时候已经变成线性查询，等同于链表查询效率了。为了解决这种多次插入而导致的不平衡，我们就需要引入红黑树。

**红黑树的特性**
红黑树就是一种平衡二叉查找树，它具有的特性
- （1）节点是红色或者黑色
- （2）根节点是黑色
- （3）每个叶子节点都是黑色的空节点（null）
- （4）每个红色节点的两个子节点都是黑色的
- （5）从任意节点到其每个叶子的所有路径都包含相同的黑色节点。

如果你的英文好，可以直接看原文
- (1)Each node is either red or black.
- (2)The root is black. This rule is sometimes omitted. Since the root can always be changed from red to black,but not necessarily vice versa, this rule has little effect on analysis.
- (3)All leaves (NIL) are black.
- (4)If a node is red, then both its children are black.
- (5)Every path from a given node to any of its descendant NIL nodes contains the same number of black nodes.
这么多的条条框框，看着就觉得满足这些条件是一件很艰难的事情，但正因为这些规则，才保证了红黑树的平衡。

![红黑树](https://github.com/Demo-H/Android-Notes/raw/master/assets/query.png)

## 二. 查询
红黑树的查询和上文所述的二叉查找树查询是一样的，当我们查某个数的时候，只需要把这个数从根节点比较，如果比根节点大，那就继续比较根节点的右节点，否则就是根节点的左节点，以此循环。



