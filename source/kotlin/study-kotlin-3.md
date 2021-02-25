---
title: Kotlin必备基础
---

在这一节呢，我为大家准备了一些Kotlin必备基础。在学习一门语言的时候我们首先需要学习的是它有怎样的数据类型
，以及它数组、集合和方法是怎样子，因为这些是我们入手一门新的语言时的第一步，也是基础中的基础。

>那么，为了让大家快速掌握本节课的内容，我会将Kotlin和Java类比着进行讲解，这也是作为一个老开发快速学习其它语言的一个屡试不爽的经验所在。
>就像习武一样，习武的人往往会借助已有的内功，可以很快速的掌握一门新的武功，那么对于从事Android的小伙伴们来说，这么多年对Java的使用就是我们的内功
>，所以在这里我会带着大家借助已有的内功来快速上手Kotlin。

## 目录

- 认识Kotlin基本类型
- 走进Kotlin的数组
- Kotlin数组的创建技巧
- Kotlin数组的遍历技巧
- 走进Kotlin的集合
- 集合的可变性与不可变性
- 集合排序
- 走进Kotlin的set与map
- 原理探索


## Kotlin基本类型

Kotlin 的基本数值类型包括 Byte、Short、Int、Long、Float、Double 等。不同于 Java 的是，**字符不属于数值类型，是一个独立的数据类型**。

>对于整数，存在四种具有不同大小和值范围的类型
<table border="1">
  <tr bgcolor="#999999">
    <th>类型</th>
    <th>位宽</th>
    <th>最小值</th>
    <th>最大值</th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td>Byte</td>
    <td>8</td>
    <td>-128</td>
    <td>-127</td>
  </tr>
   <tr  bgcolor="#eeeeee">
    <td>Short</td>
    <td>16</td>
    <td>-32768</td>
    <td>32767</td>
  </tr>
  <tr  bgcolor="#ffffff">
    <td>Int</td>
    <td>32</td>
    <td>-2,147,483,648 (-231)</td>
    <td>2,147,483,647 (231 - 1)</td>
  </tr>
  <tr  bgcolor="#eeeeee">
    <td>Long</td>
    <td>64</td>
    <td>-9,223,372,036,854,775,808 (-263)</td>
    <td>9,223,372,036,854,775,807 (263 - 1) </td>
  </tr>
</table>
<br>

>对于浮点数，Kotlin提供了Float和Double类型
<table border="1">
  <tr bgcolor="#999999">
    <th width="310">类型</th>
    <th width="310"位宽</th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td>Float</td>
    <td>32</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>Double</td>
   <td>64</td>
  </tr>
</table>

>练一练：DataType.kt

## 走进Kotlin的数组

数组在 Kotlin 中使用 Array 类来表示，它定义了 get 与 set 方法（按照运算符重载约定这会转变为 []）以及 size 属性，以及一些其他有用的成员方法：

```kotlin
class Array<T> private constructor() {
    val size: Int
    operator fun get(index: Int): T
    operator fun set(index: Int, value: T): Unit
​
    operator fun iterator(): Iterator<T>
    // ……
}
```

## Kotlin数组的创建技巧

### 使用arrayOf()方法创建数组

我们可以使用库方法 arrayOf() 来创建一个数组并传递元素值给它，这样 arrayOf(1, 2, 3) 创建了 array [1, 2, 3]。

### 使用arrayOfNulls()方法创建数组

也可以库方法 arrayOfNulls() 可以用于创建一个指定大小的、所有元素都为空的数组。


### 动态创建数组

用接受数组大小以及一个方法参数的 Array 构造方法，用作参数的方法能够返回给定索引的每个元素初始值：

```kotlin
// 创建一个 Array<String> 初始化为 ["0", "1", "4", "9", "16"]
val asc = Array(5) { i -> (i * i).toString() }
asc.forEach { println(it) }
```

如上所述，[] 运算符代表调用成员方法 get() 与 set()。

Kotlin 中数组是不型变的（invariant）。这意味着 Kotlin 不让我们把 Array<String> 赋值给 Array<Any>，以防止可能的运行时失败。

### 原生类型数组

Kotlin 也有无装箱开销的专门的类来表示原生类型数组: IntArray、ByteArray、 ShortArray 等等。这些类与 Array 并没有继承关系，但是它们有同样的方法属性集。它们也都有相应的工厂方法:

```kotlin
//通过intArrayOf、floatArrayOf、doubleArrayOf等创建数组
val x: IntArray = intArrayOf(1, 2, 3)
println("x[1] + x[2] = ${x[1] + x[2]}")
```

```kotlin
// 大小为 5、值为 [0, 0, 0, 0, 0] 的整型数组
val arr = IntArray(5)
​
// 例如：用常量初始化数组中的值
// 大小为 5、值为 [42, 42, 42, 42, 42] 的整型数组
val arr = IntArray(5) { 42 }
​
// 例如：使用 lambda 表达式初始化数组中的值
// 大小为 5、值为 [0, 1, 2, 3, 4] 的整型数组（值初始化为其索引值）
var arr = IntArray(5) { it * 1 }
```

## Kotlin数组的遍历技巧

### 数组遍历

```kotlin
for (item in array) {
    println(item)
}
```

### 带索引遍历数组

```kotlin
for (i in array.indices) {
    println(i.toString() + "->" + array[i])
}
```

### 遍历元素(带索引)

```kotlin
for ((index, item) in array.withIndex()) {
    println("$index->$item")
}
```

### forEach遍历数组

```kotlin
array.forEach { println(it) }
```

### forEach增强版

```kotlin
array.forEachIndexed { index, item ->
    println("$index：$item")
}
```

## 走进Kotlin的集合

Kotlin 标准库提供了一整套用于管理集合的工具，集合是可变数量（可能为零）的一组条目，各种集合对于解决问题都具有重要意义，并且经常用到。


- List 是一个有序集合，可通过索引（反映元素位置的整数）访问元素。元素可以在 list 中出现多次。列表的一个示例是一句话：有一组字、这些字的顺序很重要并且字可以重复。
- Set 是唯一元素的集合。它反映了集合（set）的数学抽象：一组无重复的对象。一般来说 set 中元素的顺序并不重要。例如，字母表是字母的集合（set）。
- Map（或者字典）是一组键值对。键是唯一的，每个键都刚好映射到一个值，值可以重复。

## 集合的可变性与不可变性

那什么叫Kotlin集合的可变性与不可变性呢，在Kotlin中存在两种意义上的集合，一种是可以修改的一种是不可修改的。

### 不可变集合

```kotlin
val stringList = listOf("one", "two", "one")
println(stringList)

val stringSet = setOf("one", "two", "three")
println(stringSet)
```

### 可变集合

```kotlin
val numbers = mutableListOf(1, 2, 3, 4)
numbers.add(5)
numbers.removeAt(1)
numbers[0] = 0
println(numbers)
```

>不难发现，每个不可变集合都有对应的可变集合，也就是以mutable为前缀的集合。


## 集合排序

在Kotlin中提供了强大对的集合排序的API，让我们一起来学习一下：

```kotlin
val numbers = mutableListOf(1, 2, 3, 4)
//随机排列元素
numbers.shuffle()
println(numbers)
numbers.sort()//排序，从小打到
numbers.sortDescending()//从大到小
println(numbers)

//定义一个Person类，有name 和 age 两属性
data class Language(var name: String, var score: Int)

val languageList: MutableList<Language> = mutableListOf()
languageList.add(Language("Java", 80))
languageList.add(Language("Kotlin", 90))
languageList.add(Language("Dart", 99))
languageList.add(Language("C", 80))
//使用sortBy进行排序，适合单条件排序
languageList.sortBy { it.score }
println(languageList)
//使用sortWith进行排序，适合多条件排序
languageList.sortWith(compareBy(
        //it变量是lambda中的隐式参数
        { it.score }, { it.name })
)
println(languageList)
```

## 走进Kotlin的set与map

```kotlin
/**set**/
val hello = mutableSetOf("H", "e", "l", "l", "o")//自动过滤重复元素
hello.remove("o")
//集合的加减操作
hello += setOf("w", "o", "r", "l", "d")
println(hello)

/**Map<K, V> 不是 Collection 接口的继承者；但是它也是 Kotlin 的一种集合类型**/

val numbersMap = mapOf("key1" to 1, "key2" to 2, "key3" to 3, "key4" to 1)

println("All keys: ${numbersMap.keys}")
println("All values: ${numbersMap.values}")
if ("key2" in numbersMap) println("Value by key \"key2\": ${numbersMap["key2"]}")
if (1 in numbersMap.values) println("1 is in the map")
if (numbersMap.containsValue(1)) println(" 1 is in the map")
```

## 原理探索

- Q1：两个具有相同键值对，但顺序不同的map相等吗？为什么？
- Q2：两个具有相同元素，但单顺序不同的list相等吗？为什么？


## 参考

- [Kotlin 数据类型](https://juejin.im/post/5d837c88f265da03ab428732)

