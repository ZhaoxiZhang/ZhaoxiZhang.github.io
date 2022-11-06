---
title: Java 8新特性
date: 2018-04-20
tags: Java
toc: true
---


本文译自[Java8-tutorial](https://github.com/winterbe/java8-tutorial)，并对其中内容进行了一些修改和补充。
<!-- more-->
## 接口的默认方法
在 Java 8 中，我们可以通过`default`关键字来为接口添加非抽象方法。`default`关键字修饰的方法称为默认方法，它允许我们添加新的功能到现有库的接口中，并能确保与采用旧版本接口编写的代码之间相互兼容。
对于以下例子：
```Java
interface Formula {
    double calculate(int a);

    default double sqrt(int a) {
        return Math.sqrt(a);
    }
}
```
在`Formula`接口中，除了有抽象方法`calculate`，还定义了默认方法`sqrt`，`Formula`的实现类只需实现抽象方法`calculate`，默认方法`sqrt`可直接使用接口中的定义，也可以在具体类中重写。
```Java
Formula formula = new Formula() {
    @Override
    public double calculate(int a) {
        return sqrt(a * 100);
    }
};

formula.calculate(100);     // 100.0
formula.sqrt(16);           // 4.0
```
上面的代码中，formula 以匿名对象的方式实现了`Formula`接口，而这只是为了实现`sqrt(a * 100)`，略显繁琐的，在下一部分，将讨论一种在 Java 8 中更为优雅的实现方式。

## Lambda表达式
首先让我们用 1.8 之前的 Java 版本来对一串字符串进行排序：
```Java
List<String> names = Arrays.asList("peter", "anna", "mike", "xenia");

Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return b.compareTo(a);
    }
});
```
静态工具方法`Collections.sort`接受一个列表和一个比较器来对给定的列表中的元素进行排序，你会发现你经常需要创建匿名比较器传给排序函数。
为了避免一直创建匿名对象，Java 8 通过**lambad 表达式**来简化语法规则：
```Java
Collections.sort(names, (String a, String b) -> {
    return b.compareTo(a);
});
```
上面的代码更加精简，可读性也更强，当然，还可以继续精简：
```Java
Collections.sort(names, (String a, String b) -> b.compareTo(a));
```
Lambda 表达式的主体只有一条语句时，花括号{}和`return`关键字可省略。
现在，列表有了一个`sort`方法，另外，当可以从上下文推断出参数的类型，同样可以省略掉参数类型。
### Lambad表达式的结构
- 一个 Lambda 表达式可以有零个或多个参数
- 参数的类型既可以明确声明，也可以根据上下文来推断。例如：(int a)与(a)效果相同
- 所有参数需包含在圆括号内，参数之间用逗号相隔。例如：(a, b) 或 (int a, int b) 或 (String a, int b, float c)
- 空圆括号代表参数集为空。例如：() -> 42
- 当只有一个参数，且其类型可推导时，圆括号（）可省略。例如：a -> return a*a
- Lambda 表达式的主体可包含零条或多条语句
- 如果 Lambda 表达式的主体只有一条语句，花括号{}可省略。匿名函数的返回类型与该主体表达式一致
- 如果 Lambda 表达式的主体包含一条以上语句，则表达式必须包含在花括号{}中（形成代码块）。匿名函数的返回类型与代码块的返回类型一致，若没有返回则为空

## 函数式接口
Lambda 表达式如何匹配 Java 的类型系统的呢？每个 Lambda 对应一个特定的接口，与一个给定的类型相匹配，一个所谓的函数式接口只包含一个抽象方法声明，每个 Lambda 表达式都与那个类型的抽象方法匹配。因为默认方法并非抽象的，因此我们可以向函数式接口任意添加默认方法。
我们可以使用任意只包含一个抽象方法声明的接口来作为 Lambda 表达式，为了确保使用的是函数式接口，我们可以添加`@FunctionalInterface`注解，编译器就会察觉到这个注解，并且当我们尝试往函数式接口添加第二个抽象方法声明时抛出异常。
```Java
@FunctionalInterface
interface Converter<F, T> {
    T convert(F from);
}

Converter<String, Integer> converter = (from) -> Integer.valueOf(from);
Integer converted = converter.convert("123");
System.out.println(converted);    // 123
```
假使没有`@FunctionalInterface`注解，上述代码仍然是正确的。

## 方法和构造器引用
方法引用的分类：
- 类名::静态方法名
- 对象::实例方法名
- 类名::实例方法名
- 类名::new

### 类名::静态方法名
上述例子中的代码可以进一步通过静态方法引用来精简：
```Java
Converter<String, Integer> converter = Integer::valueOf;
Integer converted = converter.convert("123");
System.out.println(converted);   // 123
```
Java 8 中你可以通过`::`关键字来传递方法或者构造器引用，上述的例子说明了如何引用一个静态方法。
### 对象::实例方法名
我们也可以引用一个对象方法：
```Java
class Something {
    String startsWith(String s) {
        return String.valueOf(s.charAt(0));
    }
}
```
```Java
Something something = new Something();
Converter<String, String> converter = something::startsWith;
String converted = converter.convert("Java");
System.out.println(converted);    // "J"
```
### 类名::实例方法名
```Java
public class Person {
    private String name;

    public Person() {

    }

    public Student(String name){
        this.name = name;
    }

    public int compareByScore(Student student){
        return this.getScore() - student.getScore();
    }
}
```
```Java
students.sort(Student::compareByScore);
students.forEach(student -> System.out.println(student.getScore()));
```
sort 方法接收的 Lambda 表达式本该有两个参数，而这个实例方法只有一个参数也满足 Lambda 表达式的定义。这就是 类名::实例方法名 这种方法引用的特殊之处，当使用 类名::实例方法名 方法引用时，一定是 Lambda 表达式所接收的第一个参数来调用实例方法，如果 Lambda 表达式接收多个参数，其余的参数作为方法的参数传递进去。
### 类名::new
让我们来看看`::`关键字在构造器引用中是如何使用的。首先，我们定义一个有多个构造函数的类：
```Java
class Person {
    String firstName;
    String lastName;

    Person() {}

    Person(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }
}
```
接下来，我们创建一个用于创建新人员的工厂接口：
```Java
interface PersonFactory<P extends Person> {
    P create(String firstName, String lastName);
}
```
除了传统方式实现工厂接口外，通过构造器引用的方式
```Java
PersonFactory<Person> personFactory = Person::new;
Person person = personFactory.create("Peter", "Parker");
```
我们通过`Person::new`向 Person 构造器传了一个引用（注：Person类中需要有无参构造器），Java编译器会自动选择正确的构造器。

## Lambda作用域
Lambda 表达式访问外部变量的方式与匿名对象非常相似，它可以访问局部外围的 final 变量、成员变量和静态变量。
### 访问局部变量
我们可以在 Lambda 表达式所在的外部范围访问`final`修饰的局部变量
```Java
final int num = 1;
Converter<Integer, String> stringConverter =
        (from) -> String.valueOf(from + num);

stringConverter.convert(2);     // 3
```
不同于匿名对象的是，上述变量 num 不一定要被声明为 final（匿名内部类中的参数必须声明为 final，其值是 capture by value的），下述代码也是正确的：
```Java
int num = 1;
Converter<Integer, String> stringConverter =
        (from) -> String.valueOf(from + num);

stringConverter.convert(2);     // 3
```
值得注意的是，虽然 num 变量不需要显式声明为 final，但实际上，编译器要求 Lambda 表达式中捕获的变量必须实际上是最终变量（也就是初始化后不可再赋新值）所以 num 不可更改，下述代码无法通过编译，原因就是 num 的值被更改了：
```Java
int num = 1;
Converter<Integer, String> stringConverter =
        (from) -> String.valueOf(from + num);
num = 3;
```
### 访问成员变量和静态变量
与局部变量不同的是，Lambda 表达式中，可以对成员变量和静态变量进行读和写操作。
```Java
class Lambda4 {
    static int outerStaticNum;
    int outerNum;

    void testScopes() {
        Converter<Integer, String> stringConverter1 = (from) -> {
            outerNum = 23;
            return String.valueOf(from);
        };

        Converter<Integer, String> stringConverter2 = (from) -> {
            outerStaticNum = 72;
            return String.valueOf(from);
        };
    }
}
```
### 访问接口的默认方法
在第一部分中关于 formula 的例子，`Formula`接口定义了一个`sqrt`的默认方法，其可以被任意一个 formula 实例包括匿名对象访问，但是在 Lambda 表达式中却不行，Lambda 表达式无法访问接口的默认方法，下述代码是错误的：
```Java
Formula formula = (a) -> sqrt(a * 100);
```
## 内置函数式接口
JDK 1.8 API 中包含了很多内置的函数式接口，其中一部分例如`Comparator`、`Runnable`在之前的 JDK 版本中就被人熟知。这些现有的接口通过`@FunctionalInterface`注解被拓展来支持 Lambda。
Java 8中的 API 也提供了一些新的函数式接口来使得编程更加简单。
以下是常用的函数式接口

| 函数式接口             | 参数类型 | 返回类型    | 抽象方法名  | 描述             | 其他方法                     |
| ----------------- | ---- | ------- | ------ | -------------- | ------------------------ |
| Runnable          | 无    | void    | run    | 作为无参数或返回值的动作运行 |                          |
| Supplier<T>       | 无    | T       | get    | 提供一个T类型的值      |                          |
| Consumer<T>       | T    | void    | accept | 处理一个T类型的值      | andThen                  |
| BiConsumer<T,U>   | T,U  | void    | accept | 处理T和U类型的值      | andThen                  |
| Function<T,R>     | T    | R       | apply  | 有一个T类型参数的函数    | compose,andThen,identity |
| BiFunction<T,U,R> | T,U  | R       | apply  | 有T和U类型参数的函数    | andThen                  |
| UnaryOperator<T>  | T    | T       | apply  | 类型T上的一元操作符     | compose,andThen,identity |
| BinaryOperator<T> | T,T  | T       | apply  | 类型T上的二元操作符     | andThen,maxBy,minBy      |
| Predicate<T>      | T    | boolean | test   | 布尔值函数          | and,or,negate,isEqual    |
| BiPredicate<T,U>  | T,U  | boolean | test   | 有两个参数的布尔值函数    | and,or,negate            |

### Predicate
`Predicate` 是一个布尔类型的函数，该函数只有一个参数，该接口包含了多种默认方法，用于处理复杂的逻辑动词。
```Java
Predicate<String> predicate = (s) -> s.length() > 0;

predicate.test("foo");              // true
predicate.negate().test("foo");     // false

Predicate<Boolean> nonNull = Objects::nonNull;
Predicate<Boolean> isNull = Objects::isNull;

Predicate<String> isEmpty = String::isEmpty;
Predicate<String> isNotEmpty = isEmpty.negate();
```
### Function
`Function`接受一个参数并且返回一个结果，可以使用默认方法（compose，andThen）将多个函数链接起来。
```Java
Function<String, Integer> toInteger = Integer::valueOf;
Function<String, String> backToString = toInteger.andThen(String::valueOf);

backToString.apply("123");     // "123"
```
### Supplier
`Supplier`返回一个给定类型的结果，与`Function`不同的是，`Supplier`不接受任何参数。
```Java
Supplier<Person> personSupplier = Person::new;
personSupplier.get();   // new Person
```
### Consumer
`Comsumer`代表了在一个输入参数上需要进行的操作.
```Java
Consumer<Person> greeter = (p) -> System.out.println("Hello, " + p.firstName);
greeter.accept(new Person("Luke", "Skywalker"));
```
### Comparator
`Comparator`在之前的 Java 版本就已经被熟知，Java 8 在这个接口中增加了多个默认方法。
```Java
Comparator<Person> comparator = (p1, p2) -> p1.firstName.compareTo(p2.firstName);

Person p1 = new Person("John", "Doe");
Person p2 = new Person("Alice", "Wonderland");

comparator.compare(p1, p2);             // > 0
comparator.reversed().compare(p1, p2);  // < 0
```
## Optional
`Optional`并非是一个函数式接口，但却是一个精巧的工具接口，用来防止`NullPointerException`，这个概念对于下一部分显得很重要，所以我们在这快速浏览一下`Optional`是如何工作的。
`Optional`是一个简单的值容器，这个值可以是 null，也可以是 non-null 的。考虑一个方法可能返回一个 non-null 值的结果，也有可能返回一个空值。在 Java 8中，为了不直接返回 null，你可以返回一个`Optional`。
```Java
Optional<String> optional = Optional.of("bam");

optional.isPresent();           // true
optional.get();                 // "bam"
optional.orElse("fallback");    // "bam"

optional.ifPresent((s) -> System.out.println(s.charAt(0)));     // "b"
```
## Streams
`java.util.Stream`代表了可以在其上面执行一个或多个操作的元素序列。流操作是中间或者完结操作。完结操作会返回一个某种类型的值，而中间操作会返回流本身，因此你可以连续链接多个方法的调用。Stream 是在一个源的基础上创建出来的，例如`java.util.Collection`中的 lists 或 sets（不支持 maps）。流操作可以被顺序或者并行执行。
让我们先来了解下序列流是如何工作的，首先，我们通过字符串列表的形式创建一个示例代码：
```Java
List<String> stringCollection = new ArrayList<>();
stringCollection.add("ddd2");
stringCollection.add("aaa2");
stringCollection.add("bbb1");
stringCollection.add("aaa1");
stringCollection.add("bbb3");
stringCollection.add("ccc");
stringCollection.add("bbb2");
stringCollection.add("ddd1");
```
Java 8 中的集合已被拓展，因此你可以直接调用`Collection.stream()·`或`Collection.parallelStream()`来创建流。接下来的部分将会解释最常用的流操作。
### Filter
Filter 接受一个 predicate 类型的接口来过滤流中的元素。该操作是一个中间操作，因此它允许我们在返回结果的时候再调用其他流操作（forEach）。ForEach 接受一个 Consumer 类型的接口变量，用来执行对多虑的流中的每一个元素的操作。ForEach是一个完结操作，并且不返回流，因此我们不能再调用其他的流操作。
```Java
stringCollection
    .stream()
    .filter((s) -> s.startsWith("a"))
    .forEach(System.out::println);

// "aaa2", "aaa1"
```
### Sorted
Sorted 是一个中间操作，其返回一个流排序后的视图，流中的元素默认按照自然顺序进行排序，除非你指定了一个`Comparator`接口来重定义排序规则。
```Java
stringCollection
    .stream()
    .sorted()
    .filter((s) -> s.startsWith("a"))
    .forEach(System.out::println);

// "aaa1", "aaa2"
```
需要注意的是，`sorted`只是创建了流排序后的视图，并没有操作操作集合，集合中元素的顺序是没有改变的。
```Java
System.out.println(stringCollection);
// ddd2, aaa2, bbb1, aaa1, bbb3, ccc, bbb2, ddd1
```
### Map
中间操作`map`通过特定的接口将每个元素转换为另一个对象，下面的例子将每一个字符串转换为全为大写的字符串。当然，你可以使用`map`将每一个对象转换为其他类型。对于带泛型结果的流对象，具体的类型还要由传递给 map 的泛型方法来决定。
```Java
stringCollection
    .stream()
    .map(String::toUpperCase)
    .sorted((a, b) -> b.compareTo(a))
    .forEach(System.out::println);

// "DDD2", "DDD1", "CCC", "BBB3", "BBB2", "AAA2", "AAA1"
```
### Match
有多种匹配操作可以用来检查某一种规则是否与流对象相匹配。所有的匹配操作都是完结操作，并且返回一个 boolean 类型的结果。
```Java
boolean anyStartsWithA =
    stringCollection
        .stream()
        .anyMatch((s) -> s.startsWith("a"));

System.out.println(anyStartsWithA);      // true

boolean allStartsWithA =
    stringCollection
        .stream()
        .allMatch((s) -> s.startsWith("a"));

System.out.println(allStartsWithA);      // false

boolean noneStartsWithZ =
    stringCollection
        .stream()
        .noneMatch((s) -> s.startsWith("z"));

System.out.println(noneStartsWithZ);      // true
```
### Count
Count 是一个完结操作，它返回一个 long 类型数值，用来标识流对象中包含的元素数量。
```Java
long startsWithB =
    stringCollection
        .stream()
        .filter((s) -> s.startsWith("b"))
        .count();

System.out.println(startsWithB);    // 3
```
### Reduce
这个完结操作通过给定的函数来对流元素进行削减操作，该缩减操作的结果保存在`Optional`变量中。
```Java
Optional<String> reduced =
    stringCollection
        .stream()
        .sorted()
        .reduce((s1, s2) -> s1 + "#" + s2);

reduced.ifPresent(System.out::println);
// "aaa1#aaa2#bbb1#bbb2#bbb3#ccc#ddd1#ddd2"
```
## Parallel Streams
正如上面提到的，stream 可以是顺序的也可以是并行的。顺序操作通过单线程执行，而并行操作通过多线程执行。
下面的例子说明了使用并行流提高运行效率是多么的容易。
首先我们创建一个包含不同元素的列表：
```Java
int max = 1000000;
List<String> values = new ArrayList<>(max);
for (int i = 0; i < max; i++) {
    UUID uuid = UUID.randomUUID();
    values.add(uuid.toString());
}
```
现在我们测量一下对这个集合进行排序需要花的时间。
- Sequential Sort

```Java
long t0 = System.nanoTime();

long count = values.stream().sorted().count();
System.out.println(count);

long t1 = System.nanoTime();

long millis = TimeUnit.NANOSECONDS.toMillis(t1 - t0);
System.out.println(String.format("sequential sort took: %d ms", millis));

// sequential sort took: 899 ms
```
- Parallel Sort

```Java
long t0 = System.nanoTime();

long count = values.parallelStream().sorted().count();
System.out.println(count);

long t1 = System.nanoTime();

long millis = TimeUnit.NANOSECONDS.toMillis(t1 - t0);
System.out.println(String.format("parallel sort took: %d ms", millis));

// parallel sort took: 472 ms
```
两个代码片段几乎一样，但是使用并行操作来排序的效率提高了接近一半，而你需要做得就仅是将`stream`替换为`parallelStream`
## Map
正如前面提到的，map 是不支持流操作的。`Map`接口本身没有可用的`stream()`方法，但是你可以根据键-值对或项通过`map.keySet().stream`，`map.values().stream()`和`map.entrySet().stream()`来创建指定的流。
此外，map 支持多种新的、有用的方法来处理常规任务。
```Java
Map<Integer, String> map = new HashMap<>();

for (int i = 0; i < 10; i++) {
    map.putIfAbsent(i, "val" + i);
}

map.forEach((id, val) -> System.out.println(val));
```
上面的代码是自解释的，`putIfAbsent`防止我们写入额外的空值检查，`forEach`接受一个 Consumer 为 map 中的每一个值进行操作。
下面的例子说明了如何利用函数来计算 map 上的代码
```Java
map.computeIfPresent(3, (num, val) -> val + num);
map.get(3);             // val33

map.computeIfPresent(9, (num, val) -> null);
map.containsKey(9);     // false

map.computeIfAbsent(23, num -> "val" + num);
map.containsKey(23);    // true

map.computeIfAbsent(3, num -> "bam");
map.get(3);             // val33
```
接下来，我们学习如何删除给定键的条目，只有当前键值映射到给定值时，才能删除指定条目
```Java
map.remove(3, "val3");
map.get(3);             // val33

map.remove(3, "val33");
map.get(3);             // null
```
另一个有用的方法：
```Java
map.getOrDefault(42, "not found");  // not found
```
合并一个 map 的条目是很简单的：
```Java
map.merge(9, "val9", (value, newValue) -> value.concat(newValue));
map.get(9);             // val9

map.merge(9, "concat", (value, newValue) -> value.concat(newValue));
map.get(9);             // val9concat
```
如果不存在该键值的条目，合并或者将键/值放入 map 中，或者调用合并函数来更改现有值。
## 日期API
Java 8 在`java.time`包下包含了全新的日期和时间 API，这个新的日期 API 与 [Joda-Time 库](http://www.joda.org/joda-time/)相似，但不完全一样。下面的例子涵盖了大部分新的 API。
### Clock
Clock 提供了对当前日期和时间的访问，Clocks 知道当前时区，可以使用它替代`
System.currentTimeMillis()`来获取当前的毫秒时间。时间线上的某一时刻也由类`Instant`表示，Instants 可以用来创建遗留的`java.util.Date`对象。
```Java
Clock clock = Clock.systemDefaultZone();
long millis = clock.millis();

Instant instant = clock.instant();
Date legacyDate = Date.from(instant);   // legacy java.util.Date
```
### Timezones
Timezone 由一个`ZoneId`来表示，他们可以通过静态工厂方法获得。时区定义了某一时刻和当地日期、时间之间转换的偏移量。
```Java
System.out.println(ZoneId.getAvailableZoneIds());
// prints all available timezone ids

ZoneId zone1 = ZoneId.of("Europe/Berlin");
ZoneId zone2 = ZoneId.of("Brazil/East");
System.out.println(zone1.getRules());
System.out.println(zone2.getRules());

// ZoneRules[currentStandardOffset=+01:00]
// ZoneRules[currentStandardOffset=-03:00]
```
### LocalTime
LocalTime 表示了一个没有指定时区的时间，例如 10 p.m 或者 17：30：15。下面的例子为上面定义的时区创建了两个本地时间，然后我们比较两个时间，并计算它们之间的小时和分钟之间的不同。
```Java
LocalTime now1 = LocalTime.now(zone1);
LocalTime now2 = LocalTime.now(zone2);

System.out.println(now1.isBefore(now2));  // false

long hoursBetween = ChronoUnit.HOURS.between(now1, now2);
long minutesBetween = ChronoUnit.MINUTES.between(now1, now2);

System.out.println(hoursBetween);       // -3
System.out.println(minutesBetween);     // -239
```
`LocalTime`带有多种工厂方法，以简化新实例的创建，包括对时间字符串进行解析操作。
```Java
LocalTime late = LocalTime.of(23, 59, 59);
System.out.println(late);       // 23:59:59

DateTimeFormatter germanFormatter =
    DateTimeFormatter
        .ofLocalizedTime(FormatStyle.SHORT)
        .withLocale(Locale.GERMAN);

LocalTime leetTime = LocalTime.parse("13:37", germanFormatter);
System.out.println(leetTime);   // 13:37
```
### LocalDate
LocalDate 表示不同的日期，例如2014-03-11。它是不可变的，并且与`LocalTime`完全类似。下面的例子演示了如何通过加减日、月、年来计算新日期。需要注意的是，每一个操作都会返回一个新实例。
```Java
LocalDate today = LocalDate.now();
LocalDate tomorrow = today.plus(1, ChronoUnit.DAYS);
LocalDate yesterday = tomorrow.minusDays(2);

LocalDate independenceDay = LocalDate.of(2014, Month.JULY, 4);
DayOfWeek dayOfWeek = independenceDay.getDayOfWeek();
System.out.println(dayOfWeek);    // FRIDAY
```
从字符串中解析 LocalDate 就跟解析 LocalTime 一样简单：
```Java
DateTimeFormatter germanFormatter =
    DateTimeFormatter
        .ofLocalizedDate(FormatStyle.MEDIUM)
        .withLocale(Locale.GERMAN);

LocalDate xmas = LocalDate.parse("24.12.2014", germanFormatter);
System.out.println(xmas);   // 2014-12-24
```
### LocalDateTime
LocalDateTIme 表示的是日期-时间。它将日期和时间组合成一个实例。`LocalDateTime`是不可变的，与 `LocalTime`和`LocalDate`工作原理类似。我们可以利用方法去获取日期时间中的某些字段值。
```Java
LocalDateTime sylvester = LocalDateTime.of(2014, Month.DECEMBER, 31, 23, 59, 59);

DayOfWeek dayOfWeek = sylvester.getDayOfWeek();
System.out.println(dayOfWeek);      // WEDNESDAY

Month month = sylvester.getMonth();
System.out.println(month);          // DECEMBER

long minuteOfDay = sylvester.getLong(ChronoField.MINUTE_OF_DAY);
System.out.println(minuteOfDay);    // 1439
```
通过一个时区的附件信息可以转换为一个实例，这个实例很容易转为`java.util.Date`类型。
```Java
Instant instant = sylvester
        .atZone(ZoneId.systemDefault())
        .toInstant();

Date legacyDate = Date.from(instant);
System.out.println(legacyDate);     // Wed Dec 31 23:59:59 CET 2014
```
日期-时间的格式化类似于 Date 或 Time。我们可以使用自定义模式来取代预定义的格式进行格式化。
```Java
DateTimeFormatter formatter =
    DateTimeFormatter
        .ofPattern("MMM dd, yyyy - HH:mm");

LocalDateTime parsed = LocalDateTime.parse("Nov 03, 2014 - 07:13", formatter);
String string = formatter.format(parsed);
System.out.println(string);     // Nov 03, 2014 - 07:13
```
不像`java.text.NumberFormat`，`DateTimeFormatter`是不可变的并且是线程安全的。
了解更多有关日期格式化的信息可以参考[这里](https://docs.oracle.com/javase/8/docs/api/java/time/format/DateTimeFormatter.html)

## 注解
Java 8中的注解是可重复的，我们直接通过一个例子来了解它。
首先，我们定义了一个包装注解，它包括了一个实际注解的数组：
```Java
@interface Hints {
    Hint[] value();
}

@Repeatable(Hints.class)
@interface Hint {
    String value();
}
```
Java 8 允许我们通过使用`@Repeatable`对同一类型使用多个注解
- 变体一：使用注解容器（老方法）
```Java
@Hints({@Hint("hint1"), @Hint("hint2")})
class Person {}
```
- 变体二：使用可重复注解（新方法）
```Java
@Hint("hint1")
@Hint("hint2")
class Person {}
```

使用变体2，Java 编译器隐式地对`@Hint`进行设置，这对于通过反射来读取注解信息非常重要。
```Java
Hint hint = Person.class.getAnnotation(Hint.class);
System.out.println(hint);                   // null

Hints hints1 = Person.class.getAnnotation(Hints.class);
System.out.println(hints1.value().length);  // 2

Hint[] hints2 = Person.class.getAnnotationsByType(Hint.class);
System.out.println(hints2.length);          // 2
```
尽管我们不会在`Person`类中声明`@Hints`注解，但是它仍然可以通过`getAnnotation(Hint.class)`来读取。然后，更便利的方法是`getAnnotationByType`，它可以直接访问`@Hint`注解。
此外，Java 8 中关于注解的使用，其还拓展了两个新的目标：
```Java
@Target({ElementType.TYPE_PARAMETER, ElementType.TYPE_USE})
@interface MyAnnotation {}
```