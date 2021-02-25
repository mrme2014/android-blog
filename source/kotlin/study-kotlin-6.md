---
title: 深入理解Kotlin类与接口
---
<!--more-->

## 目录

- 构造方法
- 继承与覆盖
- 属性
- 抽象类与接口
- 数据类
- 对象表达式与对象声明
- 作业：Kotlin实现静态成员的几种方案

## 构造方法

在 Kotlin 中的一个类可以有一个主构造方法以多个次构造方法，接下来我们先看什么是柱构造方法？

### 主构造方法

主构造方法是类头的一部分：它跟在类名（与可选的类型参数）后。

```kotlin
class KotlinClass constructor(name: String) { /*……*/ }
```

>如果主构造方法没有任何注解或者可见性修饰符，可以省略这个 constructor 关键字

```kotlin
class KotlinClass(name: String) { /*……*/ }
```

>主构造方法不能包含任何的代码，初始化的代码可以放到以 init 关键字作为前缀的初始化块（initializer blocks）中。

#### 初始化顺序

在实例初始化期间，初始化块按照它们出现在类体中的顺序执行，与属性初始化器交织在一起：

```kotlin
class InitDemo(name: String) {
    /**
     * 关于：.also(::println)的理解
     * ::表示创建成员引用或类引用，通过它来传递println方法来作为also的参数
     * 那also又是什么呢？源码跟进 -> Kotlin扩展
     */
    val firstProp = "First Prop: $name".also(::println)

    init {
        println("First initializer block that prints $name")
    }

    val secondProp = "Second Prop: ${name.length}".also(::println)

    init {
        println("Second initializer block that prints ${name.length}")
    }
}
```


>主构造的参数除了可以在初始化块中使用外，还可以在类体内声明属性时使用。


声明属性以及从主构造方法初始化属性的更简洁的写法：

```kotlin
//构造方法的参数作为类的属性并赋值，KotlinClass2在初始化时它的name与score属性会被赋值
class KotlinClass2(val name: String, var score: Int) { /*……*/ }
```

与普通属性一样，主构造方法中声明的属性可以是可变的（var）或只读的（val）。

### 次构造方法

我们也可以在类体内通过constructor前缀来声明类的次构造方法：

```kotlin
class KotlinClass3 {
    var views: MutableList<View> = mutableListOf()

    constructor(view: View) {
        views.add(view)
    }
}
```

如果类有一个主构造方法，每个次构造方法都需要委托给主构造方法，可以直接委托或者通过别的次构造方法间接委托。委托到另一个构造方法用 this 关键字即可：

```kotlin
class KotlinClass3(val view: View) {
    var views: MutableList<View> = mutableListOf()

    constructor(view: View, index: Int) : this(view) {
        views.add(view)
    }
}
```

因为初始化块中的代码实际上是主构造方法的一部分，所以初始化代码块会在次构造方法之前执行。

## 继承与覆盖

在Kotlin中，所有的类默认都是final的，如果你需要允许它可以被继承，那么你需要使用open声明：

```kotlin
open class Animal(age: Int) {
    init {
        println(age)
    }
}
```

>在 Kotlin 中所有类都有一个共同的超类 Any，Any 有三个方法：equals()、 hashCode() 与 toString()

在Kotlin中继承用:如需继承一个类，请在类头中把超类放到冒号之后：
​

```kotlin
//派生类有柱构造方法的情况
class Dog(age: Int) : Animal(age)
```

如果派生类有一个主构造方法，其基类必须用派生类主构造方法的参数初始化。

如果派生类没有主构造方法，那么每个次构造方法必须使用 super 关键字初始化其基类型。

```kotlin
//派生类无柱构造方法的情况
class Cat : Animal {
    constructor(age: Int) : super(age)
}
```


### 覆盖规则

#### 覆盖方法

Kotlin的类成员默认是隐藏的，也就是无法被覆盖，如果要覆盖我们需要用到显式修饰符（open）：

```kotlin
open class Animal(age: Int) {
    init {
        println(age)
    }

    open fun eat() {

    }
}

/**
 * 继承
 */
//派生类有主构造方法的情况
class Dog(age: Int) : Animal(age) {
    override fun eat() {

    }
}
```

eat() 方法上必须加上 override 修饰符。如果没写，编译器将会报错。 如果方法没有标注 open 如 eat()，那么子类中不允许定义相同签名的方法， 不论加不加 override。

#### 覆盖属性

属性覆盖与方法覆盖类似；在超类中声明然后在派生类中重新声明的属性必须以 override 开头，并且它们必须具有兼容的类型。

```kotlin
open class Animal(age: Int) {
    open val foot: Int = 0
}

class Dog(age: Int) : Animal(age) {
    override val foot = 4
}
```


## 属性

### 属性的声明

Kotlin 类中的属性既可以用关键字 var 声明为可变的，也可以用关键字 val 声明为只读的。

```kotlin
class Shop {
    var name: String = "Android"
    var address: String? = null
}

fun copyShop(shop: Shop): Shop {
    val shop = Shop()
    shop.name = shop.name
    shop.address = shop.address
    // ……
    return shop
}
```

### Getters 与 Setters

声明一个属性的完整语法是

```
var <propertyName>[: <PropertyType>] [= <property_initializer>]
    [<getter>]
    [<setter>]

```

其初始器（initializer）、getter 和 setter 都是可选的。如果属性类型可以从初始器 （或者从其 getter 返回值）中推断出来，也可以省略。

> 案例1：

```kotlin
val simple: Int? // 类型 Int、默认 getter、必须在构造方法中初始化
```

> 案例2：

我们可以为属性定义自定义的访问器。如果我们定义了一个自定义的 getter，那么每次访问该属性时都会调用它：

```kotlin
val isClose: Boolean
        get() = Calendar.getInstance().get(Calendar.HOUR_OF_DAY) > 11
```

如果我们定义了一个自定义的 setter，那么每次给属性赋值时都会调用它。一个自定义的 setter 如下所示：

```kotlin
var score: Float = 0.0f
    get() = if (field < 0.2f) 0.2f else field * 1.5f
    set(value) {
        println(value)
    }
```


### 延迟初始化属性

通常属性声明为非空类型必须在构造方法中初始化。 然而，这经常不方便。例如：属性可以通过依赖注入来初始化， 或者在单元测试的 setup 方法中初始化。


为处理这种情况，你可以用 lateinit 修饰符标记该属性：

```kotlin
class Test {
    lateinit var shop: Shop
    fun setup() {
        shop = Shop()
    }
}
```

在初始化前访问一个 lateinit 属性会抛出一个特定异常。

我们可以通过属性的 .isInitialized API来检测一个 lateinit var 属性是否已经初始化过：

```kotlin
if (::shop.isInitialized)
            println(shop.address)
```


## 抽象类与接口

### 抽象类

类以及其中的某些成员可以声明为 abstract:

```kotlin
abstract class Printer {
    abstract fun print()
}

class FilePrinter : Printer() {
    override fun print() {
    }
}
```

### 接口

Kotlin 的接口可以既包含抽象方法的声明也包含实现。与抽象类不同的是，接口无法保存状态，它可以有属性但必须声明为抽象或提供访问器实现。

>可以将Kotlin的接口理解为特殊的抽象类


### 接口的定义与实现

```kotlin
interface Study {
    var time: Int// 抽象的
    fun discuss()
    fun earningCourses() {
        println("Android 架构师")
    }
}

//在主构造方法中覆盖接口的字段
class StudyAS(override var time: Int) : Study {
    //在类体中覆盖接口的字段
    //    override var time: Int = 0
    override fun discuss() {

    }
}
```


### 解决覆盖冲突

实现多个接口时，可能会遇到同一方法继承多个实现的问题：

```kotlin
interface A {
    fun foo() {
        println("A")
    }
}

interface B {
    fun foo() {
        print("B")
    }
}

class D : A, B {
    override fun foo() {
        super<A>.foo()
        super<B>.foo()
    }
}
```

在上例中我们通过super<>.来解决覆盖冲突的问题。

## 数据类


我们经常创建一些只保存数据的类，在 Kotlin 中，我们可以通过data来声明一个数据类：

```kotlin
data class Address(val name: String, val number: Int) {
    var city: String = ""
    fun print() {
        println(city)
    }
}
```


### 数据类的要求：

- 主构造方法需要至少有一个参数；
- 主构造方法的所有参数需要标记为 val 或 var；
- 数据类不能是抽象、开放、密封或者内部的；

### 数据类与解构声明

为数据类生成的 Component 方法 使它们可在解构声明中使用：

```kotlin
val address = Address("Android", 1000)
address.city = "Beijing"
val (name, city) = address
println("name:$name city:$city")
```

## 对象表达式与对象声明

在Kotlin中提供了对象表达式来方面我们在需要对一个类做轻微改动并创建它的对象，而不用为之显式声明新的子类。

### 对象表达式

要创建一个继承自某个（或某些）类型的匿名类的对象，我们会这么写：

```kotlin
open class Address2(name: String) {
    open fun print() {

    }
}

class Shop2 {
    var address: Address2? = null
    fun addAddress(address: Address2) {
        this.address = address
    }

}

fun test3() {
    //如果超类型有一个构造方法，则必须传递适当的构造方法参数给它
    Shop2().addAddress(object : Address2("Android") {
        override fun print() {
            super.print()
        }
    })
}
```


如果我们只需要“一个对象而已”，并不需要特殊超类型，那么我们可以简单地写：

```kotlin
fun foo() {
    val adHoc = object {
        var x: Int = 0
        var y: Int = 0
    }
    print(adHoc.x + adHoc.y)
}
```

>请注意，匿名对象可以用作只在本地和私有作用域中声明的类型。

如果你使用匿名对象作为公有方法的返回类型或者用作公有属性的类型，那么该方法或属性的实际类型会是匿名对象声明的超类型，
如果没有声明任何超类型，就会是 Any，在匿名对象中添加的成员将无法访问。

```kotlin
class Shop2 {
    var address: Address2? = null
    fun addAddress(address: Address2) {
        this.address = address
    }

    // 私有方法，所以其返回类型是匿名对象类型
    private fun foo() = object {
        val x: String = "x"
    }

    // 公有方法，所以其返回类型是 Any
    fun publicFoo() = object {
        val x: String = "x"
    }

    fun bar() {
        val x1 = foo().x        // 没问题
//        val x2 = publicFoo().x  // 错误：未能解析的引用“x”
    }
}
```


### 对象声明


将类的`class`修饰符改为`object`就变成了对象声明：

```kotlin
object DataUtil {
    fun <T> isEmpty(list: ArrayList<T>?): Boolean {
        return list?.isEmpty() ?: false
    }
}

fun testDataUtil() {
    val list = arrayListOf("1")
    println(DataUtil.isEmpty(list))
}
```

这称为对象声明，并且它总是在 object 关键字后跟一个名称。 就像变量声明一样，对象声明不是一个表达式，不能用在赋值语句的右边。

>我们之前介绍过Kotlin没有静态成员，我们通常会用对象声明来实现静态方法的效果，另外对象声明也是创建单例的一个很好方式（反编译演示）


```kotlin
public final class DataUtil {
   public static final DataUtil INSTANCE;

   public final boolean isEmpty(@Nullable ArrayList list) {
      return list != null ? list.isEmpty() : false;
   }

   private DataUtil() {
   }

   static {
      DataUtil var0 = new DataUtil();
      INSTANCE = var0;
   }
}
```

### 伴生对象

类内部的对象声明可以用 companion 关键字标记：

```kotlin
class Student(val name: String) {
    companion object {
        val student = Student("Android")
        fun study() {
            println("Android 架构师")
        }
    }

    var age = 16
    fun printName() {
        println("My name is $name")
    }
}

fun testStudent() {
    println(Student.student)
    Student.study()
}
```

反编译看原理：

```kotlin
public final class Student {
   private int age;
   @NotNull
   private final String name;
   @NotNull
   private static final Student student = new Student("Android");
   public static final Student.Companion Companion = new Student.Companion((DefaultConstructorMarker)null);

   public final int getAge() {
      return this.age;
   }

   public final void setAge(int var1) {
      this.age = var1;
   }

   public final void printName() {
      String var1 = "My name is " + this.name;
      boolean var2 = false;
      System.out.println(var1);
   }

   @NotNull
   public final String getName() {
      return this.name;
   }

   public Student(@NotNull String name) {
      Intrinsics.checkParameterIsNotNull(name, "name");
      super();
      this.name = name;
      this.age = 16;
   }

   ...
   public static final class Companion {
      @NotNull
      public final Student getStudent() {
         return Student.student;
      }

      public final void study() {
         String var1 = "Android 架构师";
         boolean var2 = false;
         System.out.println(var1);
      }

      private Companion() {
      }

      // $FF: synthetic method
      public Companion(DefaultConstructorMarker $constructor_marker) {
         this();
      }
   }
}
```


反编译我们看到使用伴生对象实际上是在这个类内部创建了一个名为 Companion 的静态单例内部类，伴生对象中定义的属性会直接编译为外部类的静态字段，而方法会被编译为伴生对象的方法。


>在 JVM 平台，如果使用 @JvmStatic 注解，你可以将伴生对象的成员生成为真正的静态方法和字段

最后我们总结下，对象表达式和对象声明之间的语义差异：

- 对象表达式是在使用他们的地方立即执行（及初始化）的；
- 对象声明是在第一次被访问到时延迟初始化的；
- 伴生对象的初始化是在相应的类被加载（解析）时，与 Java 静态初始化器的行为一致


## 作业

- Kotlin实现静态成员的几种方案：
  - https://zhuanlan.zhihu.com/p/26713535

## 参考

- [Kotlin——中级篇（一）：类（class）详解](https://juejin.im/post/5a3297de6fb9a045055e295e)


