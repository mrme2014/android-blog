---
title: 深入理解Android线程池实现原理
---

<!--more-->

## 目录

- 为什么要引入线程池
- Java中几种默认的线程池
- 线程池实现原理
- 线程池中线程复用原理



### 为什么要引入线程池

- <font style='color=red'>**降低资源消耗**</font>。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- **提高响应速度**。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
- **提高线程的可管理性**。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。



### Java中几种默认的线程池

- 如何构建线程池

```java
ThreadPoolExecutor executor =
new ThreadPoolExecutor(5,
                       100,
                       10,
                       TimeUnit.SECONDS,
                       new PriorityBlockingQueue<Runnable>());

for(int index=0;index=100;index++){
  executor.execute(new Runnable());
}  
```

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,       
                            RejectedExecutionHandler handler)
```
<table border="1">
  <tr bgcolor="#999999">
    <th>参数</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>corePoolSize</td>
    <td>线程池中**核心**线程数量    </td>
  </tr>
  <tr>
   <td>maximumPoolSize</td>
   <td>**最大**能创建的线程数量 </td>
  </tr>
  <tr>
   <td>keepAliveTime</td>
   <td>**非核心**线程最大存活时间</td>
  </tr>
  <tr>
   <td>unit</td>
   <td>keepAliveTime的时间单位</td>
  </tr>
  <tr>
   <td>workQueue</td>
   <td>等待队列。当任务提交时，如果线程池中的线程数量**大于等于corePoolSize的时候，把该任务放入等待队列**</td>
  </tr>
  <tr>
   <td>threadFactory</td>
   <td>**线程创建工程厂**。默认使用Executors.defaultThreadFactory() 来创建线程，线程具有相同的NORM_PRIORITY优先级并且是非守护线程</td>
  </tr>
  <tr>
   <td>handler</td>
   <td>**线程池的饱和拒绝策略**。如果阻塞队列满了并且没有空闲的线程，这时如果继续提交任务，就需要采取一种策略处理该任务</td>
  </tr>
</table>
- JUC包下Executors提供的几种线程池

```java
//单一线程数,同时只有一个线程存活,但线程等待队列无界
 Executors.newSingleThreadExecutor();
//线程可复用线程池,核心线程数为0，最大可创建的线程数为Interger.max,线程复用存活时间是60s.  
 Executors.newCachedThreadPool();
//固定线程数量的线程池
 Executors.newFixedThreadPool(int corePoolSize);
//可执行定时任务,延迟任务的线程池
 Executors.newScheduledThreadPool(int corePoolSize);
```

- 线程池重要方法

```java
void execute(Runnable run)//提交任务,交由线程池调度
void shutdown()//关闭线程池,等待任务执行完成
void shutdownNow()//关闭线程池，不等待任务执行完成
int  getTaskCount()//返回线程池找中所有任务的数量
int  getCompletedTaskCount()//返回线程池中已执行完成的任务数量
int  getPoolSize()//返回线程池中已创建线程数量 
int  getActiveCount()//返回当前正在运行的线程数量
```

- 线程池状态流转

<img src="/imgs/thread/线程池状态.png" style="zoom:50%" width="2000"/>

- execute提交任务流程

<img src="/imgs/thread/execute.png" style="zoom:50%" width="2000"/>


- 双层for循环流程控制

  ```java
  public static void main(String[] args) {
      int count = 0;
      retry:
      for (int i = 0; i < 2; i++){
          for (int j = 0; j < 5; j++){
              count++;
              if (count == 3){
                   break;
               }
               
              if(count ==4){
                   break retry;
              }
                
              System.out.print(count+" ");
              }
          }
      System.out.print("双次for循环结束...");
  }
  ```

  


