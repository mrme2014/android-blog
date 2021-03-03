---
title: LiveData架构组件原理解析
---

<!--more-->

## 目录

## LiveData架构组件原理解析

- 什么是LivaData
- LiveData几种衍生类
- LiveData核心方法
- LiveData实现原理



- ### 什么是LiveData

  - LiveData组件是Jetpack新推出的基于观察者的消息订阅/分发组件，具有宿主(Activity、Fragment)生命周期感知能力，这种感知能力可确保 LiveData 仅分发消息给处于`活跃状态`的观察者，即只有处于`活跃状态`的观察者才能收到消息
  - LiveData的消息分发机制，是以往的Handler,EventBus、RxjavaBus无法比拟的，它们不会顾及当前页面是否可见,一股脑的有消息就转发。导致即便应用在后台,页面不可见,还在做一些无用的绘制，计算(细心的同学可以发现微信消息列表是在可见状态时才会更新列表最新信息的)

  > 活跃状态:Observer所在宿主处于started,resumed状态

```java
class MainActivity extends AppcompactActivity{
   public void onCreate(Bundle bundle){
     
        Handler handler = new Handler(){
           @Override
           public void handleMessage(@NonNull Message msg) {
                //无论页面可见不可见，都会去执行页面刷新,IO。更有甚者弹出对话框
           }
         };
        //1.无论当前页面是否可见,这条消息都会被分发。----消耗资源
        //2.无论当前宿主是否还存活,这条消息都会被分发。---内存泄漏
       handler.sendMessage(msg)

       
       liveData.observer(this,new Observer<User>){
            void onChanged(User user){
              
            }
        }
       //1.减少资源占用---          页面不可见时不会派发消息
       //2.确保页面始终保持最新状态---页面可见时,会立刻派发最新的一条消息给所有观察者--保证页面最新状态
       //3.不再需要手动处理生命周期---避免NPE
       //4.可以打造一款不用反注册,不会内存泄漏的消息总线---取代eventbus
       liveData.postValue(data);
   }
}
```





### LiveData的几种用法

- MutableLiveData

  我们在使用`LiveData`的做消息分发的时候，需要使用这个子类。之所以这么设计，是考虑到单一开闭原则，只有拿到`MutableLiveData`对象才可以发送消息，`LiveData`对象只能接收消息，避免拿到`LiveData`对象时既能发消息也能收消息的混乱使用。

```java
public class MutableLiveData<T> extends LiveData<T> {
    @Override
    public void postValue(T value) {
        super.postValue(value);
    }

    @Override
    public void setValue(T value) {
        super.setValue(value);
    }
}
```



- MediatorLiveData
  - 可以统一观察多个`LiveData`的发射的数据进行统一的处理
  - 同时也可以做为一个liveData，被其他Observer观察。

```java
//创建两个长得差不多的LiveData对象
LiveData<Integer> liveData1 =  new MutableLiveData();
LiveData<Integer> liveData2 = new MutableLiveData();

//再创建一个聚合类MediatorLiveData
MediatorLiveData<Integer> liveDataMerger = new MediatorLiveData<>();
//分别把上面创建LiveData 添加进来。
liveDataMerger.addSource(liveData1, observer);
liveDataMerger.addSource(liveData2, observer);

Observer observer = new Observer<Integer>() {
 @Override
 public void onChanged(@Nullable Integer s) {
      titleTextView.setText(s);
 }
//一旦liveData或liveData发送了新的数据 ，observer便能观察的到，以便统一处理更新UI
```



- Transformations.map  操作符

  可以对livedata的进行变化，并且返回一个新的livedata对象

```java
MutableLiveData<Integer> data = new MutableLiveData<>();
  
//数据转换
LiveData<String> transformData = Transformations.map(data, input -> String.valueOf(input));
//使用转换后生成的transformData去观察数据
transformData.observe( this, output -> {

});

//使用原始的livedata发送数据
data.setValue(10);
```



### LiveData核心方法

| 方法名                                          | 作用                                              |
| ----------------------------------------------- | ------------------------------------------------- |
| observe(LifecycleOwner owner,Observer observer) | 注册和宿主生命周期关联的观察者                    |
| observeForever(Observer observer)               | 注册观察者,不会反注册,需自行维护                  |
| setValue(T data)                                | 发送数据,没有活跃的观察者时不分发。只能在主线程。 |
| postValue(T data)                               | 和setValue一样。不受线程环境限制,                 |
| onActive                                        | 当且仅当有一个活跃的观察者时会触发                |
| inActive                                        | 不存在活跃的观察者时会触发                        |
|                                                 |                                                   |

### LiveData实现原理

- 黏性消息分发流程。即新注册的observer也能接收到前面发送的最后一条数据

 <img src="/imgs/jetpack/livedata_sticky.png" />

- 普通消息分发流程。即调用postValue，setValue才会触发消息的分发

<img src="/imgs/jetpack/livedata_sendvalue.png" alt="livedata_sendvalue" style="zoom:50%;" />
