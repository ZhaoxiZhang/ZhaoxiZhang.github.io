---
title: Java设计模式——单例模式（创建型模式）
date: 2018-06-14
tags: [Java,设计模式]
toc: true
---
## 概述
&nbsp;&nbsp;单例模式保证对于每一个类加载器，一个类仅有一个实例并且提供全局的访问。其是一种对象创建型模式。对于单例模式主要适用以下几个场景：<!--more-->
- 系统只需要一个实例对象，如提供一个唯一的序列号生成器
- 客户调用类的单个实例只允许使用一个公共访问点，除了该公共访问点，不能通过其他途径访问该实例


&nbsp;&nbsp;单例模式的缺点之一是在分布式环境中，如果因为单例模式而产生 bugs，那么很难通过调试找出问题所在，因为在单个类加载器下进行调试，并不会出现问题。


## 实现方式
&nbsp;&nbsp;一般来说，实现枚举有五种方式：饿汉式、懒汉式、双重锁检验、静态内部类、枚举，而这里我将这五种方式分为三部分来介绍。
### 饿汉式加载
```java
public class Singleton {
    //私有构造器，所以无法实例化类对象
    private Singleton() {}

    //类静态实例域
    private static final Singleton INSTANCE = new Singleton();

    //返回类实例
    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```
直接初始化静态实例保证了线程安全，但是此种方式不是懒加载的，单例一开始就初始化了，无法在我们需要的时候再进行初始化。

### 懒汉式加载
```java
//实例在这个方法第一次被调用的时候进行初始化
public static synchronized Singleton getInstance() {
    if (instance == null) {
        instance = new Singleton();
    }
    return instance;
}
```
`getInstance()`方法设置为`synchronized`保证了线程安全，但是其效率并不高，因为在任何时候只有一个线程能够访问这个方法，而同步操作仅需在第一次被调用的时候才被需要。

此方法的一种改进是使用双重锁检验。
```java
public class ThreadSafeDoubleCheckLocking {

    private static volatile ThreadSafeDoubleCheckLocking instance;

    private ThreadSafeDoubleCheckLocking() {}

    public static ThreadSafeDoubleCheckLocking getInstance() {
        //局部变量可以提高25%的性能，这个局部变量确保instance只在已经被初始化的情况下读取一次
        //《Effective Java 第2版》P250页
        ThreadSafeDoubleCheckLocking result = instance;
        //检查实例是否已经别初始化
        if (result == null) {
            //未被初始化，但是无法确定这时其他线程是否已经对其初始化，因此添加对象锁进行互斥
            synchronized (ThreadSafeDoubleCheckLocking.class) {
                //再一次将instance赋值给局部变量来进行检查，因为有可能在当前线程阻塞的时候，其他线程对instance进行初始化
                result = instance;
                if (result == null) {
                    //此时还未被初始化的话，在这里初始化可以保证线程安全
                    instance = result = new ThreadSafeDoubleCheckLocking();
                }
            }
        }
        return result;
    }
}
```
上面的双重锁检验使用了《Effective Java 第2版》提出的一个优化方式，另外值得一提的是，对于`instance`域被声明为`volatile`是很重要的。当一个变量定义为`volatile`之后，它就具备了两种特性，第一是保证了此变量对所有线程的可见性，“可见性”指的是当一条线程修改了这个变量的值，新值对于其他线程来说是可以立即得知的（注意基于volatile变量的运算在并发编程下并非是安全的，例如：假设被volatile修饰的域进行自增运算，而自增运算并不是原子操作，那么第二个线程就可能在读取旧值和写回新值的期间读取到这个域，导致第二个线程看到的值与第一个线程未自增前的值一样，详细了解的话可查看《深入理解Java虚拟机 第2版》P366 基于volatile型变量的特殊规则）；第二是禁止指令重排序优化。
在进行初始化的时候`instance = result = new ThreadSafeDoubleCheckLocking()`，此时 JVM 大致做了三件事：
- 1.给instance分配内存
- 2.调用构造函数进行初始化
- 3.instance对象指向被分配的内存

没有声明为`volatile`，那么指令重排序后，可能执行的顺序是 1-3-2，当线程一执行到3这个步骤，还未执行步骤2（instance非null，但未初始化），那么对于线程二，此时检测到 instance 并非是 null，直接返回 instance，就会出现错误。需要说明的一点是，JDK 1.5以后，`volatile`才真正发挥用处，因此在1.5以前，仍然是无法保证安全的，具体可查看 [The "Double-Checked Locking is Broken" Declaration](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html).

另外一种懒加载方式就是使用静态内部类的方法：
```java
public class InitializingOnDemandHolderIdiom {
    private InitializingOnDemandHolderIdiom() {}

    public static InitializingOnDemandHolderIdiom getInstance() {
        return HelperHolder.INSTANCE;
    }

    private static class HelperHolder {
        private static final InitializingOnDemandHolderIdiom INSTANCE =
                new InitializingOnDemandHolderIdiom();
    }
}
```
这种方式是线程安全的，同时也是懒加载的。`HelperHolder`是私有的，除了`getInstance()`外没有办法访问。这种方式不需要依赖其他语言特性（volatile，synchronized），也不依赖JDK版本。

### 枚举
《Effective Java 第2版》P15 中提到实现单例的一种新方式，使用枚举来实现单例。枚举类型是Java 5中新增特性的一部分，因此使用这种方式实现的枚举，要求至少是 JDK 1.5版本及其以上。枚举本身保证了线程安全，并且提供了序列化机制，因此这种方式写起来极为简洁。
```java
public enum Singleton {
    INSTANCE;
}
```
当然，对于使用枚举来实现单例模式也有一些缺点，具体可以查看 [StackOverflow](https://softwareengineering.stackexchange.com/questions/179386/what-are-the-downsides-of-implementing-a-singleton-with-javas-enum) 的讨论。

## 典型使用场景
- 日志纪录类
- 管理与数据库的连接
- 文件管理系统

## 具体实例
[java.lang.Runtime#getRuntime()](https://docs.oracle.com/javase/8/docs/api/java/lang/Runtime.html#getRuntime%28%29)
[java.awt.Desktop#getDesktop()](https://docs.oracle.com/javase/8/docs/api/java/awt/Desktop.html#getDesktop--)
[java.lang.System#getSecurityManager()](https://docs.oracle.com/javase/8/docs/api/java/lang/System.html#getSecurityManager--)

## 参考资料
- [java-design-patterns](https://github.com/iluwatar/java-design-patterns/tree/master/singleton)
- 《Effective Java 第2版》