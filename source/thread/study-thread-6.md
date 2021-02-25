---
title: Kotlin协程应用
---

<!--more-->

## 目录
- 需求分析
- 效果展示
- Coding



### 需求分析

 **如何让普通函数适配协程,成为“真正的挂起函数”。即让调用方以同步的方式拿到异步任务返回结果**

```java
//场景描述
fun getConfigContent(){
  parseAssetsFile("config.json"){fileContent->
    println(fileContent)
  }
}

//CoroutineScene4.kt
fun parseAssetsFile(fileName:String,callback(String)->Unit){
  Thread(Runnable {
       //readfile(filename)  
        Thread.sleep(2000)
        callback("assets file content") 
       }).start()
    }
}
```

```kotlin
//方案一:
lifecycleScope.launch{
  val fileContent =   lifecycleScope.async{parseAssetsFile()}.await()
  //delay()
  println(fileContent)
}

suspend parseAssetsFile(fileName:String):String{
  return "assets file content"
}
```

```kotlin
//方案二
lifecycleScope.launch{
  val fileContent = parseAssetsFile()
  println(fileContent)
}

suspend parseAssetsFile(fileName:String):String{
  //suspendCoroutine
  return suspendCancellableCoroutine { continuation ->
        Thread{
          //io ....
        continuation.resumeWith(Result.success("assets file content")) 
        }.start()                               
  }
}
```

```kotlin
public suspend fun delay(timeMillis: Long) {
    return suspendCancellableCoroutine sc@ { cont: CancellableContinuation<Unit> ->
        cont.context.delay.scheduleResumeAfterDelay(timeMillis, cont)
    }
}
```


  


