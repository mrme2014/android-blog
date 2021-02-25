---
title: 深入理解Android消息机制的原理
---

## 目录
- Handler & Looper & MessageQueue三角关系简述
- 消息入队源码分析
- 消息分发原理深入分析
- ThreadLocal内存泄漏原理
- Handler面试经典八问

### Handler & Looper & MessageQueue三角关系简述

- 一个线程至多一个looper；一个looper有一个MessageQueue；一个MessageQueue对应多个message；一个MessageQueue对多个Handler。
- 消息类型有同步、异步、同步屏障消息。

<center> <img src="/imgs/handler/WechatIMG34.png" style="zoom:50%;"></center>

```java
        //1. 直接在Runnable中处理任务
        handler.post (runnable= Runnable{
          
        })

        //2.使用Handler.Callback 来接收处理消息
        val handler =object :Handler(Callback {
            return@Callback true
          
           }
        )
          
        //3. 使用handlerMessage接收处理消息
        val handler =object :Handler(){
            override fun handleMessage(msg: Message) {
                super.handleMessage(msg)
            }
        }
        val message = Message.obtain()
        handler.sendMessage(message)
```



- **为什么主线程中创建Handler没有绑定looper，也能发送消息呢???**

  - 在ActivityThread.main()方法中调用了Looper.prepareMainLooper()方法创建了主线程的looper,  然后调用了loop()开启消息队列轮训。

  - 通常我们认为ActivityThread就是主线程，事实上它并不是一个线程，而是主线程操作的管理者。

```kotlin
public Handler() {this(null, false);}
public Handler(boolean async) { this(null, async); }
public Handler(Callback callback, boolean async) {
        //如果没调用Looper.prepare则为空
        //主线程创建Handler不用创建Looper，就是ActivityThread在进程入口帮我们调用了Looper.prepareMainLooper()，
        //之后我们创建的handler在这里就可以在这里跟当前线程的Looper进行绑定。
        
        //问题是Looper.myLooper()怎么保证获取到的是本线程的looper对象？    
        mLooper = Looper.myLooper();
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

```java
class Looper{
  // sThreadLocal 是static的变量，可以先简单理解它相当于map，key是线程，value是Looper，
    //那么你只要用当前的线程的sThreadLocal获取当前线程所属的Looper。
   static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    //Looper 所属的线程的消息队列
   final MessageQueue mQueue;
  
   private static void prepare(boolean quitAllowed) {
         //一个线程只能有一个Looper，prepare不能重复调用。
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
  
  private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
    }
  
  public static @Nullable Looper myLooper() {
        //具体看ThreadLocal类的源码的get方法，
        //简单理解相当于map.get(Thread.currentThread()) 获取当前线程的Looper
        return sThreadLocal.get();
    }
}
```



- **如何让子线程拥有消息分发的能力？?**

  像ActivityThread一样，在入口给他创建looper。

```java
class HandlerThread extends Thread{
  private  Handler handler
   @overried
   public void run(){
       Looper.prepare()
       //面试题：如何在子线程中弹出toast  Toast.show() 
       //Toast.show() 
          
       createHandler()
   
       Looper.loop()
   }
  
   private void createHandler(){
     handler  = new Handler(){
        @overried
        public void handleMessage(Message msg){
            //处理主线程发送过来的消息
        }
     }
   }
}
```



### 消息入队
<center> <img src="/imgs/handler/sendmessage.png" style="zoom:50%;"></center>

- 获取消息体——享元设计模式

```java
class Handler{
   public final boolean post(@NonNull Runnable r) {
       return  sendMessageDelayed(getPostMessage(r), 0);
   }
   private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
   }  
}

class Message{
    public static final Object sPoolSync = new Object();
    private static Message sPool;
    private static int sPoolSize = 0;   
    private static final int MAX_POOL_SIZE = 50;
  
    public static Message obtain() {
        //单向链表形式的对象池
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; 
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
  
  void recycleUnchecked() {
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
}
```


<center> <img src="/imgs/handler/WechatIMG30.png" style="zoom:50%;" width="1900"></center>

- **enqueueMessage**
  - 如果新消息的被执行时间`when`比队列中任何一个消息都早，则插入队头。唤醒Looper。
  - 如果当前队头是同步屏障消息，新消息是异步消息，则唤醒Looper。

```java
class Handler{
  private boolean enqueueMessage(@NonNull MessageQueue queue, Message msg,long uptimeMillis) {
        msg.target = this;
        ...
        return queue.enqueueMessage(msg, uptimeMillis);
    }
}

class MessageQueue{
  boolean enqueueMessage(Message msg, long when) {
        //普通消息的target字段不能为空，否则不知道该交由谁来处理这条消息
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
 
        synchronized (this) {
          
            msg.when = when;
            //mMessages始终是队头消息
            Message p = mMessages;
            boolean needWake;
            //如果队头消息为空，或消息的when=0不需要延迟,或newMsg.when<headMsg.when        
            //则插入队头
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                //如果当前Looper处于休眠状态，则本次插入消息之后需要唤醒
                needWake = mBlocked;
            } else {
                
                //要不要唤醒Looper= 当前Looper处于休眠状态 & 队头消息是同步屏障消息 & 新消息是异步消息
                //目的是为了让异步消息尽早执行
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    //按时间顺序 找到该消息 合适的位置
                    // >>msg1.when=2-->msg.when=4-->msg.when-->6
                    //>>msg1.when=2-->msg.when=4-->【newMsg.when=5】-->msg.when=6   
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                //跳转链表中节点的指向关系
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }
            //唤醒looper
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
}
```

- **postSyncBarrier同步屏障消息**
  - message.target=null,这类消息不会被真的执行，它起到flag标记的作用，MessageQueue在遍历消息队列时,
    如果队头同步屏障消息，那么会忽略同步消息，优先让异步消息得到执行。这就是它的目的，一般异步消息和同步屏障消息会一同使用。
  - 异步消息& 同步屏障 使用场景
    - ViewRootImpl接收屏幕垂直同步信息事件用于驱动UI测绘、
    - ActivityThread接收AMS的事件驱动生命周期
    - InputMethodManager分发软键盘输入事件、
    - PhoneWindowManager分发电话页面各种事件。

```java
class MessageQueue{
  public int postSyncBarrier() {
        //currentTimeMillis()系统当前时间，即日期时间，可以被系统设置修改，如果设置系统时间，时间值会发生跳变。
        //uptimeMillis() 自开机后，经过的时间，不包括深度休眠的时间 
        //sendMessageDelay,postDelay 也都使用了这个时间戳
        //意思是指，发送了这条消息，在这期间如果设备进入休眠状态，那么消息是不会被执行的，设备唤醒之后才会被执行
        return postSyncBarrier(SystemClock.uptimeMillis());
 }
 private int postSyncBarrier(long when) {
        synchronized (this) {
            //从消息池子复用，构建新消息体
            final Message msg = Message.obtain();
            //并没有给target字段赋值
            //区分是不是同步屏障，就看target是否等于null
            //换句话，target=null，则该消息被认为是同步屏障消息
            msg.when = when;

            Message prev = null;
            Message p = mMessages;
            //遍历队列所有消息，直到 找到一个 message.when>msg.when的消息，决定新消息插入的位置
            //>>msg1.when=2-->msg.when=4-->msg.when-->6
            //>>msg1.when=2-->msg.when=4-->【newMsg.when=5】-->msg.when=6   
            if (when != 0) {
                while (p != null && p.when <= when) {
                    prev = p;
                    p = p.next;
                }
            }
            //如果找到了合适的位置则插入
            if (prev != null) {
                msg.next = p;
                prev.next = msg;
            } else {
                //如果没找到则直接放到队头
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
}
```

### 消息分发

```java
class Looper{
   public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
        ......
        for (;;) {
            //无限循环，调用 MessageQueue.next获取可以执行的消息
            //注意看官方注释。might block 可能会被阻塞。
            Message msg = queue.next(); // might block
            if (msg == null) {
                return;
            }
            //分发消息
            msg.target.dispatchMessage(msg);
            ......
            //回收message对象，留着复用  
            msg.recycleUnchecked();
        }
    }
}
```

```java
class MessageQueue{
  Message next() {
        int nextPollTimeoutMillis = 0;
        for (;;) {
            //如果nextPollTimeoutMillis>0 ，此时looper会进入休眠状态，第一次循环由于等0，所以不会
            //如果第一次循环没找到需要处理的消息，则nextPollTimeoutMillis会被更新，
            //第二次循环时，如果nextPollTimeoutMillis！=0looper线程就会进入阻塞状态。
            //在此期间主线程没有实际工作要做,会释放cpu资源占用。该方法会超时自主恢复，或者插入新消息时被动唤醒
            nativePollOnce(ptr, nextPollTimeoutMillis);

            //当Looper被唤醒时，会继续向下执行
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                //如果队头是屏障消息，则尝试找到一个异步消息
                if (msg != null && msg.target == null) {
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                       //如果上面找到的消息它的时间，还没到需要执行的时机
                      //则更新nextPollTimeoutMillis。也就是下一次循环需要阻塞的时间值
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                       //找到了需要处理的消息
                        mBlocked = false;
                        //由于这个消息即将被处理，所以需要把它从队列中移除
                        //通过调整节点的指向关系，达到队列元素移除的目的
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    //如果没有找到消息，即队列为空，looper将进入永久休眠，直到新消息达到。
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

                //当队列消息为空，也就是所有任务都处理完了，或者队头消息已经达到了 可执行时间点。
              //此时派发通知 Looper即将进入空闲状态
                if (pendingIdleHandlerCount < 0 && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
              if(pendingIdleHandlerCount<=0)return
            //注册 MessageQueue.IdleHandler，可以监听当前线程的Looper是否处于空闲状态。也就意味当前线程是否处于空闲状态。
            //在主线程中可以监听这个事件来做延迟初始化，数据加载,日志上报....
            //而不是有任务就提交，从而避免抢占重要资源。
            for (int i = 0; i < pendingIdleHandlerCount; i++) {              
                final IdleHandler idler = mPendingIdleHandlers[i];
               boolean keep =idler.queueIdle();
                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            } 
            //此时置为0，他认为既然监听了线程的空闲，那么在这个  
             //queueIdle回调里，很有可能又会产生新的消息，为了让消息尽可能早的得到执行，所以此时不需要休眠了 
            nextPollTimeoutMillis=0;  
        }
    }
}
```


<center> <img src="/imgs/handler/messagequeue.png" style="zoom:60%;" width="1650"></center>
<center> <img src="/imgs/handler/nativemessage.png" style="zoom:60%;" width="1650"></center>

- **Handler.disptachMessage**

  消息分发的优先级：

  1. Message的回调方法：message.callback.run()，优先级最高
  2. Handler的回调方法：Handler.mCallback.handleMessage(msg)
  3. Handler的默认方法：Handler.handleMessage(msg)

```java
 public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```



### ThreadLocal

- ThreadLocal提供了线程独有的局部变量存储能力，可以在整个线程存活的过程中随时取用，方便了一些逻辑的实现。

```java
HiExecutor.execute(runnable ={
      1. method1(),method2(),method3(),method4()
})
  
User user = new User();
ThreadLocal local = new ThreadLocal<User>() {
     @Nullable
     @Override
     protected User initialValue() {
          return user;
         }
   };  

//然后开两个线程，去操作这个 local对象的usr对象.那么每个线程会对user对象进行一次拷贝，进行数据修改之后。

//去验证  修改后的数据会同步到主线程中的user对象，会不会影响到别的线程的user对象的值。
```

- ThreadLocalMap的数据模型
  - 每个线程对应一个ThreadLocalMap对象。本质是一个数组实现的散列map。
  - 每个元素都是Entry对象。key=threadLocal，value为任意值。
<center> <img src="/imgs/handler/threadlocal.png" style="zoom:50%;"></center>





```java
class ThreadLocal{
   public void set(T value) {
        //获取当前线程的ThreadLocalMap对象
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);//更新值，key=threadlocal
        else
            createMap(t, value);//创建并设置值
    }
  
   void createMap(Thread t, T firstValue) {
        //每个线程都有一个ThreadLocalMap，key为threadlocal对象
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
  
    
   public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
  
    private T setInitialValue() {
        //得到被调用的threadLocal的初始值(对应上文的主线程threadLocal对象)
      
        T value = initialValue();
        //得到当前线程，对应线程池的每个线程
        Thread t = Thread.currentThread();
        //获取 or 创建 每个线程的ThreadLocalMap，并把初始值存放进去，
        //从而实现变量拷贝，变成线程独享。
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
     }
  
  static class ThreadLocalMap{
    //本质是一个数组
    private Entry[] table;
   
    static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;
            Entry(ThreadLocal<?> key, Object value) {
                //key 被弱引用持有
                super(k);
                //value 强引用赋值
                value = v;
            }
        }
    }
  
   private void set(ThreadLocal<?> key, Object value) {
            int len = tables.length;
            //获取key的hash散列值，使得value 均匀的分布在table[]数组内
            int i = key.threadLocalHashCode & (len-1);
            .....
            replaceStaleEntry(...)//移除key=null的元素
            .....  
            tables[i] = new Entry(key, value);
            ......
     }
}
```



### Handler面试八问

- 为什么主线程不会因为 `Looper.loop()` 里的死循环卡死？

  > 主线程确实是通过 `Looper.loop()` 进入了循环状态，因为这样主线程才不会像我们一般创建的线程一样，当可执行代码执行完后，线程生命周期就终止了。
  >
  > 在主线程的 MessageQueue 没有消息时，便阻塞在 `MessageQueue.next()` 中的 `nativePollOnce()` 方法里，此时主线程会释放 CPU 资源进入休眠状态，直到新消息达到。所以主线程大多数时候都是处于休眠状态，并不会消耗大量 CPU 资源。
  >
  > 这里采用的 linux的epoll 机制，是一种 IO 多路复用机制，可以同时监控多个文件描述符，当某个文件描述符就绪（读或写就绪），则立刻通知相应程序进行读或写操作拿到最新的消息，进而唤醒等待的线程。

- `post` 和 `sendMessage` 两类发送消息的方法有什么区别？

  > `post` 一类的方法发送的是 Runnable 对象，但是其最后还是会被封装成 Message 对象，将 Runnable 对象赋值给 Message 对象中的 callback 变量，然后交由 `sendMessageAtTime()` 方法发送出去。在处理消息时，会在 `dispatchMessage()` 方法里首先被 `handleCallback(msg)` 方法执行，实际上就是执行 Message 对象里面的 Runnable 对象的 `run` 方法。
  >
  > 而 `sendMessage` 一类的方法发送的直接是 Message 对象，处理消息时，在 `dispatchMessage` 里优先级会低于 `handleCallback(msg)` 方法，是通过自己重写的 `handleMessage(msg)` 方法执行。

- 为什么要通过 `Message.obtain()` 方法获取 Message 对象？

  > `obtain` 方法可以从全局消息池中得到一个空的 Message 对象，这样可以有效节省系统资源。同时，通过各种 obtain 重载方法还可以得到一些 Message 的拷贝，或对 Message 对象进行一些初始化。

- Handler 实现发送延迟消息的原理是什么？

  > 我们常用 `postDelayed()` 与 `sendMessageDelayed()` 来发送延迟消息，其实最终都是将延迟时间转为确定时间，然后通过 `sendMessageAtTime()` -> `enqueueMessage` -> `queue.enqueueMessage` 这一系列方法将消息插入到 MessageQueue 中。所以并不是先延迟再发送消息，而是直接发送消息，再借助MessageQueue 的设计来实现消息的延迟处理。
  >
  > 消息延迟处理的原理涉及 MessageQueue 的两个静态方法 `MessageQueue.next()` 和 `MessageQueue.enqueueMessage()`。通过 Native 方法阻塞线程一定时间，等到消息的执行时间到后再取出消息执行。

- 同步屏障 SyncBarrier 是什么？有什么作用？

  > 在一般情况下，同步和异步消息处理起来没有什么不同。只有在设置了同步屏障后才会有差异。同步屏障从代码层面上看是一个 Message 对象，但是其 target 属性为 null，用以区分普通消息。在 `MessageQueue.next()` 中如果当前消息是一个同步屏障，则跳过后面所有的同步消息，找到第一个异步消息来处理。但是开发者调用不了。在ViewRootImpl的UI测绘流程有体现

- IdleHandler 是什么？有什么作用？

  > 当消息队列没有消息时调用或者如果队列中仍有待处理的消息，但都未到执行时间时，也会调用此方法。用以监听主线程空闲状态。

- 为什么非静态类的 Handler 导致内存泄漏？如何解决？

  > 首先，非静态的内部类、匿名内部类、局部内部类都会隐式的持有其外部类的引用。也就是说在 Activity 中创建的 Handler 会因此持有 Activity 的引用。
  >
  > 当我们在主线程使用 Handler 的时候，Handler 会默认绑定这个线程的 Looper 对象，并关联其 MessageQueue，Handler 发出的所有消息都会加入到这个 MessageQueue 中。Looper 对象的生命周期贯穿整个主线程的生命周期，所以当 Looper 对象中的 MessageQueue 里还有未处理完的 Message 时，因为每个 Message 都持有 Handler 的引用，所以 Handler 无法被回收，自然其持有引用的外部类 Activity 也无法回收，造成泄漏。
  >
  > ##### 使用静态内部类 + 弱引用的方式

- 如何让在子线程中弹出toast

  > 调用`Looper.prepare`以及`Looper.loop()`,但是切记线程任务执行完，需要手动调用`Looper.quitSafely()`否则线程不会结束。
