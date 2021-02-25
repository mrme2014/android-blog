---
title: 让人爱不释手的Kotlin扩展（Extensions）技术探秘与应用
---

在这一节为大家继续带来Kotlin中的一些高级的内容：Kotlin中的Kotlin扩展（Extensions）。

## Whay

- 提供架构的易用性
- 减少代码量，让代码更加整洁、纯粹
- 提高编码的效率，生产力提高

在《以架构师角度认识Kotlin》一节我们有提到：在Kotlin中提供了大量的扩展，使得我们的代码更加简洁，开发出来的框架更加易用，那么Kotlin的扩展到底是怎样子的，以及它的实现原理如何呢，那么在这一节将为大家揭晓这些答案。

## 目录

- 扩展方法
- 扩展方法的使用
  - 在Kotlin中使用
  - 在Java中使用
- 原理解析：Kotlin扩展函数是怎么实现的
- 泛型扩展方法
- 扩展属性
- 为伴生对象添加扩展
- Kotlin中常用的扩展
- 案例：使用Kotlin扩展为控件绑定监听器减少模板代码

Kotlin 能够扩展一个类的新功能而无需继承该类。 例如，你可以为一个你不能修改的来自第三方库中的类编写一个新的函数。 这个新增的函数就像那个原始类本来就有的函数一样，可以用普通的方法调用。 这种机制称为 扩展函数 。此外，也有 扩展属性 ， 允许你为一个已经存在的类添加新的属性。想想是不是感觉很疯狂呢？那接下来就往我们开启这种疯狂吧。

## 扩展方法

Kotlin的扩展函数可以让你作为一个类成员进行调用的函数，但是是定义在这个类的外部。这样可以很方便的扩展一个已经存在的类，为它添加额外的方法。在Kotlin源码中，有大量的扩展函数来扩展Java，这样使得Kotlin比Java更方便使用，效率更高。

### 扩展方法的原型

![kotlin-extension](/imgs/kotlin/kotlin-extension.png))

### 扩展方法的使用

#### 在Kotlin中使用

```kotlin
class Jump {
    fun test() {
        println("jump test")
        //在被扩展的类中使用
        doubleJump(1f)
    }
}

fun Jump.doubleJump(howLong: Float): Boolean {
    println("jump:$howLong")
    println("jump:$howLong")
    return true
}

Jump().doubleJump(2f)
//在被扩展类的外部使用
Jump().test()
```

#### 在Java中使用

在Java中调用Kotlin扩展，需要通过扩展所在的文件名+.的方式进行调用：

```kotlin
KotlinExtensionKt.doubleJump(new Jump(), 2.0f);
```

另外，需要注意的是我们需要为这个方法传递它被扩展类的对象来作为接受者，为什么要传递接受者对象，这是由扩展的实现原理所决定的，在原理解析部分会讲解。

## 原理解析：Kotlin扩展方法是怎么实现的

>优秀架构师：不仅要知其然，也要能知其所以然

在体验到Kotlin扩展带个我们高效编程的同时，我们不禁要问自己几个问题：

- Kotlin的扩展是怎么实现的？
- Kotlin的扩展会不是有性能问题？


接下来我们就从Kotlin反编译出Java代码上来一探究竟：

```kotlin
fun main() {
    val test = mutableListOf(1, 2, 3)
    test.swap(1, 2)
    println(test)
}

fun MutableList<Int>.swap(index1: Int, index2: Int) {
    val tmp = this[index1]
    this[index1] = this[index2]
    this[index2] = tmp
}
```

>反编译出Java源码

```java
public final class KotlinExtensionKt {
   public static final void main() {
      List test = CollectionsKt.mutableListOf(new Integer[]{1, 2, 3});
      swap(test, 1, 2);
      boolean var1 = false;
      System.out.println(test);
   }

   // $FF: synthetic method
   public static void main(String[] var0) {
      main();
   }

   public static final void swap(@NotNull List $this$swap, int index1, int index2) {
      Intrinsics.checkParameterIsNotNull($this$swap, "$this$swap");
      int tmp = ((Number)$this$swap.get(index1)).intValue();
      $this$swap.set(index1, $this$swap.get(index2));
      $this$swap.set(index2, tmp);
   }
}
```

从反编译出的Java源码分析，扩展函数的实现非常简单，它没有修改接受者类型的成员，仅仅是通过静态方法来实现的。所以我们不必担心扩展函数会带来额外的性能消耗。

## 泛型扩展方法

为了考虑到扩展函数的通用型，我们可以借助上面课程中学习到的泛型，来为扩展方法进行泛型化改造，以`fun MutableList<Int>.swap(index1: Int, index2: Int)`为例，接下来我们为它进行泛型化改造：

```kotlin
//泛型化扩展函数
fun <T> MutableList<T>.swap1(index1: Int, index2: Int) {
    val tmp = this[index1]
    this[index1] = this[index2]
    this[index2] = tmp
}

val test2 = mutableListOf("Android Q", "Android N", "Android M")
test2.swap1(0,1)
println(test2)
```

## 扩展属性

扩展属性提供了一种方法能通过属性语法进行访问的API来扩展。尽管它们被叫做属性，但是它们不能拥有任何状态，它不能添加额外的字段到现有的Java对象实例。

```kotlin
//为String添加一个lastChar属性，用于获取字符串的最后一个字符
val String.lastChar: Char get() = this.get(this.length - 1)

///为List添加一个last属性用于获取列表的最后一个元素，this可以省略
val <T>List<T>.last: T get() = get(size - 1)

val listString = listOf("Android Q", "Android N", "Android M")
println("listString.last${listString.last}")
```

## 为伴生对象添加扩展

如果一个类定义了伴生对象 ，那么我们也可以为伴生对象定义扩展函数与属性：

```kotlin
class Jump {
    companion object {}
}
fun Jump.Companion.print(str: String) {
    println(str)
}

Jump.print("伴生对象的扩展")
```

就像伴生对象的常规成员一样：可以只使用类名作为限定符来调用伴生对象的扩展成员：


## Kotlin中常用的扩展


在Kotlin的源码中定义了大量的扩展，比如：`let`,`run`,`apply`，了解并运用这些函数能帮我们提高编码效率，接下来就往我们一起揭开这些扩展函数的神秘面纱吧！


### let扩展

>函数原型：

```kotlin
fun <T, R> T.let(f: (T) -> R): R = f(this)
```

let扩展函数的实际上是一个作用域函数，当你需要去定义一个变量在一个特定的作用域范围内，那么let函数是一个不错的选择；let函数另一个作用就是可以避免写一些判断null的操作。

```kotlin
fun testLet(str: String?) {
    //限制str2的作用域
    str.let {
        val str2 = "let扩展"
        println(it + str2)
    }
//    println(str2)//报错

    //避免为null的操作
    str?.let {
        println(it.length)
    }
}
```

### run扩展

>函数原型：

```kotlin
fun <T, R> T.run(f: T.() -> R): R = f()
```

run函数只接收一个lambda函数为参数，以闭包形式返回，返回值为最后一行的值或者指定的return的表达式，**在run函数中可以直接访问实例的公有属性和方法**。

```kotlin
data class Room(val address: String, val price: Float, val size: Float)

fun testRun(room: Room) {
    room.run {
        println("Room:$address,$price,$size")
    }
}
```


### apply扩展

>函数原型：

```kotlin
fun <T> T.apply(f: T.() -> Unit): T { f(); return this }
```

apply函数的作用是：调用某对象的apply函数，在函数范围内，可以任意调用该对象的任意方法，并返回该对象。

从结构上来看apply函数和run函数很像，唯一不同点就是它们各自返回的值不一样，run函数是以闭包形式返回最后一行代码的值，而apply函数的返回的是传入对象的本身。

apply一般用于一个对象实例初始化的时候，需要对对象中的属性进行赋值。或者动态inflate出一个XML的View的时候需要给View绑定数据也会用到，这种情景非常常见。

```kotlin
fun testApply() {
    ArrayList<String>().apply {
        add("testApply")
        add("testApply")
        add("testApply")
        println("$this")
    }.let { println(it) }
}
```


## 案例：使用Kotlin扩展为控件绑定监听器减少模板代码

> 定义扩展

```kotlin
//为Activity添加find扩展方法，用于通过资源id获取控件
fun <T : View> Activity.find(@IdRes id: Int): T {
    return findViewById(id)
}

//为Int添加onClick扩展方法，用于为资源id对应的控件添加onClick监听
fun Int.onClick(activity: Activity, click: () -> Unit) {
    activity.find<View>(this).apply {
        setOnClickListener {
            click()
        }
    }
}
```

> 应用扩展

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        val textView = find<TextView>(R.id.test)
        R.id.test.onClick(this) {
            textView.text = "Kotlin泛型"
        }
    }
}
```

在这个案例中我们通过两个扩展方法，大大减少了我们在获取控件，以及为控件绑定onClick监听时候的模板代码，而且代码可读性更高，更加直观，这便是Kotlin扩展的强大之处。

Kotlin扩展的应用案例远不止这些，需要大家在下去之后能够活学活用，来发掘属于你自己的Kotlin扩展吧。

* [辅助资料](https://cloud.tencent.com/developer/article/1146533)


