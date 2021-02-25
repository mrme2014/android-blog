---
title: 探秘Kotlin协程机制
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



## 目录

- 什么是协程

- 协程的用法

- 协程的启动

- 协程挂起,恢复原理逆向剖析

  



### 什么是协程
<img src="/imgs/thread/coroutine_1.png" style="zoom:50%" width="1500"/>

```kotlin
//客户端顺序进行三次网络异步请求，并用最终结果更新UI
request1(parameter) { value1 ->
	request2(value1) { value2 ->
		request3(value2) { value3 ->
			updateUI(value3)            
		} 
	}              
}
```

这种结构的代码无论是阅读起来还是维护起来都是极其糟糕的。**对多个回调组成的嵌套耦合，我亲切地称为 "回调地狱"。**

- 协程的写法

```kotlin
GlobalScope.launch(Dispatchers.Main){
  val value1 = request1()
  val value2 = request2(value1)
  val value3 = request2(value2）
  updateUI(value3)
}

suspend request1( )
suspend request2(..)
suspend request3(..)
```

- 场景2：并发流程控制
<img src="/imgs/thread/coroutine2.png" style="zoom:50%" width="1500"/>

```kotlin
//客户端顺序并发三次网络异步请求，并用最终结果更新UI
fun request1(parameter) { value1 ->
	request2(value1) { value2 ->
      this.value2=value2   
	    if(request3){
         updateUI()       
      }
	} 
  request3(value2) { value3 ->
      this.value3=value3                
	    if(request2) {
        updateUI()
      }     
	}                                  
}

fun updateUI() 
```

- 协程写法

```kotlin
GlobalScope.launch(Dispatchers.Main){
   val value1 =    request1()
   val deferred2 = GlobalScope.async{request2(value1)}
   val deferred3 = GlobalScope.async{request3(value2)}
   updateUI(deferred2.await(),deferred3.await())
}

suspend request1( )
suspend request2(..)
suspend request3(..)
```

**协程的目的是为了让多个任务之间更好的协作,解决异步回调嵌套。能够以同步的方式编排代码完成异步工作。将异步代码像同步代码一样直观。同时它也是一个并发流程控制的解决方案。**

协程主要是**让原来要使用“异步+回调”写出来的复杂代码, 简化成看似同步写出来的方式**,弱化了线程的概念（对线程的操作进一步抽象）



### 协程的用法

- 引入gradle依赖

```java
//在kotlin项目中配合jetpack架构引入协程
api 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.2.0'
api 'androidx.lifecycle:lifecycle-runtime-ktx:2.2.0'
api 'androidx.lifecycle:lifecycle-livedata-ktx:2.2.0'
  
//在kotlin项目但非jetpack 架构项目中引入协程
api "org.jetbrains.kotlinx:kotlinx-coroutines-core:1.2.1"
api 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.1.1'
```

- 常用的创建协程的方法

```kotlin
//创建协程时,可以通过Dispatchers.IO,MAIN,Unconfined指定协程运行的线程
val job:Job =GlobalScope.launch(Dispatchers.Main)
val deffered:Deffered=GlobalScope.async（Dispatchers.IO）
```

- Job：协程构建函数的返回值，可以把 Job 看成协程对象本身，包含了对协程的控制方法。
Deffered是Job的子类，实际上就增加了个await方法。能够让当前协程暂时挂起,暂停往下执行。当await方法有返回值后,会恢复协程,继续往下执行
<table border="1">
  <tr bgcolor="#999999">
    <th width="385">方法</th>
    <th width="475">说明</th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td>start()        </td>
    <td>手动启动协程</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>join()          </td>
   <td>等待协程执行完毕</td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>cancel()        </td>
   <td>取消一个协程</td>
  </tr>
</table>
<br></br>
- 协程的启动
```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job{
  val newContext = newCoroutineContext(context)
  val coroutine =  StandaloneCoroutine(newContext, active = true)
  coroutine.start(start, coroutine, block)
}
```
- CoroutineContext - 可以理解为协程的上下文，是一种key-value数据结构
<table border="1">
  <tr bgcolor="#999999">
    <th width="385">CoroutineContext</th>
    <th width="475">List<E>  </th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td><E: Element> get(key: Key<E>): E </td>
    <td>get(int index)</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>plus(context: Element)</td>
   <td>add(int index, E element)</td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>minusKey(key: Key<*>)</td>
   <td>remove(E element)</td>
  </tr>
</table>

<br></br>
- CoroutineDispatcher 协程运行的线程调度器
协程调度器
<img src="/imgs/thread/coroutine_dispatcher.png" style="zoom:50%" width="1500"/>
<table border="1">
  <tr bgcolor="#999999">
    <th width="385">模式</th>
    <th width="475">说明</th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td><Dispatchers.IO </td>
    <td>显示指定协程运行的线程,为IO线程 </td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>Dispatchers.Main  </td>
   <td>指定这个协程运行在主线程</td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>Dispatchers.Default</td>
   <td>默认的,启动携程时会启动一个线程</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>Dispatchers.Unconfined</td>
   <td>不指定，就是在当前线程运行,协程恢复后的运行的线程取决于协程挂起时所在的线程</td>
  </tr>
</table>
<br></br>
- CoroutineStart - 启动模式，默认是DEAFAULT，也就是创建就启动；还有一个是LAZY，意思是等你需要它的时候，再调用启动
<table border="1">
  <tr bgcolor="#999999">
    <th width="385">模式</th>
    <th width="475">说明</th>
  </tr>
  <tr bgcolor="#ffffff">
    <td>CoroutineStart().DEAFAULT </td>
    <td>模式模式，创建即启动协程,可随时取消</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>ATOMIC</td>
   <td>自动模式,同样创建即启动,但启动前不可取</td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>LAZY</td>
   <td>延迟启动模式，只有当调用start方法时才会启动</td>
  </tr>
</table>

### 协程挂起,恢复原理逆向剖析

- 挂起函数

 **被关键字`suspend`修饰的方法在编译阶段,编译器会修改方法的签名. 包括返回值,修饰符,入参,方法体实现**。协程的挂起是靠挂起函数中实现的代码。

```kotlin
suspend fun request(): String {
     delay(2 * 1000)//suspend fun()
     println("after delay")
     return "result from request"
}

public static final Object request(Continuation completion) {
  ContinuationImpl requestContinuation = completion;
        if ((completion.label & Integer.MIN_VALUE) == 0) 
            requestContinuation = new ContinuationImpl(completion) {
                @Override
                Object invokeSuspend(Object o) {
                    label |= Integer.MIN_VALUE;
                    return request(this);
                }
            };
        }
        switch (requestContinuation.label) {
            case 0: {
                requestContinuation.label = 1;
                Object delay = DelayKt.delay(2000, requestContinuation);
                if (delay == COROUTINE_SUSPENDED) {
                    return COROUTINE_SUSPENDED;
                }
            }
        }
  System.out.println("after delay")
  return "result from request";
}
```

- 协程挂起与协程恢复

**协程的核心是挂起----恢复,挂起--恢复的本质是return & callback回调**
<img src="/imgs/thread/coroutine_resume.png" style="zoom:50%" width="1500"/>



###  协程回顾

- **什么是协程**

  - 协程是一种解决方案，是一种解决嵌套,并发,弱化线程概念的方案。能让多个任务之间更好的协作,能够以同步的方式编排代码完成异步工作。将异步代码写的像同步代码一样直观。

    

- **协程的启动**

  - 根据创建协程指定的调度器HandlerDispatcher,DefaultScheduler,UnconfinedDispatcher来执行任务,以决定协程中的代码块运行在那个线程上。

    

- **协程的挂起，恢复**

  - 本质是方法的挂起，恢复。本质是return +callback。
  - 用编译时的变换处理方法间的callback，这样可以很直观地写顺序执行的异步代码。

  

- **协程是线程框架吗？**

  - 协程的本质是编译时return +callback。只不过在调度任务时提供了能够运行在IO线程的调度器。

    

- **什么时候使用协程**

  - 多任务并发流程控制场景使用比较好, 流程控制比较简单,不会涉及线程阻塞与唤醒，性能比java并发控制手段高。



  


