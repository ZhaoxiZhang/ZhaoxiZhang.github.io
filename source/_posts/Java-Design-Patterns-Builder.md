---
title: Java设计模式——建造者模式（创建型模式）
date: 2018-06-17
tags: [Java,设计模式]
toc: true
---
## 概述
&nbsp;&nbsp;建造者模式也称为生成器模式，是一种对象创建型模式，它可以将复杂对象的建造过程抽象出来（抽象类别），使这个抽象过程的不同实现方法可以构造出不同表现（属性）的对象。
<!--more-->
&nbsp;&nbsp;建造者模式意在为重叠构造器这种反模式(telescoping constructor anti-pattern)找到一种解决方案，对于重叠构造器反模式，我们经常能看到类似于下列的构造器形式（下述例子来源于《Effective Java》）：
```Java
public NutritionFacts(int servingSize, int servings) {
    this(servingSize, servings, 0);
}

public NutritionFacts(int servingSize, int servings, int calories) {
    this(servingSize, servings, calories, 0);
}

public NutritionFacts(int servingSize, int servings, int calories, int fat) {
    this(servingSize, servings, calories, fat, 0);
}

public NutritionFacts(int servingSize, int servings, int calories, int fat,
                      int sodium) {
    this(servingSize, servings, calories, fat, sodium, 0);
}

public NutritionFacts(int servingSize, int servings, int calories, int fat,
                      int sodium, int carbohydrate) {
    this.servingSize = servingSize;
    this.servings = servings;
    this.calories = calories;
    this.fat = fat;
    this.sodium = sodium;
    this.carbohydrate = carbohydrate;
}
```
&nbsp;&nbsp;如上所见，这些构造器可能包含了一些我们并不想设置的参数，但是还是不得不为其传递值，并且，当参数一多，代码就会变得难以阅读，时常出现无法知道某一个值的具体含义，必须仔细对照构造器来查找。伴随着参数的增多，这种方式的构造器将很快是去控制。
&nbsp;&nbsp;因此当遇到这种许多构造器参数的时候，可以选用建造者模式。

## 实现方式
```Java
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        // 必要参数
        private final int servingSize;
        private final int servings;

        // 可选参数
        private int calories = 0;
        private int fat = 0;
        private int carbohydrate = 0;
        private int sodium = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }

        public Builder calories(int val) {
            calories = val;
            return this;
        }

        public Builder fat(int val) {
            fat = val;
            return this;
        }

        public Builder carbohydrate(int val) {
            carbohydrate = val;
            return this;
        }

        public Builder sodium(int val) {
            sodium = val;
            return this;
        }

        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }

    public static void main(String[] args) {
        NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
                .calories(100).sodium(35).carbohydrate(27).build();
    }
}
```
&nbsp;&nbsp;如上代码所示，这样的客户端代码容易编写，并且易于阅读。对于参数值，可以单独设置组合，不再需要传递不必要的参数值。

## 适用环境
- 需要生成的产品对象有复杂的内部结构，这些产品对象通常包含多个成员属性
- 构建的过程允许对构建的对象进行不同的表示

## 具体实例
- [java.lang.StringBuilder](https://docs.oracle.com/javase/8/docs/api/java/lang/StringBuilder.html)
- [java.lang.StringBuffer](https://docs.oracle.com/javase/8/docs/api/java/lang/StringBuffer.html#append-boolean-)

## 参考资料
- [java-design-patterns](https://github.com/iluwatar/java-design-patterns/tree/master/singleton)
- 《Effective Java 第2版》