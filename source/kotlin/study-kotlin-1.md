---
title: 以架构师角度认识Kotlin
---

## 目录

* Kotlin概述
* 学习Kotlin对架构师的意义
* Kotlin设计理念
* Kotlin构建流程
* 该如何有效的学习
* 作业：Kotlin与Java的异同

## Kotlin概述

**Kotlin是一种在Java虚拟机上运行的静态类型编程语言**。它主要是JetBrains开发团队所开发出来的编程语言。虽然**Kotlin与Java语法并不兼容**，但**Kotlin被设计成可以和Java代码相互运作**，并可以重复使用如Java集合框架等现有Java引用的方法库。它很容易**在Android项目中替代Java或者同Java一起使用**。

Google在2019年的Google I/O大会上宣布Kotlin被选为Android开发首选语言。

### Kotlin的特点很多：

* 简洁易用: Kotlin中提供了大量的扩展，使得我们的代码更加简洁，开发出来的框架更加易用
* 安全: 避免空指针异常等整个类的错误
* 互操作性: 充分利用 JVM、Android 和浏览器的现有库
* 工具友好: 可用任何 Java IDE 或者使用命令行构建



## 学习Kotlin对架构师的意义Kotlin

那现在该不该学习Kotlin呢？以及学习Kotlin对架构师来说会有那些意义呢？

- 学习正当时：目前Kotlin已经成为Android开发的官方首选语言，现在学习Kotlin是正当时；
- 顺应潮流，为了未来：现在包括一二线在内的各大互联网公司都在往Kotlin上转，现在学习Kotlin不仅是顺应潮流，更是为了为了未来着想；
- 提升开发效率：在开发效率上Kotlin开发要比Java高很多；
- 带领团队进行技术革新：提升团队的对前沿技术的追求与提升团队技术氛围，满足小伙伴对（钱、成长、平台）中成长的追求；

## Kotlin设计理念

<img src="/imgs/kotlin/kotlin-design-concept.jpg" />

上图是来在于Kotlin语言设计团队的老大Andrey Breslav在2018年KotlinConf的会议上的演讲，里面包含了他对Kotlin设计上的一些理念的介绍，感兴趣的同学可以在油管上搜一下这个视频的完整版。

- 视频：[KotlinConf 2018 - Conference Opening Keynote by Andrey Breslav](https://www.youtube.com/watch?v=PsaFVLr8t4E)

在KotlinConf 2018大会上Andrey Breslav表示了Kotlin并没有独创一些当前没有或大众不太熟悉的设计理念，而是吸收了众多其他语言的精髓，并且提供强大的IDE支持，能真正方便开发者运用到实际项目之中。

- 不仅简洁而且可读性强：主要指的是Kotlin支持隐藏例如getter、setter等Java模板代码，并且有大量的标准库以及灵活的重载和扩展机制，来使代码变得更加直观；

<img src="/imgs/kotlin/java-vs-kotlin.png" />
上图是在设置控件点击事件时分别用Java和Kotlin实现的对比，可以看出Kotlin实现省去了一些模板代码，不仅看起来简洁而且能让开发者专注于具体逻辑的实现，可读性更强。

- 不仅富有表现力而且可复用性更强：Kotlin将一些常用的功能比如：Android的一些扩展等封装成库，以方便开发者使用，在复用性方面它比Java有过之而不及；

- 对互通性的追求要大于独创性：为什么这么说呢，在Kotlin发展的初期，很多人会觉得Kotlin无非就是从其它语言copy过来的，之说以这么觉得是因为你看像高阶函数，闭包，扩展，Lambda表达式等都不是Kotlin首创的，都是从现有的语言中
  借鉴过来的；Kotlin的设计者的理念是不追求独创性，而是追求更好的互通性，Kotlin要想超越Java必须要实现和Java的互通（互操作性），而不是另起炉灶。

- 强大的工具支撑以NPE保护使健全性更强：大家都知道Kotlin是JetBrains的团队开发的，而JetBrains也是大名鼎鼎的IntelliJ IDEA的作者，而我们开发Android 所以的AS也是Google和而JetBrains合作的产物，可以说在工具方面AS 为Kotlin开发Android提供了包括编译器检查等强大的支持。


## Kotlin构建流程

<img src="/imgs/kotlin/kotlin-build.png" />

上图是Kotlin和Java的构建流程对比：

- *kt会被Kotlin编译器编程编译成.class的字节码文件，然后被归档成.jar，最后呢由各平台打包工具输出最终的原因程序
- 上图不难理解*kt最终会被编译成Java的字节码文件，那为什么在最后一步还需要一个Kotlin运行时呢？想必很多小伙伴对这块肯定会有所疑惑
  - 这是因为，我们用Java来写的程序所有的实现都会有标准的Java类库来做支撑，比如：`java.lang.*`, `java.util.*`，但Kotlin中的类库是不在标准的Java类库中的，所以，Kotlin应用程序需要在最后一步借助Kotlin运行时来支撑这些Java标准类库没有的实现。

## 该如何有效的学习

- 首先过一遍上述的`Kotlin与Java的异同`对`Kotlin`中差异的部分有个印象
- 跟着本课程的Kotlin讲解学习一篇
- 善用工具（`君子生非异也，善假于物也`）
  - 官方文档：[https://kotlinlang.org/docs/reference/](https://kotlinlang.org/docs/reference/)
  - 终极工具：善用AS的 `convert java file to kotlin file` 工具
  - 懂你的Google

### 终极工具（Practice）

如果你遇到一些代码不知道该如何用Kotlin来实现的时候，不妨试试下面的工具：

```java
public interface BrightnessListener {
    String[] onViewBrightness(String... args);
}
```

通过如下步骤将上述Java文件转成Kotlin：

- `AS` -> `Code` -> `convert java file to kotlin file`

或者将上述代码复制到kotlin文件中利用AS的自动转换功能进行转换：

```kotlin
interface BrightnessListener {
    fun onViewBrightness(vararg args: String?): Array<String?>?
}
```

> Kotlin可以理解为Java的语法糖，我们可以借助AS提供的工具来查看我们Kotlin代码的Java 模样，也就是说IDE 会将我们的Kotlin代码转成怎样的Java代码：


- `AS` -> `Tools` -> `Kotlin` -> `Show kotln Bytecode`



## 作业：Kotlin与Java的异同


### 打印日志

 > Java

```java
System.out.print("hello world");
System.out.println("hello world");
```

>  Kotlin

```kotlin
print("hello world")
println("hello world")
```

### 定义变量与常量常

> Java

```java
String name = "hello world";
final String name = "hello world";
```

> Kotlin

```kotlin
var name = "hello world"
val name = "hello world"
```


### null声明

> Java

```java
String otherName;
otherName = null;
```

> Kotlin

```kotlin
var otherName : String?
otherName = null
```


### 空判断

>  Java

```java
if (text != null) {
    int length = text.length();
}
```

>  Kotlin

```kotlin
text?.let {
    val length = text.length
}
// or simply
val length = text?.length
```


### 字符串拼接

>  Java

```java
String firstName = "Android";
String lastName = "Architect";
String message = "My name is: " + firstName + " " + lastName;
```

>  Kotlin

```kotlin
val firstName = "Android"
val lastName = "Architect"
val message = "My name is: $firstName $lastName"
```


### 换行

> Java

```java
String text = "First Line\n" +
              "Second Line\n" +
              "Third Line";
```

> Kotlin

```kotlin
val text = """
        |First Line
        |Second Line
        |Third Line
        """.trimMargin()
```



### 三元表达式

> Java

```java
String text = x > 5 ? "x > 5" : "x <= 5";
```

> Kotlin

```kotlin
val text = if (x > 5)
              "x > 5"
           else "x <= 5"
```


### 操作符

> java

```java
final int andResult  = a & b;
final int orResult   = a | b;
final int xorResult  = a ^ b;
final int rightShift = a >> 2;
final int leftShift  = a << 2;
final int unsignedRightShift = a >>> 2;
```

> Kotlin

```kotlin
val andResult  = a and b
val orResult   = a or b
val xorResult  = a xor b
val rightShift = a shr 2
val leftShift  = a shl 2
val unsignedRightShift = a ushr 2
```


### 类型判断和转换 (声明式)

> Java

```java
Car car = (Car) object;
```

> Kotlin

```kotlin
var car = object as Car
```


### 类型判断和转换 (隐式)

> Java

```java
if (object instanceof Car) {
   Car car = (Car) object;
}
```

> Kotlin

```kotlin
if (object is Car) {
   var car = object // 自动识别
}
```


### 多重条件

> Java

```java
if (score >= 0 && score <= 300) { }
```

> Kotlin

```kotlin
if (score in 0..300) { }
```


### 更灵活的case语句

> Java

```java
int score = // some score;
String grade;
switch (score) {
    case 10:
    case 9:
        grade = "Excellent";
        break;
    case 8:
    case 7:
    case 6:
        grade = "Good";
        break;
    case 5:
    case 4:
        grade = "OK";
        break;
    case 3:
    case 2:
    case 1:
        grade = "Fail";
        break;
    default:
        grade = "Fail";
}
```

> Kotlin

```kotlin
var score = // some score
var grade = when (score) {
    9, 10 -> "Excellent"
    in 6..8 -> "Good"
    4, 5 -> "OK"
    in 1..3 -> "Fail"
    else -> "Fail"
}
```


### for循环

> Java

```java
for (int i = 1; i <= 10 ; i++) { }

for (int i = 1; i < 10 ; i++) { }

for (int i = 10; i >= 0 ; i--) { }

for (int i = 1; i <= 10 ; i+=2) { }

for (int i = 10; i >= 0 ; i-=2) { }

for (String item : collection) { }

for (Map.Entry<String, String> entry: map.entrySet()) { }
```

> Kotlin

```kotlin
for (i in 1..10) { }

for (i in 1 until 10) { }

for (i in 10 downTo 0) { }

for (i in 1..10 step 2) { }

for (i in 10 downTo 0 step 2) { }

for (item in collection) { }

for ((key, value) in map) { }
```


### 更方便的集合操作

> Java

```java
final List<Integer> listOfNumber = Arrays.asList(1, 2, 3, 4);

final Map<Integer, String> keyValue = new HashMap<Integer, String>();
map.put(1, "Android");
map.put(2, "Ali");
map.put(3, "Mindorks");

// Java 9
final List<Integer> listOfNumber = List.of(1, 2, 3, 4);

final Map<Integer, String> keyValue = Map.of(1, "Android",
                                             2, "Ali",
                                             3, "Mindorks");
```

> Kotlin

```kotlin
val listOfNumber = listOf(1, 2, 3, 4)
val keyValue = mapOf(1 to "Android",
                     2 to "Ali",
                     3 to "Mindorks")
```


### 遍历

> Java

```java
// Java 7 and below
for (Car car : cars) {
  System.out.println(car.speed);
}

// Java 8+
cars.forEach(car -> System.out.println(car.speed));

// Java 7 and below
for (Car car : cars) {
  if (car.speed > 100) {
    System.out.println(car.speed);
  }
}

// Java 8+
cars.stream().filter(car -> car.speed > 100).forEach(car -> System.out.println(car.speed));
```

> Kotlin

```kotlin
cars.forEach {
    println(it.speed)
}

cars.filter { it.speed > 100 }
      .forEach { println(it.speed)}
```


###  方法定义

> Java

```java
void doSomething() {
   // logic here
}

void doSomething(int... numbers) {
   // logic here
}
```

> Kotlin

```kotlin
fun doSomething() {
   // logic here
}

fun doSomething(vararg numbers: Int) {
   // logic here
}
```


### 带返回值的方法

> Java

```java
int getScore() {
   // logic here
   return score;
}
```

> Kotlin

```kotlin
fun getScore(): Int {
   // logic here
   return score
}

// as a single-expression function

fun getScore(): Int = score
```



### 无结束符号

> Java

```java
int getScore(int value) {
    // logic here
    return 2 * value;
}
```

> Kotlin

```kotlin
fun getScore(value: Int): Int {
   // logic here
   return 2 * value
}

// as a single-expression function

fun getScore(value: Int): Int = 2 * value
```


### constructor 构造器

> Java

```java
public class Utils {

    private Utils() {
      // This utility class is not publicly instantiable
    }

    public static int getScore(int value) {
        return 2 * value;
    }

}
```

> Kotlin

```kotlin
class Utils private constructor() {

    companion object {

        fun getScore(value: Int): Int {
            return 2 * value
        }

    }
}

// another way

object Utils {

    fun getScore(value: Int): Int {
        return 2 * value
    }

}
```


### Get Set 构造器

> Java

```java
public class Developer {

    private String name;
    private int age;

    public Developer(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }
}
```

>  Kotlin

```kotlin
data class Developer(val name: String, val age: Int)

```


## 参考

[from-java-to-kotlin](https://github.com/MindorksOpenSource/from-java-to-kotlin)







