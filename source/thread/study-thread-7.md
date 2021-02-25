---
title: 架构师如何做多线程优化
---

<!--more-->

## 目录
- 线程池
- 线程安全
- 线程协作
- 协程



### 线程池

线程池吞吐量，任务优先级，线程池监控

```java
ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory) 
```
<table border="1">
  <tr bgcolor="#999999">
    <th width="310">workQueue类型</th>
    <th >作用</th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td>SynchronousQueue</td>
    <td>一个**不存储元素的阻塞队列**。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于**阻塞状态**，吞吐量会有较高提升，静态工厂方法Executors.newCachedThreadPool使用了这个队列 </td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>PriorityBlockingQueue</td>
   <td>一个具有**优先级的无限阻塞队列**</td>
  </tr>
</table>
<br></br>

### 并发安全

- ##### Synchronized

  在客户端绝大多数并发场景下，不要求知道锁的细节的话，直接使用synchoriznd关键字加锁 就可以了。自从1.6优化了之后，性能已经够用了。

- ##### 减少持锁时间

  尽管锁在同一时间只能允许一个线程持有，其它想要占用锁的线程都得在临界区外等待锁的释放，这个等待的时间我们希望尽可能的短。

```java
public void syncMethod(){
    noneLockedCode1();//2s
    synchronized(this){
        needLockedMethed();2s
    }
    noneLockedCode2();2s
}
```

- ##### 锁分离

  读读，读写，写读，写写。只要有写锁进入才需要做同步处理，但是对于大多数应用来说，读的场景要远远大于写的场景，因此一旦使用读写锁，在读多写少的场景中，就可以很好的提高系统的性能。
<table border="1">
  <tr bgcolor="#999999">
    <th width="310"></th>
    <th width="310">读锁</th>
    <th width="310">写锁</th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td>读锁</td>
    <td>可以访问 </td>
    <td>不可访问 </td>
  </tr>
  <tr  bgcolor="#eeeeee">
    <td>写锁</td>
    <td>不可访问 </td>
    <td>不可访问 </td>
  </tr>
</table>
<br></br>
- ##### 锁粗化

```java
public void doSomethingMethod(){
    synchronized(lock){
      //do some thing
    }
    .....
    //这是还有一些代码，做其它不需要同步的工作，但能很快执行完毕
    .....
    synchronized(lock){
        //do other thing
    }
}
```

```java
public void doSomethingMethod(){
    //进行锁粗化：整合成一次锁请求、释放
    synchronized(lock){
        //do some thing
        //做其它不需要同步但能很快执行完的工作
        //do other thing
    }
}
```

- ##### 原子类

  在并发量不大且读多写少的场景下使用较好，因为原子类再写入数据的 是通过do-while循环，不停的把当前内存的值和寄存器中的值 做比较来实现的，在并发量大的时候，相比原子类的自旋,加锁的性能会更高。

- ##### 并发容器

  ConcurrentHashMap,CopyOnWriteArrayList



### 线程协作

- ##### Object.wait--notify(notifyAll)

  搭配Synchronized使用,简单易用,特别要注意wait先于notify调用,否则会出现死锁。

- ##### Condition.await--signal(signalAll)

  搭配ReentrantLock,指定唤醒,按组唤醒。更加安全和高效。因此通常来说比较推荐使用Condition

- ##### CountDownLatch

  是通过一个计数器来实现的，计数器的初始值是线程的数量。每当一个线程执行完毕后，计数器的值就-1，当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了

  **wait():** 让调用者线程进入阻塞,谁调用谁阻塞

  **countDown():** 使得计数器的值-1,为0时，阻塞线程被唤醒

```java
CountDownLatch latch = new CountDownLatch(5);
        ExecutorService service = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 5; i++) {
            final int no = i + 1;
            Runnable runnable = new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep((long) (Math.random() * 10000));
                        System.out.println("No." + no + "准备好了。");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        latch.countDown();
                    }
                }
            };
            service.submit(runnable);
        }
System.out.println("等待所有人准备完毕.....");
latch.await();
System.out.println("所有人都准备好了，可以发车了");
```



- ##### Semaphore

  信号量,常用于并发限流,基于“许可证”的并发控制

  **void acquire(int permits):** 阻塞获取许可证

  **boolean tryAcquire(int permits):** 尝试获取许可证,立刻返回

  **boolean tryAcquire(int permits, long timeout, TimeUnit unit)**

  **release(int permits):** 释放已获得许可数量

```java
Semaphore semaphore = new Semaphore(3);
for (int i=0;i<10;i++){
      int finalIndex =i+1;
      new Thread(new Runnable(){
         @Override
            public void run() {
               //获取许可证
                semaphore.acquire();
                Thread.sleep(newRandom().nextInt(5000)); //模拟随机执行时长  
                System.out.println("working thread No:"+finalIndex)
               //释放许可证
               semaphore.release(); 
            }
      }).start()     
}
```



### 协程

协程是一种解决方案，是一种解决嵌套,并发,弱化线程概念的方案。能让多个任务之间更好的协作,能够以同步的方式编排代码完成异步工作。将异步代码写的像同步代码一样直观。

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

  


