---
title: 走进Android线程世界
---

<!--more-->

## 目录

- 线程与进程
- 线程的几种创建方式
- 线程的优先级
- 线程的几种状态与常用方法
- 线程间消息通讯
- 线程安全

<img src="/imgs/thread/thread_introduction_png.png" />

### 线程与进程

- 一个进程至少一个线程
- 进程可以包含多个线程
- 进程在执行过程中拥有独立的内存空间，而线程运行在进程内

### 线程的几种创建方式

- new Thread：可复写 Thread#run 方法。也可传递 Runnable 对象，更加灵活。
- 缺点:缺乏统一管理，可能无限制新建线程，相互之间竞争，及可能占用过多系统资源导致死机或 oom

```java
//传递Runnable对象
1.new Thread(new Runnable(){
  .....
}).start()

//复写Thread#run方法
2.class MyThread extends Thread{
  public void run(){
    ....
  }
}
new MyThread().start()
```

<br/>

- AysncTask,轻量级的异步任务工具类,提供任务执行的进度回调给 UI 线程
- 场景：需要知晓任务执行的进度,多个任务串行执行
- 缺点：生命周期和宿主的生命周期不同步,有可能发生内存泄漏,默认情况所有任务串行执行

```java
class MyAsyncTask extends AsyncTask<String, Integer, String> {
            private static final String TAG = "MyAsyncTask";
            @Override
            protected String doInBackground(String... params) {
                for (int i = 0; i < 10; i++) {
                    publishProgress(i * 10);
                }
                return params[0];
            }
            @Override
            protected void onPostExecute(String result) {
                Log.e(TAG, "result: " + result);
            }
            @Override
            protected void onProgressUpdate(Integer... values) {
                Log.e(TAG, "onProgressUpdate: " + values[0].intValue());
            }
        }
// #1 子类复写方法
AsyncTask asyncTask = new MyAsyncTask();
//AsyncTask所有任务默认串行执行
asyncTask.execute("execute MyAsyncTask");
   or
asyncTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR,"execute MyAsyncTask")

// #2 使用#execute方法，同样串行执行
AsyncTask.execute(new Runnable() {
          @Override
          public void run() {
           ......
       }
  });

// #3 使用内置THREAD_POOL_EXECUTOR线程池 并发执行
 AsyncTask.THREAD_POOL_EXECUTOR.execute(new Runnable() {
            @Override
            public void run() {

            }
        });
```

<br/>

- HandlerThread,适用于主线程需要和工作线程通信,适用于持续性任务,比如轮训的场景，所有任务串行执行
- 缺点:不会像普通线程一样主动销毁资源，会一直运行着，所以可能会造成内存泄漏

```java
HandlerThread thread = new HandlerThread("concurrent-thread");
thread.start();
ThreadHandler handler = new ThreadHandler(thread.getLooper()) {
      @Override
      public void handleMessage(@NonNull Message msg) {
                switch (msg.what) {
                    case MSG_WHAT_FLAG_1:
                        break;
                }
            }
        };
handler.sendEmptyMessage(MSG_WHAT_FLAG_1);
thread.quitSafely();

//定义成静态,防止内存泄漏
static class ThreadHandler extends Handler{
  public ThreadHandler(Looper looper){
    super(looper)
  }
}
```

<br/>

- IntentService,适用于我们的任务需要跨页面读取任务执行的进度，结果。比如后台上传图片，批量操作数据库等。任务执行完成功后，就会自我结束，所以不需要手动 stopservice,这是他跟 service 的区分

```java
class MyIntentService extends IntentService{
 @Override
 protected void onHandleIntent(@Nullable Intent intent) {
   int command = intent.getInt("command")
   ......                                                       }
}
context.startService(new Intent())
```

<br/>

- ✨ThreadPoolExecutor:适用快速处理大量耗时较短的任务场景

```java
 Executors.newCachedThreadPool();//线程可复用线程池
 Executors.newFixedThreadPool();//固定线程数量的线程池
 Executors.newScheduledThreadPool();//可指定定时任务的线程池
 Executors.newSingleThreadExecutor();//线程数量为1的线程池
```

### 线程的优先级

```java
public static void main(String[] args){
   Thread thread = new Thread();
   thread.start();
   int ui_proi = Process.getThreadPriority(0)
   int th_proi = thread.getPriority();

   //输出结果
   ui_proi =5
   th_proi=5
}
```

- 线程的优先级具有继承性，在某线程中创建的线程会继承此线程的优先级。那么我们在 UI 线程中创建了线程，则线程优先级是和 UI 线程优先级一样，平等的和 UI 线程抢占 CPU 时间片资源。

- JDK Api,限制了新设置的线程的优先级必须为[1~10],优先级 priority 的值越高，获取 CPU 时间片的概率越高。UI 线程优先级为 5

```java
java.lang.Thread.setPriority(int newPriority)
```

- Android Api, 可以为线程设置更加精细的优先级(-20~19).优先级 priority 的值越低，获取 CPU 时间片的概率越高。UI 线程优先级为-10

```java
android.os.Process.setThreadPriority(int newPriority)
```

### 线程的几种状态与常用方法

<table border="1">
  <tr bgcolor="#999999">
    <th>方法名</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>NEW</td>
    <td>初始状态,线程被新建,还没调用start方法  </td>
  </tr>
  <tr>
   <td>RUNNABLE</td>
   <td>运行状态,把"运行中"和"就绪"统称为运行状态</td>
  </tr>
  <tr>
   <td>BLOCKED</td>
   <td>阻塞状态，表示线程阻塞于锁</td>
  </tr>
   <tr>
   <td>TIME_WAITING</td>
   <td>超时等待状态，表示可以在指定的时间超时后自行返回</td>
  </tr>
   <tr>
   <td>TERMINATED</td>
   <td>终止状态，表示当前线程已经执行完毕</td>
  </tr>
  </table>

<img src="/imgs/thread/thread_status.png" />
<table border="1">
  <tr bgcolor="#999999">
    <th>方法名</th>
    <th>说明</th>
  </tr>
  <tr>
    <td>wait</td>
    <td>进入等待池,释放资源对象锁,可使用notify,notifyAll,或等待超时来唤醒 </td>
  </tr>
  <tr>
   <td>join</td>
   <td>等待目标线程执行完后在执行此线程</td>
  </tr>
  <tr>
   <td>yield</td>
   <td>暂停当前正在执行的线程对象，不会释放资源锁,使同优先级或更高优先级的线程有执行的机会</td>
  </tr>
   <tr>
   <td>sleep</td>
   <td>使调用线程进入休眠状态,但在一个synchronized块中执行sleep,线程虽然会休眠，但不会释放资源对象锁</td>
  </tr>
  </table>
### 线程间消息通讯

- 主线程向子线程发送消息

```java
class LooperThread extends Thread {
            private Looper looper;

            public Looper getLooper() {
                synchronized (this){
                    if (looper == null && isAlive()) {
                        try {
                            wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
                return looper;
            }

            @Override
            public void run() {
                Looper.prepare();
                synchronized (this) {
                    looper = Looper.myLooper();
                    notifyAll();
                }
                Looper.loop();
            }

            public void quit() {
                looper.quit();
            }
        }
LooperThread looperThread = new LooperThread();
looperThread.start();
Handler handler = new Handler(looperThread.getLooper()) {
       @Override
       public void handleMessage(@NonNull Message msg) {
       Log.e(TAG, "handleMessage: " + msg.what);
            }
        };
handler.sendEmptyMessage(MSG_WHAT_FLAG_1);
```
