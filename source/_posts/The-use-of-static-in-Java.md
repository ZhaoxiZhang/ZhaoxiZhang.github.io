---
title: Java中static的用法
date: 2018-05-31
tags: Java
toc: true
---

## 静态域
&nbsp;&nbsp;如果将域定义为 static，每个类中只有一个这样的域。而每一个对象对于所有的实例域却都有自己的一份拷贝。例如，假定需要给每一个雇员赋予唯一的标识码，这里给 Employee 类添加一个实例域 id 和一个静态域 nextId：
<!--more-->
```java
class Employee{
    private static int nextId = 1;
    private int id;
}
```
&nbsp;&nbsp;现在，每一个雇员对象都有一个自己的 id 域，但这个类的所有实例将共享一个 nextId。nextId 属于类，而不属于任何一个对象，没有对象的存在，静态域 nextId 也存在。
&nbsp;&nbsp;对于静态变量，静态常量在 Java 中更为常见，我们平常中比较熟悉的 Math 类中就定义有静态常量供我们直接使用：
```java
public final class{
    public static final double E = 2.7182818284590452354;
    public static final double PI = 3.14159265358979323846;
}
```
对于静态域，其使用场景一般用于：
- 在对象之间共享数据
- 方便访问

## 静态方法
&nbsp;&nbsp;同样，我们仍然可以在 Math 类中看到很多的静态方法：
```java
public final class{
    public static double sin(double a) {
        return StrictMath.sin(a);
    }

    public static double cos(double a) {
        return StrictMath.cos(a);
    }
}
```
&nbsp;&nbsp;对于静态方法的调用，我们可以使用“类名.方法名”的方式，也可以使用“对象名.方法名”的方式调用，静态方法在访问本类的成员的时候，只允许访问静态成员（即静态变（常）量和静态方法），而不允许访问实例成员，因为实例成员与具体的对象关联，而静态方法属于类，不与任何对象绑定。
## 静态代码块
&nbsp;&nbsp;可以将多个静态初始化动作组织成一个特殊的“静态子句”构成静态代码块：
```java
public class Test{
    static int i;
    static{
        i = 1;
    }
}
```
对于上述代码，实际就是一段跟在 static 关键字后面的代码，与其他静态初始化动作一样，这段代码只执行一次：当首次生成这个类的一个对象时，或者首次访问属于那个类的静态数据成员时（即便从未生成过那个类的对象）
```java
class Cup{
    Cup(int marker){
        System.out.println("Cup(" + marker + ")");
    }
    void f(int marker){
        System.out.println("f(" + marker + ")");
    }
}

class Cups{
    static Cup cup1;
    static Cup cup2;
    public static final String HELLOWORLD = "hello world";
    static {
        cup1 = new Cup(1);
        cup2 = new Cup(2);
    }
    Cups(){
        System.out.println("Cups()");
    }
}

public class ExplicitStatic {
    public static void main(String... args){
        System.out.println("Inside main()");
        //Cups.cup1.f(99);   //(1)
        //Cups cups = new Cups();  //(2)
        System.out.println(Cups.HELLOWORLD);  //(3)
    }
}

/*执行（1）处的代码，其输出如下：
    Inside main()
    Cup(1)
    Cup(2)
    f(99)

执行（2）处的代码，其输出如下：
    Inside main()
    Cup(1)
    Cup(2)
    Cups()
执行（3）处的代码，其输出如下：
    Inside main()
    hello world
*/
```
值得注意的是，对于执行（2）的代码，**静态代码块中的内容优于构造器先执行**。另外，如果（1），（2）的代码都不执行，那么静态代码块中的代码也不会得到执行。
此外，可能令人不解的是，对于执行（3）处的代码，其为什么没有输出静态代码块中的内容？这是因为虽然在Java源码中引用了 Cups 类中的常量 HELLOWORLD，但其实在编译阶段通过常量传播优化，已经将此常量的值“hello world”存储到了 ExpliciStatic 类的常量池，以后 ExpliciStatic 对常量 Cups.HELLOWORLD 的引用实际都转化为 ExpliciStatic 类对自身常量池的引用了。这点在《深入理解Java虚拟机》中虚拟机类加载机制一章中有进行阐述。
## static与final
static与final连用修饰成员，一般作为全局常量使用，其修饰的变量无法修改，修饰的方法无法被覆盖。

## static常见面试题
- 非静态内部类里面为什么不能有静态属性和静态方法
static 类型的属性和方法在类加载的时候就会存在于内存中，要使用某个类的 static 属性或者方法的前提是这个类已经加载到 JVM 中，非 static 内部类默认是持有外部类的引用且依赖外部类存在，所以如果一个非 static 的内部类一旦具有 static 的属性或者方法就会出现内部类未加载时却试图在内存中创建内部类的 static 属性和方法，这自然是错误的，类都不存在（没被加载）却希望操作它的属性和方法。从另一个角度讲非 static 的内部类在实例化的时候才会加载（不自动跟随主类加载），而 static 的语义是类能直接通过类名来访问类的 static 属性或者方法，所以如果没有实例化非 static 的内部类就等于非 static 的内部类没有被加载，所以无从谈起通过类名访问 static 属性或者方法。