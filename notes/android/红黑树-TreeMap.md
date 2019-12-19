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

![红黑树](https://github.com/Demo-H/Android-Notes/raw/master/assets/red_and_black.png)

## 二. 查询
红黑树的查询和上文所述的二叉查找树查询是一样的，当我们查某个数的时候，只需要把这个数从根节点比较，如果比根节点大，那就继续比较根节点的右节点，否则就是根节点的左节点，以此循环。

```

    /**
     * 返回指定键所映射的值，如果对于该键而言，此映射不包含任何映射关系，则返回 null。 
     * 更确切地讲，如果此映射包含从键 k 到值 v 的映射关系，根据该映射的排序 key 比较起来等于 k，
     * 那么此方法将返回 v；否则返回 null。（最多只能有一个这样的映射关系。） 
     * 返回 null 值并不一定 表明映射不包含该键的映射关系；也可能此映射将该键显式地映射为 null。
     * 可以使用 containsKey 操作来区分这两种情况。
     * 覆写：类 AbstractMap<K,V> 中的 get
     * @param key	要返回其关联值的键 
     * @return	指定键所映射的值；如果此映射不包含该键的映射关系，则返回 null
     */
    public V get(Object key) {
        Entry<K,V> p = getEntry(key);
        return (p==null ? null : p.value);
    }

   /**
     * 返回与指定键对应的项
     * @param key	要返回其关联值的键 
     * @return 返回与指定键对应的项
     */
    final Entry<K,V> getEntry(Object key) {
    	// 为了提高性能，加载（Offload）基于比较器的版本
        if (comparator != null) // 使用比较器的getEntry版本，返回与指定键对应的项
            return getEntryUsingComparator(key);
        if (key == null)	//  如果指定键为 null 并且此映射使用自然顺序
            throw new NullPointerException();
        @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key; // 使用自然顺序比较器
        Entry<K,V> p = root; // 父节点
        while (p != null) {
            int cmp = k.compareTo(p.key);
            if (cmp < 0)	// 左子节点
                p = p.left;
            else if (cmp > 0) // 右子节点
                p = p.right;
            else
                return p;
        }
        return null;
    }

```

## 三. 插入
```
    /**
     * 将指定值与此映射中的指定键进行关联。如果该映射以前包含此键的映射关系，那么将替换旧值。
     * 覆写：类 AbstractMap<K,V> 中的 put
     * @param key	要与指定值关联的键
     * @param value	要与指定键关联的值 
     * @return	与 key 关联的先前值；如果没有针对 key 的映射关系，则返回 null。
     * 		    （返回 null 还可能表示该映射以前将 null 与 key 关联。） 
     */
    public V put(K key, V value) {
        Entry<K,V> t = root;
        if (t == null) {	// 根节点为空，直接设置为根节点
            compare(key, key); // 类型(可能为空)检查
 
            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;	// 获取比较器
        if (cpr != null) {	// 使用指定的比较器
            do {
                parent = t;
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)	// 左子节点
                    t = t.left;
                else if (cmp > 0)	// 右子节点
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        else {	// 使用自然顺序比较器
            if (key == null)	// 如果指定键为 null 并且此映射使用自然顺序，或者其比较器不允许使用 null 键
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)	// 左子节点
                    t = t.left;
                else if (cmp > 0)	// 右节点
                    t = t.right;
                else
                    return t.setValue(value);  //如果该红黑树之前就有这个Key值节点得话，就直接替换然后return
            } while (t != null);   // 如果t为空的时候，结束循环，parent为Key值需要插入的位置的父节点
        }
        Entry<K,V> e = new Entry<>(key, value, parent);
        // 插入节点e
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
        //节点插入后，需要检查红黑树规则是否被打破，并进行修复
        fixAfterInsertion(e); 
        size++;
        modCount++;
        return null;
    }
   
   /**
     * 插入数据后修复
     * 新插入节点为红色
     * 一、父节点是黑色，直接插入不需要重构
     * 二、父节点是红色 （基准节点不是根节点）
     * @param x	需要插入的节点
     */
    private void fixAfterInsertion(Entry<K,V> x) {
        x.color = RED;	// x节点为红色
 
        while (x != null && x != root && x.parent.color == RED) {	// x节点非空且x不是根节点且x的父节点的颜色为红色
            if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {	// 父节点是祖父节点的左子节点
                Entry<K,V> y = rightOf(parentOf(parentOf(x)));	// y等于父节点的右子节点
                if (colorOf(y) == RED) {	// y为红色
                    setColor(parentOf(x), BLACK);	// 父节点为黑色
                    setColor(y, BLACK);	// y为黑色
                    setColor(parentOf(parentOf(x)), RED); // 祖父节点为红色
                    x = parentOf(parentOf(x));	// x为祖父节点
                } else { // y为黑色
                    if (x == rightOf(parentOf(x))) { // x等于父节点的右子节点
                        x = parentOf(x);	// x等于父节点
                        rotateLeft(x);	// x节点进行左旋操作
                    }
                    setColor(parentOf(x), BLACK);	// 父节点为黑色
                    setColor(parentOf(parentOf(x)), RED);	// 祖父为红色
                    rotateRight(parentOf(parentOf(x)));	// 祖父节点进行右旋操作
                }
            } else {	// 父节点是祖父节点的右子节点
                Entry<K,V> y = leftOf(parentOf(parentOf(x)));	// y等于祖父节点的左子节点
                if (colorOf(y) == RED) {	// y为红色
                    setColor(parentOf(x), BLACK);	// 父节点为黑色
                    setColor(y, BLACK);	// y为黑色
                    setColor(parentOf(parentOf(x)), RED);	// 祖父节点为红色
                    x = parentOf(parentOf(x));	// x为祖父节点
                } else {	// y为黑色
                    if (x == leftOf(parentOf(x))) {	// x为父节点的左子节点
                        x = parentOf(x);	// x为父节点
                        rotateRight(x);	// x节点进行右旋操作
                    }
                    setColor(parentOf(x), BLACK);	// 父节点为黑色
                    setColor(parentOf(parentOf(x)), RED);	// 祖父节点为红色
                    rotateLeft(parentOf(parentOf(x)));	// 祖父节点进行左旋操作
                }
            }
        }
        root.color = BLACK;	// 根节点为黑色
    }

   
```


