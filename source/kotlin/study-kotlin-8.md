---
title: 深入理解Kotlin注解
---

在这一节为大家继续带来Kotlin中的一些高级的内容：Kotlin中的注解。

## Why

- 架构开发的一把利器；
- 使逻辑实现更加简洁，让代码更加清晰易懂；
- 能够帮助你研究和理解别的框架；
- **自己造轮子需要，能用注解解决问题**；

比如：有了注解我们可以很好地实现调用前权限检测，登录拦截，AOP等功能


提到注解大家可能都不陌生，那Kotlin的注解是怎样子的，它又和我们所熟悉的Java注解又有何异同呢？


## 目录

- Kotlin的注解是怎样子的？
- 如何声明Kotlin注解？
- 如何使用Kotlin注解？
- Kotlin中的元注解
- 注解的使用场景
- 案例：自定义注解实现API调用时的请求方法检查


## Kotlin的注解是怎样子的？

注解就是为了给代码提供元数据，并且注解是不直接影响代码的执行，**在Kotlin中注解核心概念和Java一样，并且100%与Java注解兼容**。
一个注解允许你把额外的元数据关联到一个声明上，然后元数据就可以被某种方式(比如运行时反射方式以及一些源代码工具)访问。

>注解实际上就是一种代码标签，它作用的对象是代码。它可以给特定的注解代码标注一些额外的信息。然而这些信息可以选择不同保留时期，比如源码期、编译期、运行期。然后在不同时期，可以通过某种方式获取标签的信息来处理实际的代码逻辑，这种方式常常就是我们所说的反射。


## 如何声明Kotlin注解？

在Kotlin中的声明注解的方式和Java稍微不一样，在Java中主要是通过 @interface关键字来声明，而在Kotlin中只需要通过 annotation class 来声明。

>Java注解的声明

```kotlin
//Java中的注解通过@interface关键字进行定义，它和接口声明类似，只不过在前面多加@
@interface ApiDoc {
    String value();
}
```

>Kotlin注解的声明

```kotlin
//和一般的声明很类似，只是在class前面加上了annotation修饰符
annotation class ApiDoc(val value: String)
```

## 如何使用Kotlin注解？

在Kotlin中使用注解和Java一样：

```kotlin
@ApiDoc("修饰类")
class Box {
    @ApiDoc("修饰字段")
    val size = 6

    @ApiDoc("修饰方法")
    fun test() {

    }
}
```

## Kotlin中的元注解

和Java一样在Kotlin中一个Kotlin注解类自己本身也可以被注解，可以给注解类加注解，我们把这种注解称为元注解。

Kotlin中的元注解类定义于kotlin.annotation包中，主要有：

- @Target：定义注解能够应用于那些目标对象
- @Retention：注解的保留期
- @Repeatable：标记的注解可以多次应用于相同的声明或类型
- @MustBeDocumented：修饰的注解将被文档工具提取到API文档中

4种元注解，相比Java中5种元注解少了@Inherited，在这里四种元注解中最常用的是前两种，接下来我们就来重点分析下前两种元注解：

### @Target

@Target顾名思义就是目标对象，也就是我们定义的注解能够应用于那些目标对象，可以同时指定多个作用的目标对象。

#### @Target的原型

```kotlin
@Target(AnnotationTarget.ANNOTATION_CLASS)//可以给标签自己贴标签
@MustBeDocumented
public annotation class Target(vararg val allowedTargets: AnnotationTarget)
```

从@Target的原型中我们可以看出，它接受一个vararg可变数量的参数，所以可以同时指定多个作用的目标对象，并且参数类型限定为AnnotationTarget。

在@Target注解中可以同时指定一个或多个目标对象，那么到底有哪些目标对象呢？接下来让我们一起走进AnnotationTarget枚举类的源码：

##### AnnotationTarget

```kotlin
public enum class AnnotationTarget {
    CLASS, //表示作用对象有类、接口、object对象表达式、注解类
    ANNOTATION_CLASS,//表示作用对象只有注解类
    TYPE_PARAMETER,//表示作用对象是泛型类型参数(暂时还不支持)
    PROPERTY,//表示作用对象是属性
    FIELD,//表示作用对象是字段，包括属性的幕后字段
    LOCAL_VARIABLE,//表示作用对象是局部变量
    VALUE_PARAMETER,//表示作用对象是函数或构造函数的参数
    CONSTRUCTOR,//表示作用对象是构造函数，主构造函数或次构造函数
    FUNCTION,//表示作用对象是函数，不包括构造函数
    PROPERTY_GETTER,//表示作用对象是属性的getter函数
    PROPERTY_SETTER,//表示作用对象是属性的setter函数
    TYPE,//表示作用对象是一个类型，比如类、接口、枚举
    EXPRESSION,//表示作用对象是一个表达式
    FILE,//表示作用对象是一个File
    @SinceKotlin("1.1")
    TYPEALIAS//表示作用对象是一个类型别名
}
```

一旦注解被限定了@Target那么它只能被应用于限定的目标对象上，为了验证这一说法，我们为`ApiDoc`限定下目标对象：

```kotlin
@Target(AnnotationTarget.CLASS)
annotation class ApiDoc(val value: String)

@ApiDoc("修饰类")
class Box {
    @ApiDoc("修饰字段")
    val size = 6

    @ApiDoc("修饰方法")
    fun test() {

    }
}
```

这样一来ApiDoc注解只能被应用于类上，如果将它应用在方法或字段上则会抛出异常：

```bash
This annotation is not applicable to target 'member property with backing field'
```

### @Retention

@Retention 我们可以理解为保留期，和Java一样Kotlin有三种时期: 源代码时期(SOURCE)、编译时期(BINARY)、运行时期(RUNTIME)。

#### @Retention原型

```kotlin
@Target(AnnotationTarget.ANNOTATION_CLASS)//目标对象是注解类
public annotation class Retention(val value: AnnotationRetention = AnnotationRetention.RUNTIME)
```

Retention接收一个AnnotationRetention类型的参数，该参数有个默认值，默认是保留在运行时期。

#### AnnotationRetention

```kotlin
@Retention元注解取值主要来源于AnnotationRetention枚举类
public enum class AnnotationRetention {
    SOURCE,//源代码时期(SOURCE): 注解不会存储在输出class字节码中
    BINARY,//编译时期(BINARY): 注解会存储出class字节码中，但是对反射不可见
    RUNTIME//运行时期(RUNTIME): 注解会存储出class字节码中，也会对反射可见, 默认是RUNTIME
}
```

## 注解的使用场景

- 提供信息给编译器：编译器可以利用注解来处理一些，比如一些警告信息，错误等
- 编译阶段时处理：利用注解信息来生成一些代码，在Kotlin生成代码非常常见，一些内置的注解为了与Java API的互操作性，往往借助注解在编译阶段生成一些额外的代码
- 运行时处理：某些注解可以在程序运行时，通过反射机制获取注解信息来处理一些程序逻辑

## 案例：自定义注解实现API调用时的请求方法检查


```kotlin
public enum class Method {
    GET,
    POST
}

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
annotation class HttpMethod(val method: Method)

interface Api {
    val name: String
    val version: String
        get() = "1.0"
}

@HttpMethod(Method.GET)
class ApiGetArticles : Api {
    override val name: String
        get() = "/api.articles"
}

fun fire(api: Api) {
    val annotations = api.javaClass.annotations
    val method = annotations.find { it is HttpMethod } as? HttpMethod
    println("通过注解得知该接口需要通过：${method?.method} 方式请求")
}
```


- https://juejin.im/post/5cb7ebeee51d456e8833394b#heading-9



