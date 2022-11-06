---
title: Java——LinkedHashMap源码解析
date: 2018-12-12
tags: Java——源码分析
toc: true
---


以下针对JDK 1.8版本中的**LinkedHashMap**进行分析。
对于`HashMap`的源码解析，可阅读[Java——HashMap源码解析](https://www.cnblogs.com/ZhaoxiCheung/p/Java-HashMap-Source-Analysis.html)
## 概述
&nbsp;&nbsp;哈希表和链表基于`Map`接口的实现，其具有可预测的迭代顺序。此实现与`HashMap`的不同之处在于它维护了一个包括所有条目（Entry）的双向链表。相比于无序的`HashMap`，`LinkedHashMap`迭代顺序支持按插入条目顺序或者按访问条目顺序，默认迭代顺序为按插入顺序。对于相同 <u>key</u> 的重复插入，其不会改变插入顺序。
<!--more-->
&nbsp;&nbsp;此实现可以让客户端免受由`HashMap`（和`Hashtable`）提供的未指定的，通常是混乱的排序，而对于与`TreeMap`提供的默认根据键排序的功能相比，其性能成本会更小。使用它可以生成一个与原来顺序相同的映射副本，而与原映射的实现无关：
```Java
void foo(Map m) {
    Map copy = new LinkedHashMap(m);
    ...
}
```
如果模块通过输入得到一个映射，复制这个映射，然后返回由此副本确定其顺序的结果，这种情况下这项技术特别有用（客户端通常期望返回的内容与其出现的顺序相同）。

&nbsp;&nbsp;`LinkedHashMap`提供一种特殊的构造方法来创建哈希表，其迭代顺序根据条目的访问顺序排序，从近期访问最少到近期访问最多的顺序（访问顺序）。这种映射的迭代顺序很适合构建 <u>LRU Cache</u>。调用`put`、`putIfPresent`、`get`、`getOrDefault`、`compute`、`computeIfAbsent`、`computerIfPresent`或者`merge`方法都算是对相应条目的访问（假定调用完成后它还存在）。`replace()`方法只有在值被替换的情况下，才算是对条目的访问。`putAll`方法以指定映射的条目集迭代器提供的键-值映射关系的顺序，为指定映射的每个映射关系生成一个条目访问。任何其他方法均不生成条目访问。特别是，collection 视图上的操作不 影响底层映射的迭代顺序。

&nbsp;&nbsp;可以重写`removeEldestEntry(Map.Entry) `方法来实施策略，以便在将新的条目添加到哈希表时，如果超过指定容量，自动移除旧的条目，这在实现 <u>LRU Cahce</u>的时候将非常有用。

&nbsp;&nbsp;这个类提供了所有可选的`Map`的操作，并且允许`null`元素。和`HashMap`一样，假定哈希函数将元素均匀分布到各个桶中，对于基本操作如`add`、`contains`和`remove`，其提供了常数时间的性能。由于增加了维护链表的开支，其性能很可能比`HashMap`稍逊一筹，不过有一点是例外的：`LinkedHashMap`的 collection 视图迭代所需时间与映射的大小（<u>size</u>）成比例，而与容量（<u>capacity</u>）无关；`HashMap`迭代时间很可能开支较大，因为它所需要的时间与其容量（<u>capacity</u>）成比例。

&nbsp;&nbsp;`LinkedHashMap`有两个因子影响着其性能：**初始容量**和**负载因子**。它们的定义与`HashMap`完全相同。要注意，为初始容量选择非常高的值对此类的影响比对`HashMap`要小，因为此类的迭代时间不受容量的影响。

&nbsp;&nbsp;**值得注意的是，这个类对于`Map`接口都不是同步的。**如果多个线程并发的访问一个哈希表，并且至少有一个线程对这个哈希表进行结构性更改，那么必须增添额外的同步操作。这一般通过对自然封装该映射的对象进行同步操作来完成。如果不存在这样的对象，则应该使用`Collections.synchronizedMap `方法来“包装”该哈希表。最好在创建时完成这一操作，以防止对哈希表的意外的非同步访问：`Map m = Collections.synchronizedMap(new LinkedHashMap(...));`

&nbsp;&nbsp;对于结构性更改指任何添加或者删除一个或者多个条目，或者在按访问顺序的哈希表中影响迭代顺序的任何操作。在按插入顺序的哈希表中，仅更改已存在的 key 对应的 value 值不是结构性修改。在按访问顺序的哈希表中，仅利用`get`查询不是结构性修改。）

&nbsp;&nbsp;Collection（由此类的所有 collection 视图方法所返回）的 iterator 方法返回的迭代器都是快速失败的：在迭代器创建之后，如果从结构上对映射进行修改，除非通过迭代器自身的`remove`方法，其他任何时间任何方式的修改，迭代器都将抛出`ConcurrentModificationException`。因此，面对并发的修改，迭代器很快就会完全失败，而不冒将来不确定的时间发生任意不确定行为的风险。

&nbsp;&nbsp;注意，迭代器的快速失败行为无法得到保证，因为一般来说，不可能对是否出现不同步并发修改做出任何硬性保证。快速失败迭代器会尽最大努力抛出 `ConcurrentModificationException`。因此，为提高这类迭代器的正确性而编写一个依赖于此异常的程序是错误的做法：迭代器的快速失败行为应该仅用于检测 bug。

## 源码分析

### 构造函数
```Java
/**
 * 根据指定的初始容量和负载因子，初始化一个空的按照插入顺序排序的 LinkedHashMap 的实例
 */
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

/**
 * 根据指定的容量和默认的负载因子（0.75），初始化一个空的按照插入顺序排序的 LinkedHashMap 的实例
 */
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

/**
 * 根据默认的容量（16）和负载因子（0.75），初始化一个空的按照插入顺序排序的 LinkedHashMap 实例
 */
public LinkedHashMap() {
    super();
    accessOrder = false;
}

/**
 * 初始化一个根据传入的映射关系并且按照插入顺序排序的 LinkedHashMap 的实例
 * 这个 LinkedHashMap 实例的负载因子为0.75，容量不小于指定的映射关系的数量的最小2次幂
 */
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    putMapEntries(m, false);
}

/**
 * 根据指定的容量、负载因子、排序模式来初始化一个空的 LinkedHashMap 的实例
 */
public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```
&nbsp;&nbsp;从上面的构造函数可以看出来：`accessOrder = false`，如果没有特别指定排序模式，那么其将按照插入顺序来作为迭代顺序。

### 三个重要的回调函数
在`HashMap`源码中，预留了三个回调函数，来让`LinkedHashMap`进行后期操作：
```Java
// Callbacks to allow LinkedHashMap post-actions
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
```
在`LinkedHashMap`中，这三个函数实现如下：
```Java
//移除节点的时候会触发回调，将节点从双向链表中删除，在调用 removeNode 函数时候会执行
void afterNodeRemoval(Node<K, V> e) { // unlink
    LinkedHashMap.Entry<K, V> p =
        (LinkedHashMap.Entry<K, V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}

//新节点插入时会触发回调，根据条件判断是否移除最老的条目，在调用 compute computeIfAbsent merge putVal 函数时候会实行
//实现 LruCache 的时候会用到这个函数
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K, V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

//将节点放置链表尾，在调用 putVal 函数时会执行，保证最近访问节点在链表尾部
void afterNodeAccess(Node<K, V> e) { // move node to last
    LinkedHashMap.Entry<K, V> last;
    //accessOrder为 true表示按照访问顺序排序，并且此时的键值对不在链表尾部
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K, V> p =
            (LinkedHashMap.Entry<K, V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
从上面三个回调函数可以看出，其主要是在对条目进行操作的时候触发来维护双向链表。另外值得一提的是`afterNodeInsertion`和`removeEldestEntry`函数，在构建 <u>LruCache</u> 时将非常有用。对于`removeEldestEntry`，其默认返回`false`，因此默认情况下不会删除最旧的元素：
```Java
/**
 * @param    eldest 哈希表中最近插入的条目，或者如果迭代顺序是按照访问顺序排序，则是最近最少访问的条目。
 *                  如果这个方法返回 true，则这是将被删除的条目。如果在 put 或 putAll 调用之前哈希表为空时，触发此调用，
 *                  则这将是刚插入的条目;换句话说，如果哈希表包含单个条目，则最老的条目也是最新的。
 * @return   返回 true 表明将删除最老的条目
 */
protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
    return false;
}
```
如果需要删除最旧条目，则返回true。在将新条目插入后，`put`和`putAll`将调用此方法。它为实现者提供了在每次添加新条目时删除最旧条目的机会。如果用来实现缓存，则此选项非常有用：它允许哈希表通过删除过时条目来减少内存消耗。
示例使用：重写这个函数实现，以下例子将允许在增长到100个条目时，然后在每次添加新条目时删除最旧的条目，保持100个条目的稳定状态。
```Java
private static final int MAX_ENTRIES = 100;
protected boolean removeEldestEntry(Map.Entry eldest) {
   return size() > MAX_ENTRIES;
}
```
此方法通常不通过重写来修改哈希表，而是通过返回值来判断是否对哈希表进行修改。当然，此方法允许直接修改哈希表，但如果它这样做，则必须返回false（表示哈希表不应尝试任何进一步的修改）。如果在此方法中修改哈希表后返回 true，那么对于结果是未指定。

### 存储
&nbsp;&nbsp;`LinkedHashMap`直接使用了`HashMap`的`put`函数，但重写了`newNode`、`afterNodeAccess`和`afterNodeInsertion`方法。
```Java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    //将节点放置链表尾部
    linkNodeLast(p);
    return p;
}

// 将新增节点放置链表尾部
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```

### 删除
&nbsp;&nbsp;同样的，`LinkedHashMap`仍然直接使用了`HashMap`的`remove`函数，只是对`afterNodeRemoval`回调函数进行了重写。对于`afterNodeRemoval`函数上面已经分析过了。

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
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```
&nbsp;&nbsp;与`HashMap`相比，其多了一步对 <u>accessOrder</u> 的判断来维护链表，当指定迭代顺序按照访问顺序排序时，`get`操作表明对指定的条目进行了一次访问，那么此条目应该移到链表尾部。对于`afterNodeAccess`在上面已经分析过了，值得注意的是，在调用`afterNodeAccess`时，会修改 <u>modeCount</u>，所以当你正在`accessOrder = true`的模式下迭代`LinkedHashMap`时，如果同时查询访问数据，会导致 <u>fail-fast</u>，因为迭代的顺序已经变了。

### 其他
&nbsp;&nbsp;对于`LinkedHashMap`其与`HashMap`还有一些不同，由于`LinkedHashMap`维护一个双向链表，因此在判断哈希表中是否存储着某个键值对的时候，不需要在整个数组桶中查找，而只需要对链表遍历即可，这也是`LinkedHashMap`的其中一处优化。
```Java
public boolean containsValue(Object value) {
    for (LinkedHashMap.Entry<K, V> e = head; e != null; e = e.after) {
        V v = e.value;
        if (v == value || (value != null && value.equals(v)))
            return true;
    }
    return false;
}
```

## 实现 LruCache
在 LeetCode 有一道题——[Lru Cache](https://leetcode.com/problems/lru-cache/)：设计和实现一个  LRU (最近最少使用) 缓存机制，那么就可以利用`LinkedHashMap`可选的迭代顺序——按访问顺序的模式来进行实现：
```Java
class LRUCache {
    private int capacity;
    private Map<Integer, Integer> cache;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new java.util.LinkedHashMap<Integer, Integer> (capacity, 0.75f, true) {
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > capacity;
            }
        };
    }

    public int get(int key) {
        if (cache.containsKey(key)) {
            return cache.get(key);
        } else
            return -1;
    }

    public void put(int key, int value) {
        cache.put(key, value);
    }
}

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache obj = new LRUCache(capacity);
 * int param_1 = obj.get(key);
 * obj.put(key,value);
 */
```
当然，如果觉得直接使用`LinkedHashMap`的方式太过取巧，我们仍可以借鉴`LinkedHashMap`的思想来进行实现——使用 <u>HashMap 和 双向链表</u> 的组合来实现：
```Java
class LRUCache {
    class Node{
        Integer key;
        Integer value;
        Node prev;
        Node next;

        public Node(Integer key, Integer value){
            this.key = key;
            this.value = value;
        }
    }

    private Map<Integer, Node>map;
    Node head;
    Node tail;
    int size;

    public LRUCache(int capacity) {
        size = capacity;
        map = new HashMap<>(capacity);
        head = new Node(null, null);
        tail = new Node(null, null);

        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        Node node = map.get(key);
        if (null != node){
            map.remove(node.key);

            node.prev.next = node.next;
            node.next.prev = node.prev;

            appendTail(node);
            map.put(key, node);
        }

        int value = null == node ? -1 : node.value;
        return value;
    }

    public void put(int key, int value) {
        Node node = map.get(key);
        if (null != node){
            map.remove(node.key);

            node.prev.next = node.next;
            node.next.prev = node.prev;

            node.value = value;
        }else if (map.size() == size){
            Node tmp = head.next;
            map.remove(tmp.key);

            head.next = tmp.next;
            tmp.next.prev = head;

            tmp = null;
        }

        if (null == node)   node = new Node(key, value);
        appendTail(node);
        map.put(key, node);
    }

    public void appendTail(Node node){
        tail.prev.next = node;
        node.prev = tail.prev;
        node.next = tail;
        tail.prev = node;
    }
}

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache obj = new LRUCache(capacity);
 * int param_1 = obj.get(key);
 * obj.put(key,value);
 */
```