---
title: 解锁Android多线程开发核心知识点
---

<!--more-->

## 目录

- 什么是线程并发安全
- 线程安全的几种分类
- 如何保证线程安全



### 什么是线程并发安全

- 演示买票场景
- 线程安全的本质是 能够让并发线程，有序的运行（这个有序有可能是先来后到排队，有可能有人插队，但不管怎么着，同一时刻只能一个线程有权访问同步资源），线程执行的结果，能够对其他线程可见。



### 线程安全的几种分类

- synchronized关键字
- ReentrantLock锁
- AtomicInteger...原子类

<img src="/imgs/thread/线程安全.png" style="zoom:50%" width="1600"/>

- synchronized，ReentrantLock-锁
<img src="/imgs/thread/悲观锁.png" style="zoom:50%" width="1600"/>

- 原子类-自旋

<img src="/imgs/thread/乐观锁.png" style="zoom:50%" width="1600"/>

- 锁适合写操作多的场景，先加锁可以保证写操作时数据正确。

- 原子类适合读操作多的场景，不加锁的特点能够使其读操作的性能大幅提升。

### 如何保证线程安全

- AtomicInterger原子包装类,CAS（Compare-And-Swap）实现无锁数据更新。自旋的设计能够有效避免线程因阻塞-唤醒带来的系统资源开销

- 适用场景：多线程计数,原子操作,并发数量小的场景。

  ```java
  //# 1构建对象
  AtomicInteger atomicInteger = new AtomicInteger(1);
  
  //#2 调用Api
  atomicInteger.getAndIncrement();
  atomicInteger.getAndAdd(2);
  
  atomicInteger.getAndDecrement()
  atomicInteger.getAndAdd(-2);
  ```

- volatile可见性修饰

  volatile修饰的成员变量在每次被线程访问时，都强迫从共享内存重新读取该成员的值，而且，当成员变量值发生变化时，强迫将变化的值重新写入共享内存，

  不能解决非原子操作的线程安全性。性能不及原子类高

  ```java
   volatile int count
   public void increment() {
       //其他线程可见
       count =5 
       
       //非原子操作,其他线程不可见 
       count=count+1;
       count++;
   }
  ```



- synchronized 

  锁java对象，锁Class对象，锁代码块

  - 锁方法。加在方法上，未获取到对象锁的其他线程都不可以访问该方法，

  ```java
  synchronized void printThreadName() {
          
  }
  ```

  - 锁Class对象。加在static 方法上相当于给Class对象加锁，哪怕是不同的java 对象实例，也需要排队执行

  ```java
  static synchronized void printThreadName() {
        
  }
  ```

  - 锁代码块。未获取到对象锁的其他线程可以执行同步块之外的代码

  ```java
  void printThreadNam() {
   String name = Thread.currentThread().getName();
   System.out.println("线程：" + name + " 准备好了...");
      synchronized (this) {
                  
      }
  }
  ```

  Synchronized的优势是什么呢？

  - 哪怕我们一个同步方法中出现了异常，那么jvm也能够为我们自动释放锁，能主动从而规避死锁。不需要开发者手动释放锁，

  劣势是什么呢？

   - 必须要等到获取锁对象的线程执行完成，或者出现异常，才能释放掉。不能中途释放锁，不能中断一个正在试图获得锁的线程
   - 另外咱们也不知道多个线程竞争锁的时候，获取锁成功与否，所以不够灵活
   - 每个锁仅有单一的条件(某个对象)不能设定超时

  

- ReentrantLock悲观锁 ，可重入锁，公平锁，非公平锁

  - 基本用法

  ```java
  ReentrantLock lock = new ReentrantLock();
  try{
    
    lock.lock()
    ...
  }finally{
    lock.unLock()
  }
  ```

  ```java
   void lock()//获取不到会阻塞
   boolean tryLock()//尝试获取锁，成功返回true。
   boolean tryLock(3000, TimeUnit.MILLISECONDS)//在一定时间内去不断尝试获取锁
   void lockInterruptibly();//可使用Thread.interrupt()打断阻塞状态，退出竞争，让给其他线程
  ```

  - 可重入,避免死锁

  ```java
  ReentrantLock lock = new ReentrantLock();
  public void doWork(){
    try{
    
    lock.lock()
    doWork();//递归调用,使得统一线程多次获得锁
  }finally{
    lock.unLock()
  }
  }
  ```

  - 公平锁与非公平锁，
  - 公平锁，所有进入阻塞的线程排队依次均有机会执行
  - 默认非公平锁，允许线程插队，避免每一个线程都进入阻塞，再唤醒，性能高。因为线程可以插队，导致队列中可能会存在线程饿死的情况，一直得不到锁，一直得不到执行。

  ```java
  ReentrantLock lock = new ReentrantLock(true/false);
  ```

  

- ReentrantLock进阶用法 --Condition条件对象

  可使用它的await-singnal 指定唤醒一个(组)线程。相比于wait-notify要么全部唤醒，要么只能唤醒一个，更加灵活可控

  ```java
  ReentrantLock lock = new ReentrantLock();
  Condition worker1 = lock.newCondition();
  Condition worker2 = lock.newCondition();
  
  class Worker1{
    .....
      worker1.await()//进入阻塞,等待唤醒
    .....
  }
  
  class Worker2{
    .....
      worker2.await()//进入阻塞,等待唤醒
    .....
  }
  
  class Boss{
    
    if(...){
      worker1.signal()//指定唤醒线程1
    }else{
      worker2.signal()//指定唤醒线程2
    }
  }
  ```

- ReentrantReadWriteLock共享锁，排他锁

  - 共享锁,所有线程均可同时获得，并发量高，比如在线文档查看
  - 排他锁,同一时刻只有一个线程有权修改资源，比如在线文档编辑

  ```java
  ReentrantReadWriteLock reentrantReadWriteLock;
  ReentrantReadWriteLock.ReadLock readLock;
  ReentrantReadWriteLock.WriteLock writeLock;
  ```

  

### 如何正确的使用锁&原子类

- 减少持锁时间

  尽管锁在同一时间只能允许一个线程持有，其它想要占用锁的线程都得在临界区外等待锁的释放，这个等待的时间根据实际的应用及代码写法可长可短。

```java
public void syncMethod(){
    noneLockedCode1();//2s
    synchronized(this){
        needLockedMethed();2s
    }
    noneLockedCode2();2s
}
```

- 锁分离

  读读，读写，写读，写写。只要有写锁进入才需要做同步处理，但是对于大多数应用来说，读的场景要远远大于写的场景，因此一旦使用读写锁，在读多写少的场景中，就可以很好的提高系统的性能，这就是锁分离

|      |   读锁   |   写锁   |
| ---- | :------: | :------: |
| 读锁 | 可以访问 | 不可访问 |
| 写锁 | 不可访问 | 不可访问 |

- 锁粗化

  多次加锁,

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


