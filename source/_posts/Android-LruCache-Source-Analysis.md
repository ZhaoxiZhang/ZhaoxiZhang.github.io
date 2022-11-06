---
title: Android——LruCache源码解析
date: 2019-01-01
tags: Android——源码分析
toc: true
---


以下针对 Android API 26 版本的源码进行分析。
在了解`LruCache`之前，最好对`LinkedHashMap`有初步的了解，`LruCache`的实现主要借助`LinkedHashMap`。`LinkedHashMap`的源码解析，可阅读[Java——LinkedHashMap源码解析](https://www.cnblogs.com/ZhaoxiCheung/p/Java-LinkedHashMap-Source-Analysis.html)
<!--more-->
## <span id="概述">概述</span>
&nbsp;&nbsp;`LruCahce`其 Lru 是 Least Recently Used 的缩写，即最近最少使用，是包含对有限数量值的强引用的缓存。每当一个值被访问，它将被移到队尾。当缓存达到指定的数量时，位于队头的值将被移除，并且可能被 GC 回收。如果缓存的值包含需要显式释放的资源，那么需要重写`entryRemoved`方法。如果 key 对应的缓存未命中，通过重写`create`方法创建对应的 value。这可以简化代码调用：即使存在缓存未命中，也允许假设始终返回一个值。
&nbsp;&nbsp;默认情况下，缓存大小以条目数量度量。在不同缓存对象下，通过重写`sizeOf`方法测量 key-value 缓存的大小。例如如下的例子，这个缓存限制了 4MiB 大小的位图：
```Java
int cacheSize = 4 * 1024 * 1024; // 4MiB
LruCache<String, Bitmap> bitmapCache = new LruCache<String, Bitmap>(cacheSize) {
    protected int sizeOf(String key, Bitmap value) {
        return value.getByteCount();
    }
}
```
&nbsp;&nbsp;这个类是线程安全的，通过在缓存上执行同步操作来以原子方式执行多个缓存操作：
```Java
synchronized (cache) {
    if (cache.get(key) == null) {
        cache.put(key, value);
    }
}
```
&nbsp;&nbsp;这个类不允许空值作为 key 或者 value，对于`get`、`put`、`remove`方法返回`null`值是明确的行为：缓存中不存在这个键。

## 源码分析

### 主要字段
```Java
    //LruCache 主要借助 LinkedHashMap 按元素访问顺序的迭代顺序（此时 accessOrder = true）来实现
    private final LinkedHashMap<K, V> map;

    /** 不同 key-value 条目下缓存的大小，不一定是 key-value 条目的数量 */
    private int size;
    //缓存大小的最大值
    private int maxSize;

    //存储的 key-value 条目的个数
    private int putCount;
    //创建 key 对应的 value 的次数
    private int createCount;
    //缓存移除的次数
    private int evictionCount;
    //缓存命中的次数
    private int hitCount;
    //缓存未命中的次数
    private int missCount;
```

### 构造函数
```Java
/**
 * maxSize 对于缓存没有重写 sizeOf 方法的时候，这个数值指定了缓存中可以容纳的最大条目的数量；
 * 对于其他缓存，这是缓存中条目大小的最大总和。
 */
public LruCache(int maxSize) {
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }
    this.maxSize = maxSize;

    //指定了哈希表初始容量为0，负载因子为0.75，迭代顺序为按照条目访问顺序
    //因此在有对条目进行访问的操作的时候，条目都会被放置到队尾，具体细节详看 LinkedHashMap 的解析
    this.map = new LinkedHashMap<K, V>(0, 0.75f, true);
}
```

### Size操作
`LruCache`在默认情况下，<u>size</u> 指的是 key-value 条目的个数，当重写`sizeOf`函数时，可以自定义 key-value 条目的单位大小，如[概述](#概述)中位图的例子，其通过重写`sizeOf`函数，返回的大小值并非是 1，而是不同`Bitmap`对象的字节大小。
```Java
/**
 * 以用户定义的单位返回 key-value 条目的大小
 * 默认实现返回1，因此 size 是条目数，max size是最大条目数
 * 条目的大小在缓存中时不得更改
 */
protected int sizeOf(K key, V value) {
    return 1;
}

private int safeSizeOf(K key, V value) {
    int result = sizeOf(key, value);
    if (result < 0) {
        throw new IllegalStateException("Negative size: " + key + "=" + value);
    }
    return result;
}

/**
 * 删除最旧的条目，直到剩余条目总数小于等于指定的大小。
 */
public void trimToSize(int maxSize) {
    while (true) {
        K key;
        V value;
        synchronized (this) {
            if (size < 0 || (map.isEmpty() && size != 0)) {
                throw new IllegalStateException(getClass().getName()
                        + ".sizeOf() is reporting inconsistent results!");
            }

            //哈希表中条目的大小小于指定的大小即终止
            if (size <= maxSize) {
                break;
            }

            //获取哈希表中最旧的条目
            Map.Entry<K, V> toEvict = map.eldest();

            //哈希表为空，终止
            if (toEvict == null) {
                break;
            }

            key = toEvict.getKey();
            value = toEvict.getValue();
            map.remove(key);
            size -= safeSizeOf(key, value);

            //移除元素的次数
            evictionCount++;
        }

        //此处 evicted 为 true，表明是为了腾出空间而进行的删除条目操作
        entryRemoved(true, key, value, null);
    }
}

/**
 * 调整缓存的大小
 */
public void resize(int maxSize) {
    if (maxSize <= 0) {
        throw new IllegalArgumentException("maxSize <= 0");
    }

    synchronized (this) {
        this.maxSize = maxSize;
    }
    trimToSize(maxSize);
}

```

### 查询
```Java
/**
 * 指定 key 对应的 value 值存在时返回，否则通过 create 方法创建相应的 key-value 对。
 * 如果对应的 value 值被返回，那么这个 key-value 对将被移到队尾。
 * 当返回 null 时，表明没有对应的 value 值并且也无法被创建
 */
public final V get(K key) {
    //缓存不允许 key 值为 null，因此对于查询 null 的键可直接抛出异常
    if (key == null) {
        throw new NullPointerException("key == null");
    }

    V mapValue;
    synchronized (this) {
        mapValue = map.get(key);
        //缓存命中
        if (mapValue != null) {
            hitCount++;
            return mapValue;
        }
        //缓存未命中
        missCount++;
    }

    /*
     * 尝试创建一个 value 值，这可能需要花费较长的时间完成，当 create 返回时，哈希表可能变得不同
     * 如果在 create 工作时向哈希表添加了一个冲突的值（key 已经有对应的 value 值，但 create 方法返回了一个不同的 value 值）
     * 那么将该值保留在哈希表中并释放创建的值。
     */
    V createdValue = create(key);
    if (createdValue == null) {
        return null;
    }

    synchronized (this) {
        //缓存创建的次数
        createCount++;
        mapValue = map.put(key, createdValue);

        if (mapValue != null) {
            // mapValue 不为 null，说明存在一个冲突值，保留之前的 value 值
            map.put(key, mapValue);
        } else {
            size += safeSizeOf(key, createdValue);
        }
    }

    if (mapValue != null) {
        entryRemoved(false, key, createdValue, mapValue);
        return mapValue;
    } else {
        trimToSize(maxSize);
        return createdValue;
    }
}

/**
 * 在缓存未命中之后调用以计算相应 key 的 value。
 * 当能计算 key 对应的 value 时，返回 value，否则返回 null。默认实现一律返回 null 值。
 *
 * 这个方法在被调用的时候没有添加额外的同步操作，因此其他线程可能在这个方法执行时访问缓存
 *
 * 如果 key 对应的 value 存储在缓存中，那么通过 create 创建的 value 将通过 entryRemoved 方法释放。
 * 这种情况主要发生在：当多个线程同时请求相同的 key （导致创建多个值）时，或者当一个线程调用 put 而另一个线程为其创建值时
 */
protected V create(K key) {
    return null;
}
```

### 存储
```Java
/**
 * 对于 key，缓存其相应的 value，key-value 条目放置于队尾
 *
 * @return 返回先前 key 对应的 value 值
 */
public final V put(K key, V value) {
    if (key == null || value == null) {
        throw new NullPointerException("key == null || value == null");
    }

    V previous;
    synchronized (this) {
        putCount++;
        size += safeSizeOf(key, value);
        previous = map.put(key, value);
        if (previous != null) {
            size -= safeSizeOf(key, previous);
        }
    }

    if (previous != null) {
        //evicted 为 true，表明不是为了腾出空间而进行的删除操作
        entryRemoved(false, key, previous, value);
    }

    trimToSize(maxSize);
    return previous;
}
```

### 删除
```Java
/**
 * 删除 key 对应的条目
 *
 * 返回 key 对应的 value值
 */
public final V remove(K key) {
    if (key == null) {
        throw new NullPointerException("key == null");
    }

    V previous;
    synchronized (this) {
        previous = map.remove(key);
        if (previous != null) {
            size -= safeSizeOf(key, previous);
        }
    }

    if (previous != null) {
        entryRemoved(false, key, previous, null);
    }

    return previous;
}

/**
 * 当条目需要被移除或删除时调用
 * 当一个值被移除以腾出空间，通过调用 remove 删除，或者被 put 调用替换值时，会调用此方法。默认实现什么也不做
 *
 * 这个方法在被调用的时候没有添加额外的同步操作，因此其他线程可能在这个方法执行时访问缓存
 *
 * @param evicted true 表明条目正在被删除以腾出空间，false 表明删除是由 put 或 remove 引起的（并非是为了腾出空间）
 *
 * @param newValue key 的新值。如果非 null，则此删除是由 put 引起的。否则它是由 remove引起的
 */
protected void entryRemoved(boolean evicted, K key, V oldValue, V newValue) {}
```