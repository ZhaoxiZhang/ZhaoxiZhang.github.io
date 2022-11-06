---
title: Java——HashMap源码解析
date: 2018-10-21
tags: Java——源码分析
toc: true
---


以下针对JDK 1.8版本中的**HashMap**进行分析。
## 概述
&nbsp;&nbsp;&nbsp;&nbsp;哈希表基于`Map`接口的实现。此实现提供了所有可选的映射操作，并且允许键为`null`，值也为`null`。HashMap 除了不支持同步操作以及支持`null`的键值外，其功能大致等同于 Hashtable。这个类不保证元素的顺序，并且也不保证随着时间的推移，元素的顺序不会改变。
<!--more-->
&nbsp;&nbsp;&nbsp;&nbsp;假设散列函数使得元素在哈希桶中分布均匀，那么这个实现对于 **put** 和 **get** 等操作提供了常数时间的性能。

&nbsp;&nbsp;&nbsp;&nbsp;对于一个 HashMap 的实例，有两个因子影响着其性能：**初始容量**和**负载因子**。容量就是哈希表中哈希桶的个数，初始容量就是哈希表被初次创建时的容量大小。负载因子是在进行自动扩容之前衡量哈希表存储键值对的一个指标。当哈希表中的键值对超过`capacity * loadfactor`时，就会进行 resize 的操作。

&nbsp;&nbsp;&nbsp;&nbsp;作为一般规则，默认负载因子（0.75）在时间和空间成本之间提供了良好的折衷。负载因子越大，空间开销越小，但是查找的开销变大了。

&nbsp;&nbsp;&nbsp;&nbsp;注意，迭代器的快速失败行为不能得到保证，一般来说，存在非同步的并发修改时，不可能作出任何坚决的保证。快速失败迭代器尽最大努力抛出`ConcurrentModificationException`异常。因此，编写依赖于此异常的程序的做法是错误的，正确做法是：迭代器的快速失败行为应该仅用于检测程序错误。

## 源码分析

### 主要字段
```Java
/**
 * 初始容量大小 —— 必须是2的幂次方
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
 * 最大容量
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * 默认负载因子
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
 * 当链表长度超过这个值时转换为红黑树
 */
static final int TREEIFY_THRESHOLD = 8;

/**
 * The bin count threshold for untreeifying a (split) bin during a
 * resize operation. Should be less than TREEIFY_THRESHOLD, and at
 * most 6 to mesh with shrinkage detection under removal.
 */
static final int UNTREEIFY_THRESHOLD = 6;

/**
 * The smallest table capacity for which bins may be treeified.
 * (Otherwise the table is resized if too many nodes in a bin.)
 * Should be at least 4 * TREEIFY_THRESHOLD to avoid conflicts
 * between resizing and treeification thresholds.
 */
static final int MIN_TREEIFY_CAPACITY = 64;


/**
 * table 在第一次使用时进行初始化并在需要的时候重新调整自身大小。对于 table 的大小必须是2的幂次方。
 */
transient Node<K,V>[] table;

/**
 * Holds cached entrySet(). Note that AbstractMap fields are used
 * for keySet() and values().
 */
transient Set<Map.Entry<K,V>> entrySet;

/**
 * 键值对的个数
 */
transient int size;

/**
 * HashMap 进行结构性调整的次数。结构性调整指的是增加或者删除键值对等操作，注意对于更新某个键的值不是结构特性调整。
 */
transient int modCount;

/**
 * 所能容纳的 key-value 对的极限（表的大小 capacity * load factor），达到这个容量时进行扩容操作。
 */
int threshold;

/**
 * 负载因子，默认值为 0.75
 */
final float loadFactor;
```
&nbsp;&nbsp;&nbsp;&nbsp;从上面我们可以得知，HashMap中指定的哈希桶数组<u>table.length</u>必须是2的幂次方，这与常规性的把哈希桶数组设计为素数不一样。指定为2的幂次方主要是在两方面做优化：
- 扩容：扩容的时候，哈希桶扩大为当前的两倍，因此只需要进行左移操作
- 取模：由于哈希桶的个数为2的幂次，因此可以用**&**操作来替代耗时的模运算， `n % table.length -> n & (table.length - 1)`

### 哈希函数
```Java
/**
 * 哈希函数
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;key 的哈希值通过它自身**hashCode**的高十六位与低十六位进行亦或得到。这么做得原因是因为，由于哈希表的大小固定为 2 的幂次方，那么某个 key 的 hashCode 值大于 table.length，其高位就不会参与到 hash 的计算（对于某个 key 其所在的桶的位置的计算为 `hash & (table.length - 1)`）。因此通过`hashCode()`的高16位异或低16位实现的：`(h = key.hashCode()) ^ (h >>> 16)`，主要是从速度、功效、质量来考虑的，保证了高位 Bits 也能参与到 Hash 的计算。

### tableSizeFor函数
```Java
/**
 * 返回大于等于capacity的最小2的整数次幂
 */
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;根据注释可以知道，这个函数返回大于或等于**cap**的最小二的整数次幂的值。比如对于3，返回4；对于10，返回16。详解如下：
假设对于**n**（32位数）其二进制为 01xx...xx，
n >>> 1，进行无符号右移一位， 001xx..xx，位或得 011xx..xx
n >>> 2，进行无符号右移两位， 00011xx..xx，位或得 01111xx..xx
依此类推，无符号右移四位再进行位或将得到8个1，无符号右移八位再进行位或将得到16个1，无符号右移十六位再进行位或将得到32个1。根据这个我们可以知道进行这么多次无符号右移及位或操作，那么可让数**n**的二进制位最高位为1的后面的二进制位全部变成1。此时进行 +1 操作，即可得到最小二的整数次幂的值。（《高效程序的奥秘》第3章——2的幂界方 有对此进行进一步讨论，可自行查看）
回到上面的程序，之所以在开头先进行一次 -1 操作，是为了防止传入的**cap**本身就是二的幂次方，此时得到的就是下一个二的幂次方了，比如传入4，那么在不进行 -1 的情况下，将得到8。

### 构造函数
```Java
/**
 * 传入指定的初始容量和负载因子
 */
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    //返回2的幂次方
    this.threshold = tableSizeFor(initialCapacity);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;对于上面的构造器，我们需要注意的是`this.threshold = tableSizeFor(initialCapacity);`这边的 threshold 为 2的幂次方，而不是`capacity * load factor`，当然此处并非是错误，因为此时 table 并没有真正的被初始化，初始化动作被延迟到了`putVal()`当中，所以 threshold 会被重新计算。
```Java
/**
 * 根据指定的容量以及默认负载因子（0.75）初始化一个空的 HashMap 实例
 *
 * 如果 initCapacity是负数，那么将抛出 IllegalArgumentException
 */
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/**
 * 根据默认的容量和负载因子初始化一个空的 HashMap 实例
 */
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

/**
 * Constructs a new <tt>HashMap</tt> with the same mappings as the
 * specified <tt>Map</tt>.  The <tt>HashMap</tt> is created with
 * default load factor (0.75) and an initial capacity sufficient to
 * hold the mappings in the specified <tt>Map</tt>.
 *
 * @param   m the map whose mappings are to be placed in this map
 * @throws  NullPointerException if the specified map is null
 */
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```
### 查询
```Java
/**
 * 返回指定 key 所对应的 value 值，当不存在指定的 key 时，返回 null。
 *
 * 当返回 null 的时候并不表明哈希表中不存在这种关系的映射，有可能对于指定的 key，其对应的值就是 null。
 * 因此可以通过 containsKey 来区分这两种情况。
 */
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

/**
 * 1.首先通过 key 的哈希值找到其所在的哈希桶
 * 2.对于 key 所在的哈希桶只有一个元素，此时就是 key 对应的节点，
 * 3.对于 key 所在的哈希桶超过一个节点，此时分两种情况：
 *     如果这是一个 TreeNode，表明通过红黑树存储，在红黑树中查找
 *     如果不是一个 TreeNode，表明通过链表存储（链地址法），在链表中查找
 * 4.查找不到相应的 key，返回 null
 */
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}

```

### 存储
```Java
/**
 * 在映射中，将指定的键与指定的值相关联。如果映射关系之前已经有指定的键，那么旧值就会被替换
 */
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

/**
 * * @param onlyIfAbsent if true, don't change existing value
 *
 * 1.判断哈希表 table 是否为空，是的话进行扩容操作
 * 2.根据键 key 计算得到的 哈希桶数组索引，如果 table[i] 为空，那么直接新建节点
 * 3.判断 table[i] 的首个元素是否等于 key，如果是的话就更新旧的 value 值
 * 4.判断 table[i] 是否为 TreeNode，是的话即为红黑树，直接在树中进行插入
 * 5.遍历 table[i]，遍历过程发现 key 已经存在，更新旧的 value 值，否则进行插入操作，插入后发现链表长度大于8，则将链表转换为红黑树
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    //哈希表 table 为空，进行扩容操作
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // tab[i] 为空，直接新建节点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        //tab[i] 首个元素即为 key，更新旧值
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //当前节点为 TreeNode，在红黑树中进行插入
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            //遍历 tab[i]，key 已经存在，更新旧的 value 值，否则进心插入操作，插入后链表长度大于8，将链表转换为红黑树
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //链表长度大于8
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // key 已经存在
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //key 已经存在，更新旧值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //HashMap插入元素表明进行了结构性调整
    ++modCount;
    //实际键值对数量超过 threshold，进行扩容操作
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### 扩容
```Java
/**
 * 初始化或者对哈希表进行扩容操作。如果当前哈希表为空，则根据字段阈值中的初始容量进行分配。
 * 否则，因为我们扩容两倍，那么对于桶中的元素要么在原位置，要么在原位置再移动2次幂的位置。
 */
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        //超过最大容量，不再进行扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //容量没有超过最大值，容量变为原来两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            //阈值变为原来两倍
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        //调用了HashMap的带参构造器，初始容量用threshold替换，
        //在带参构造器中，threshold的值为 tableSizeFor() 的返回值，也就是2的幂次方，而不是 capacity * load factor
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        //初次初始化，容量和阈值使用默认值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        //计算新的阈值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    //以下为扩容过程的重点
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                //将原哈希桶置空，以便GC
                oldTab[j] = null;
                //当前节点不是以链表形式存在，直接计算其应放置的新位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                //当前节点是TreeNode
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    //节点以链表形式存储
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //原索引 + oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;因为哈希表使用2次幂的拓展（指长度拓展为原来的2倍），所以在扩容的时候，元素的位置要么在原位置，要么在原位置再移动2次幂的位置。为什么是这么一个规律呢？我们假设 n 为 table 的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。
![](https://img2018.cnblogs.com/blog/885804/201810/885804-20181019224530421-1332118445.png)
元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：
![](https://img2018.cnblogs.com/blog/885804/201810/885804-20181019224541190-1566201160.png)
因此，我们在扩容的时候，只需要看看原来的hash值新增的那个 bit 是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图：
![](https://img2018.cnblogs.com/blog/885804/201810/885804-20181019224623146-495310474.png)

### 删除
```Java
/**
 * 删除指定的 key 的映射关系
 */
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

/**Java
 *
 * 1.根据 key 的哈希值在哈希桶中查找是否存在这个包含有这个 key 的节点
 *      链表头节点是要查找的节点
 *      节点是TreeNode，在红黑树中查找
 *      在链表中进行查找
 * 2.如果查找到对应的节点，进行删除操作
 *      从红黑树中删除
 *      将链表头节点删除
 *      在链表中删除
 *
 * @param hash key 的 hash 值
 * @param key 指定的 key
 * @param value 当 matchhValue 为真时，则要匹配这个 value
 * @param matchValue 为真并且与 value 相等时进行删除
 * @param movable if false do not move other nodes while removing
 * @return the node, or null if none
 */
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        //链表头即为要删除的节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            //节点为TreeNode，在红黑树中查找是否存在指定的key
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                //在链表中查找是否存在指定的key
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            //从红黑树中删除
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            //链表头删除
            else if (node == p)
                tab[index] = node.next;
            //链表中的元素删除
            else
                p.next = node.next;
            //进行结构特性调整
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}

/**
 * 删除所有的映射关系
 */
public void clear() {
    Node<K,V>[] tab;
    modCount++;
    if ((tab = table) != null && size > 0) {
        size = 0;
        for (int i = 0; i < tab.length; ++i)
            //置 null 以便 GC
            tab[i] = null;
    }
}
```

## 问题
- 对于`new HashMap(18)`，那么哈希桶数组的大小是多少
- HashMap 要求哈希桶数组的长度是2的幂次方，这么设计的目的是为什么
- HashMap 何时对哈希桶数组开辟内存
- 哈希函数是如何设计的，这么设计的意图是什么
- HashMap 扩容的过程，扩容时候对 rehash 进行了什么优化

## 参考资料
[Java 8系列之重新认识HashMap](https://tech.meituan.com/java_hashmap.html)