---
title: 深入理解Kotlin泛型
---
<!--more-->
<script type="text/javascript">
    // 禁止右键菜单
    // true是允许，false是禁止
    document.oncontextmenu = function(){ return false; };
    // 禁止文字选择
    document.onselectstart = function(){ return false; };
    // 禁止复制
    document.oncopy = function(){ return false; };
    // 禁止剪切
    document.oncut = function(){ return false; };
    // 禁止粘贴
    document.onpaste = function(){ return false; };
    // 禁止键盘事件
    document.onkeydown = function(){ return false; };
</script>

Kotlin 的泛型与 Java 一样，都是一种语法糖，即只在源代码中有泛型定义，到了class级别就被擦除了。
泛型（Generics）其实就是把类型参数化，真正的名字叫做类型参数，它的引入给强类型编程语言加入了更强的灵活性。

在这一节为大家继续带来Kotlin中的一些高级的内容：Kotlin中的泛型。

## Why

- 架构开发的一把利器；
- 使我们的代码或开发出来的框架更加的通用；
- 增加程序的健壮性，避开运行时可能引发的 ClassCastException；
- 能够帮助你研究和理解别的框架；
- **自己造轮子需要，能用泛型解决问题**；


在 Java 中，我们常见的泛型有：泛型类、泛型接口、泛型方法和泛型属性，Kotlin 泛型系统继承了 Java 泛型系统，同时添加了一些强化的地方。

## 目录

- 泛型接口/类（泛型类型）
- 泛型字段
- 泛型方法
- 泛型约束
- 泛型中的out与in

## 泛型接口/类（泛型类型）

定义泛型类型，是在类型名之后、主构造函数之前用尖括号括起的大写字母类型参数指定：


### 泛型接口

>Java：

```java
//泛型接口
interface Drinks<T> {
    T taste();
    void price(T t);
}
```

>Kotlin：

```kotlin
//泛型接口
interface Drinks<T> {
    fun taste(): T
    fun price(t: T)
}
```


## 泛型类

>Java

```java
abstract class Color<T> {
    T t;
    abstract void printColor();
}
class Blue {
    String color = "blue";
}
class BlueColor extends Color<Blue> {
    public BlueColor(Blue1 t) {
        this.t = t;
    }
    @Override
    public void printColor() {
        System.out.println("color:" + t.color);
    }
}
```

>Kotlin

```kotlin
abstract class Color<T>(var t: T/*泛型字段*/) {
    abstract fun printColor()
}

class Blue {
    val color = "blue"
}

class BlueColor(t: Blue) : Color<Blue>(t) {
    override fun printColor() {
        println("color:${t.color}")
    }

}
```


## 泛型字段

定义泛型类型字段，可以完整地写明类型参数，如果编译器可以自动推定类型参数，也可以省略类型参数：

```kotlin
abstract class Color<T>(var t: T/*泛型字段*/) {
    abstract fun printColor()
}
```

## 泛型方法

Kotlin 泛型方法的声明与 Java 相同，类型参数要放在方法名的前面：

>Java

```java
public static <T> T fromJson(String json, Class<T> tClass) {
    T t = null;
    try {
        t = tClass.newInstance();
    } catch (Exception e) {
        e.printStackTrace();
    }
    return t;
}
```

>Kotlin

```kotlin
fun <T> fromJson(json: String, tClass: Class<T>): T? {
    /*获取T的实例*/
    val t: T? = tClass.newInstance()
    return t
}
```

## 泛型约束

Java 中可以通过有界类型参数来限制参数类型的边界，Kotlin中泛型约束也可以限制参数类型的上界：

>Java

```java
public static <T extends Comparable<? super T>> void sort(List<T> list){}
```


>Kotlin

```kotlin
fun <T : Comparable<T>?> sort(list: List<T>?){}
```

```kotlin
sort(listOf(1, 2, 3)) // OK，Int 是 Comparable<Int> 的子类型
//    sort(listOf(Blue())) // 错误：Blue 不是 Comparable<Blue> 的子类型
```

>对于多个上界的情况

```kotlin
//多个上界的情况
fun <T> test(list: List<T>, threshold: T): List<T>
        where T : CharSequence,
              T : Comparable<T> {
    return list.filter { it > threshold }.map { it }
}
```

所传递的类型T必须同时满足 where 子句的所有条件，在上述示例中，类型 T 必须既实现了 CharSequence 也实现了 Comparable。

## 泛型中的out与in

在Kotlin中out代表协变，in代表逆变，为了加深理解我们可以将Kotlin的协变看成Java的上界通配符，将逆变看成Java的下界通配符：

```kotlin
//Kotlin使用处协变
fun sumOfList(list: List<out Number>)

//Java上界通配符
void sumOfList(List<? extends Number> list)

//Kotlin使用处逆变
fun addNumbers(list: List<in Int>)

//Java下界通配符
void addNumbers(List<? super Integer> list)
```

## 总结
![Choosing empty activity](/imgs/kotlin/7-table.png)

总的来说，Kotlin 泛型更加简洁安全，但是和 Java 一样都是有类型擦除的，都属于编译时泛型。



