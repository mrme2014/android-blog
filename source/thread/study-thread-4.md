---
title: 深入理解Android线程池实现原理
---

<!--more-->

## 目录

- 需求分析
- 成果展示
- Coding实现

### 需求分析

**设计一个全局通用的线程池组件-HiExecutor**

- 支持任务优先级
- 支持线程池暂停、恢复、关闭
- 支持异步任务结果回调


### 成果展示

<video src="video.mp4" width=770 controls></video>

```java
HiExecutor.execute(10, new Runnable {
    ...do what you want.      
});
 
void pause()//暂停线程池
void reume()//恢复线程池
void execute(int priority,Runnable runnable)//仍给线程池结果不管
void execute(int priority,Callable<T> runnable)//可以拿到任务执行的结果
```



### Coding实现

- 线程池参数构造

```java
int corePoolSize=cpuCount+1;
int maximumPoolSize= cpuCount*2+1;
BlockingQueue<Runnable> workQueue= new PriorityBlockingQueue()

new ThreadPoolExecutor(
            corePoolSize,
            maxPoolSize,
            keepAliveTime
            TimeUnit.SECONDS
            new PriorityBlockingQueue,
            threadFactory）
```

- 实现线程池中任务按优先级执行

```java
class PriorityRunnable(int priority,Runnable runnable) implements Runnable, Comparable<PriorityRunnable>{
  @override
  public  void run() {
       //让真正的runnable任务去执行
       runnable.run()
  }
  
  @override
  public int compareTo(PriorityRunnable other) {
    //实现Comparable接口,复写该方法,用以比较各任务的优先级排序
      return if (this.priority < other.priority) 1 else if (this.priority > other.priority) -1 else 0
    }
}
}
```



- 实现线程池的暂停/恢复

```java
class ThreadPoolExecutor{
  ReentrantLock lock = new ReentrantLock();
  Condition pauseCondition = lock.newCondition();
  void beforeExecute(Thread thread, Runnable runnable){
    if(isPaused){
    try{
      //使当前线程阻塞
        lock.lock();
        pauseCondition.await()
    }finally{
      lock.unlock();
    }
  }
 }
}
```



- 实现异步任务结果主动切换到主线程

```java
abstract class Callable<T> implements Runnable {
  @override     
  public void run() {
          mainHandler.post { onPrepare() }

          T t = onBackground()

          mainHandler.post { onCompleted(t) }
        }

  public void onPrepare() {
       //任务开始前的准备，比如转菊花
  }
  //子线程真正的去执行
  abstract T onBackground()
          
  //执行完抛到主线程
   abstract void onCompleted(T t)
 }
```


  


