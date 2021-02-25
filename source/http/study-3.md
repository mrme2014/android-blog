---
title: 见微知著从源码到原理剖析retrofit核心知识点
---

## 目录

- Retrofit概述
- 实现原理源码剖析
- 设计模式



### Retrofit概述

- Retrofit最初的样子

```kotlin
val retrofit =  Retrofit.Builder()
    .baseUrl("https://api.devio.org/as/")
    .build();

interface Api {
  //默认情况下方法返回类型为Call<T>,response类型为ResponseBody
    @GET("/user/login") 
    fun login(): Call<ResponseBody>
}
```

- Retrofit扩展玩法

```kotlin
val retrofit=Retrofit.Builder()
            .baseUrl("https://api.devio.org/as/")
            .callFactory(OkHttpClient())
            .addConverterFactory(GsonConvertFactory.create())
            .addCallAdapterFactory(RxJavaCallAdapterFactory.create())  
            .addCallAdapterFactory(HiCallAdapterFactory.create())
            .build()

interface Api {
    @GET("/user/login") //添加GsonConvertFactory ,response类型可以写成User
    fun login2(): Call<User>

    @GET("/user/login") //添加RxJavaCallFactory, 方法返回类型可以写成Observer
    fun login3(): Observable<User>
   
    @GET("/user/login") //retrofit +  coroutine
    suspend fun login4(): Response<User>
  
    @GET("/user/login")
    suspend fun login5(): User
}
```

- RxJavaCallAdapterFactory

```java
class  RxJava2CallAdapter<R>:CallAdapter<R, Observable>{
   override fun adapt(Call<R> call):Observable {
    //经过适配转换，那么就可以利用RxJava的链式调用能力
    val observable =CallExecuteObservable<>(call);
    return observable
  }
}
```

- HiCallAdapterFactory

```java
class  HiCallAdapter<R>:CallAdapter<R, HiCall> {
  override fun adapt(call:Call<R>):HiCall {
    //经过适配转换，就能实现网路请求结束自动切到主线程
    val hiCall = HiCall<>(call);
    return hiCall
  }
}
```

- HiCall

```kotlin
class HiCall:Call<T>{
    private var delegate:Call<T>
    fun HiCall(delegate:Call<T>){
      this.delegate= delegate
    }
    fun enqueue(callback: Callback<T>){
      delegate.enqueue{response->
          mainHandler.post{
            callback.onSuccess(response)
         }
      }
    }
}
```





### 设计模式

#### (1)Builder模式:

- **生成Retrofit对象和ServiceMethod对象时候都用到了Builder模式**

#### (2)工厂模式:

- **Retrofit的ConverterFactory和AdapterFactory,可以用来生产response转换器和 Call接口转换器**

#### (3)代理模式:

- **可以在原方法执行之前和之后做一些操作（Log,做事务控制等），也可以用来实现延迟加载,切线程**

#### (4)适配器模式:

-  **适配器模式用来将接口A转化成接口B，在Retrofit中用来将Call异步接口转化成其他的异步接口**

#### (5)注解 Annotation：

- **许多目前热门的Android框架都广泛使用了Annotation来对调用者暴露接口，如arouter、databinding、room**

 

### 总结

Retrofit的设计符合了高内聚，低耦合的原则，有效的将其他框架组织起来，并使其之间解耦，这增强了Retrofit的易用性和灵活性。Retrofit合理运用多种设计模式以及其面向接口的编程方式是其达到高内聚低耦合的关键。没有重新造轮子，而是复用其他轮子，让轮子们高效组合到一起也是Retrofit的意义。
