---
title: Java——ArrayList源码解析
date: 2018-11-07
tags: Java——源码分析
toc: true
---


以下针对JDK 1.8版本中的**ArrayList**进行分析。
## 概述
&nbsp;&nbsp;&nbsp;&nbsp;`ArrayList`基于`List`接口实现的大小可变的数组。其实现了所有可选的`List`操作，并且元素允许为任意类型，包括`null`元素。除了实现`List`接口，此类还提供了操作内部用于存储列表数组大小的方法（这个类除了没有实现同步外，功能基本与`Vector`一致）。
<!--more-->
&nbsp;&nbsp;&nbsp;&nbsp;每个`ArrayList`实例都有一个容量。容量是用于存储列表中元素的数组的大小。它始终至少与列表大小一样大。随着元素添加到`ArrayList`，其容量会自动增加。除了添加元素具有恒定的摊销时间成本这一事实之外，增长策略并没有详细指出。

&nbsp;&nbsp;&nbsp;&nbsp;我们在添加大容量数据的时候可以使用`ensureCapacity`方法来主动扩容，这可以减少自动扩容的次数。

&nbsp;&nbsp;&nbsp;&nbsp;值得注意的是，这些实现都不是同步的。因此，当多个线程并发访问一个`ArrayList`实例并且至少有一个线程对这个实例进行结构性调整的时候，必须在外部额外实现同步（对于结构性调整，主要指增加或删除一个或多个元素，更精确的说，就是对这个列表的大小进行了调整，对于更改元素的数值并非是结构性调整）。这通常通过在自然封装列表的某个对象上进行同步来完成。如果不存在此类对象，则应使用`Collections.synchronizedList`方法“包装”该列表。这最好在创建时完成，以防止意外地不同步访问列表：`List list = Collections.synchronizedList(new ArrayList(...));`

&nbsp;&nbsp;&nbsp;&nbsp;注意，迭代器的快速失败行为无法得到保证，因为一般来说，不可能对是否出现不同步并发修改做出任何硬性保证。快速失败迭代器会尽最大努力抛出`ConcurrentModificationException`。因此，为提高这类迭代器的正确性而编写一个依赖于此异常的程序是错误的做法：迭代器的快速失败行为应该仅用于检测 bug。

## 源码分析

### <span id="主要字段">主要字段</span>
```Java
/**
 * 默认初始容量
 */
private static final int DEFAULT_CAPACITY = 10;

/**
 * 用于ArrayList空实例的共享空数组实例
 */
private static final Object[] EMPTY_ELEMENTDATA = {};

/**
 * 用于默认大小空实例的共享空数组实例。我们将this（DEFAULTCAPACITY_EMPTY_ELEMENTDATA）
 * 和EMPTY_ELEMENTDATA区别开来，以便在添加第一个元素时知道数组大小要扩容为多少多少。
 */
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * 存储 ArrayList 元素的数组缓冲区。ArrayList 的容量是此数组缓冲区的长度。
 * 当第一个元素添加进空数组时候 elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA 将会被扩容至 DEFAULT_CAPACITY
 */
transient Object[] elementData; // non-private to simplify nested class access

/**
 * ArrayList的数组大小（ArrayList包含的元素个数
 */
private int size;
```
&nbsp;&nbsp;&nbsp;&nbsp;注意此处的`elementData`字段是用的`transient`修饰的以及对于空实例有`DEFAULTCAPACITY_EMPTY_ELEMENTDATA `和`EMPTY_ELEMENTDATA`两个共享空数组实例，下面会对提到的这些注意点进行分析。

### 构造函数
```Java
/**
 * 根据指定的容量初始化空的列表，注意当容量为 0 时，使用的是 EMPTY_ELEMENTDATA
 */
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

/**
 * 初始化容量为 10 的空列表
 */
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

/**
 * Constructs a list containing the elements of the specified
 * collection, in the order they are returned by the collection's
 * iterator.
 */
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;注意到对于无参构造器使用的是`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，而对于带参构造器，当 <u>initialCapacity</u> 为0时，使用的是`EMPTY_ELEMENTDATA`；另外，在无参构造器中的注释——“初始化容量为10的空列表”，我们不禁有以下疑惑：
- `EMPTY_ELEMENTDATA`和`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`都是空的对象数组，为什么在构造器中要对其进行区分
- 无参构造器中，只是把空的对象数组赋值给了`elementData`，为什么注释称声明了长度为10的空数组

对于以上问题，将在**存储和扩容**部分进行讲解。

### 存储和扩容
```Java
/**
 * 增加 ArrayList 实例的容量，确保 ArrayList 实例能存储 minCapacity 个元素
 */
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
        // any size if not default element table
        ? 0
        // larger than default for default empty table. It's already
        // supposed to be at default size.
        : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}
```
&nbsp;&nbsp;&nbsp;&nbsp;当ArrayList实例是个空列表并且 `elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA`时，<u>minExpand</u> 设置为`DEFAULT_CAPACITY`(10)，此时如果 <u>minCapacity</u> 小于 <u>minExpand</u>，那么不马上进行扩容操作，在进行`add`操作时候，会初始化一个容量为 10 的空列表，这样不仅符合无参构造器中的注释，并且保证了 ArrayList 实例能够存储 <u>minCapacity</u> 个元素。

```Java
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```
&nbsp;&nbsp;&nbsp;&nbsp;在`add`操作内部，会调用这个**私有**方法来确保有足够的容量来放置元素。注意，这个函数一开始就对 <u>elementData</u> 进行判断是否为 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA`，如果是的话，证明是无参构造器初始化的实例，在下一步会初始化一个容量为 10 的空列表，符合无参构造器中的注释，其实就是一个延迟初始化的技巧。

```Java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // 扩容操作
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

/**
 * 允许分配的最大数组大小
 *一些 VM 会在数组头部储存头数据，试图尝试创建一个比 Integer.MAX_VALUE - 8 大的数组可能会产生 OOM 异常。
 */
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

/**
 * 增加 ArrayList 实例的容量，确保 ArrayList 实例能存储 minCapacity 个元素
 */
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    //扩容为当前容量的 1.5 倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}

/**
 * 将指定的元素追加到此列表的末尾
 */
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

/**
 * 在列表中将指定元素插入到指定位置，将其后元素都向右移动一个位置
 */
public void add(int index, E element) {
    //检查 index 是否越界
    rangeCheckForAdd(index);

    //确保有足够的容量能够添加元素
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}

/**
 * 将指定集合中的全部元素添加到列表尾端
 */
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}

/**
 * 将指定集合中的全部元素插入到列表指定的位置后面
 */
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index);

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount

    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved);

    System.arraycopy(a, 0, elementData, index, numNew);
    size += numNew;
    return numNew != 0;
}
```
&nbsp;&nbsp;&nbsp;&nbsp;因此对于无参构造器的注释的疑问，到达这里就可以解答了，它确确实实初始化了一个大小为 10 的空列表，只是不是一开始就初始化，而是使用了延迟初始化的方式，在`add`的时候才进行初始化。
&nbsp;&nbsp;&nbsp;&nbsp;对于另一个问题，无参构造器使用`DEFAULTCAPACITY_EMPTY_ELEMENTDAT`，对于`new ArrayList(0);`使用的是`EMPTY_ELEMENTDATA`，前者是不知道需要的容量大小，后者预估元素较少。因此`ArrayList`对此做了区别，通过引用判断来区别用户行为，使用不同的扩容算法（扩容速度：无参：10->15->22...，有参且参数为0 ：0->1->2->3->4->6->9...)。另外，在 JDK 1.7 中，没有通过两个空数组来对用户行为进行区分，因此容量为 0 的话，会创建很多空数组`new Object[0]`，因此上述方式也对这种情况进行了优化。
```Java
//JDK 1.7
public ArrayList(int initialCapacity) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    this.elementData = new Object[initialCapacity];
}


public ArrayList() {
    this(10);
}
```

### 删除
```Java
/**
 * 从列表中删除指定位置的元素，并将其后位置的元素向左移动
 */
public E remove(int index) {
    //检查是否超过数组越界
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    //置null，让 GC 可以工作
    elementData[--size] = null;
    return oldValue;
}

/**
 * 删除列表中首次出现的指定的元素，若列表不存在相应元素，则不做改变
 */
public boolean remove(Object o) {
    //若指定元素为 null，因其为空，没有 equals 方法，因此这两个地方做一个区分
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

/*
 * Private remove method that skips bounds checking and does not
 * return the value removed.
 */
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}

/**
 * 删除列表中所有的元素
 */
public void clear() {
    modCount++;

    // 置 null 以便让 GC 回收
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}

/**
 * 从列表中删除 fromIndex <= pos < toIndex 的元素
 */
protected void removeRange(int fromIndex, int toIndex) {
    modCount++;
    int numMoved = size - toIndex;
    System.arraycopy(elementData, toIndex, elementData, fromIndex,
                     numMoved);

    // clear to let GC do its work
    int newSize = size - (toIndex-fromIndex);
    for (int i = newSize; i < size; i++) {
        elementData[i] = null;
    }
    size = newSize;
}
```

### ArrayList的序列化
在 [主要字段](#主要字段) 部分我们可以看到，<u>elementData</u> 是通过`transient`修饰的（<u>transient</u>具体用法可参看[Java对象序列化](https://www.cnblogs.com/ZhaoxiCheung/p/Java-Serializable.html)一文），通过`transient`声明，因此其无法通过序列化技术保存下来，但仔细阅读源码发现其内部实现了序列化和反序列化函数：
```Java
/**
 * 保存 ArrayList 中实例的状态到序列中
 */
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();

    // Write out size as capacity for behavioural compatibility with clone()
    s.writeInt(size);

    // Write out all elements in the proper order.
    for (int i=0; i<size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount) {
        throw new ConcurrentModificationException();
    }
}

/**
 * 从序列中恢复 ArrayList 中实例的状态
 */
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    elementData = EMPTY_ELEMENTDATA;

    // Read in size, and any hidden stuff
    s.defaultReadObject();

    // Read in capacity
    s.readInt(); // ignored

    if (size > 0) {
        // be like clone(), allocate array based upon size not capacity
        ensureCapacityInternal(size);

        Object[] a = elementData;
        // Read in all elements in the proper order.
        for (int i=0; i<size; i++) {
            a[i] = s.readObject();
        }
    }
}
```
通过一个例子验证其序列化和反序列化过程：
```Java
package test;

import java.io.*;
import java.util.ArrayList;
import java.util.List;

public class ListDemo {
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        List<String>list = new ArrayList<>();
        list.add("hello");
        list.add("world");

        //write Obj to File
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("file"));
        oos.writeObject(list);
        oos.close();

        //read Obj from File
        File file = new File("file");
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file));
        List<String>newList = (List<String>) ois.readObject();
        ois.close();
        System.out.println(newList);
    }
}
/*
[hello, world]
 */
```
可以得出结论：ArrayList支持进行序列化操作，此时不禁会思考既然要将 ArrayList 的字段序列化（即将 elementData 序列化），那为什么又要用 transient 修饰 elementData 呢？实际上，ArrayList 通过动态数组的技术，当数组放满后，自动扩容，但是扩容后的容量往往都是大于或者等于 ArrayList 所存元素的个数。如果直接序列化 elementData 数组，那么就会序列化一大部分没有元素的数组，导致浪费空间，为了保证在序列化的时候不会将这么大部分没有元素的数组进行序列化，因此设置为 transient。
```Java
// Write out all elements in the proper order.
for (int i=0; i<size; i++)
{
    s.writeObject(elementData[i]);
}
```
从源码中，可以观察到循环时是使用`i < size`而不是`i<elementData.length`，说明序列化时，只需实际存储的那些元素，而不是整个数组。

## 问题
- `List<Integer>list = new ArrayList<>(10); out.println(list.get(1));`是否会抛出异常
- ArrayList 支持序列化，为什么 <u>elementData</u> 要设置为`transient`
- 对于空实例数组，为什么要区分`DEFAULTCAPACITY_EMPTY_ELEMENTDATA `和`EMPTY_ELEMENTDATA`