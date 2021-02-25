---
title: Lifecycle架构组件原理解析
---

<!--more-->

## 目录

- 什么是Lifecycle
- 如何使用Lifecycle观察宿主状态
- Fragment是如何实现Lifecycle的 
- Activity是如何实现Lifecycle的
- Lifecycle是如何分发宿状态的



### 什么是Lifecycle

- 具备宿主声明后期感知能力的组件。它能持有组件（如 Activity 或 Fragment）生命周期状态的信息，并且允许其他观察者监听宿主的状态



### Lifecycle怎么使用

```java
//1. 自定义的LifecycleObserver观察者，用注解声明每个方法观察的宿主的状态
class LocationObserver extends LifecycleObserver{
    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    void onStart(@NotNull LifecycleOwner owner){
      //开启定位
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    void onStop(@NotNull LifecycleOwner owner){
       //停止定位
    }
}

//2. 注册观察者,观察宿主生命周期状态变化
class MyFragment extends Fragment{
  public void onCreate(Bundle bundle){
    
    MyLifecycleObserver observer =new MyLifecycleObserver()
    getLifecycle().addObserver(observer);
    
  }
}
```

- FullLifecyclerObserver

```java
interface FullLifecycleObserver extends LifecycleObserver {
    void onCreate(LifecycleOwner owner);
    void onStart(LifecycleOwner owner);
    void onResume(LifecycleOwner owner);
    void onPause(LifecycleOwner owner);
    void onStop(LifecycleOwner owner);
    void onDestroy(LifecycleOwner owner);
}
class LocationObserver extends FullLifecycleObserver{
    void onStart(LifecycleOwner owner){}
    void onStop(LifecycleOwner owner){}
}

```

- LifecycleEventObserver

```java
public interface LifecycleEventObserver extends LifecycleObserver {
    void onStateChanged(LifecycleOwner source, Lifecycle.Event event);
}

class LocationObserver extends LifecycleEventObserver{
    @override
    void onStateChanged(LifecycleOwner source, Lifecycle.Event event){
      //需要自行判断life-event是onstart, 还是onstop
    }
}
```



- Fragment实现Lifecycle

```java
public class Fragment implements LifecycleOwner {
LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
  @Override
  public Lifecycle getLifecycle() {  
      //复写自LifecycleOwner,所以必须new LifecycleRegistry对象返回
      return mLifecycleRegistry;
  }
  
void performCreate(){
     mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
  }
  
 void performStart(){
     mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START);
  }
  .....
 void performResume(){
     mLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_RESUME);
  }  
}
```

- LifecycleOwner、Lifecycle、LifecycleRegistry的关系

![lifecycle3](/imgs/jetpack/lifecycle3.png)





- Activity实现Lifecycle

```java
public class ComponentActivity extends Activity implements LifecycleOwner{
  private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
   @NonNull
   @Override
   public Lifecycle getLifecycle() {
      return mLifecycleRegistry;
   }
  
  protected void onCreate(Bundle bundle) {
      super.onCreate(savedInstanceState);
      //往Activity上添加一个fragment,用以报告生命周期的变化
      //目的是为了兼顾不是继承自AppCompactActivity的场景.
      ReportFragment.injectIfNeededIn(this); 
}
```

- ReportFragment核心源码

```java
public class ReportFragment extends Fragment{
  public static void injectIfNeededIn(Activity activity) {
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            manager.executePendingTransactions();
        }
  }

    @Override
    public void onStart() {
        super.onStart();
        dispatch(Lifecycle.Event.ON_START);
    }

    @Override
    public void onResume() {
        super.onResume();
        dispatch(Lifecycle.Event.ON_RESUME);
    }

    @Override
    public void onPause() {
        super.onPause();
        dispatch(Lifecycle.Event.ON_PAUSE);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        dispatch(Lifecycle.Event.ON_DESTROY);
    }

    private void dispatch(Lifecycle.Event event) {
         Lifecycle lifecycle = activity.getLifecycle();
         if (lifecycle instanceof LifecycleRegistry) {
             ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
         }
}
```

- 宿主生命周期与宿主状态模型图

<img src="/imgs/jetpack/lifecycle_state.png" alt="lifecycle_state" style="zoom:50%;" />

- 添加observer时,完整的生命周期事件分发

![lifecycle4](/imgs/jetpack/lifecycle4.png)

- 作业:基于Lifecycle实现APP前后台切换事件观察的能力

```java
class AppLifecycleOwner extends LifecycleOwner{
  LifecycleRegistry registry = new LifecycleRegistry(this)
  @ovrride
  Lifecycle getLifecycle(){
    return  registry
  }
  
  void init(Application application){
    //利用application的  ActivityLifecycleCallbacks 去监听每一个Activity的onstart,onStop事件。
    //计算出可见的Activity数量，从而计算出当前处于前台还是后台。然后分发给每个观察者
  }
}
```

