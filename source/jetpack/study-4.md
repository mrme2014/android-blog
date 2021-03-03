---
title: ViewModel架构组件原理解析
---

<!--more-->

## 目录

## ViewModel架构组件原理解析

- 什么是ViewModel
- ViewModel的用法
- ViewModel复用实现原理
- ViewModel数据复用进阶SavedState



### 什么是ViewModel

- **具备宿主生命后期感知能力的数据存储组件**。
- **ViewModel保存的数据，在页面因**<font color='red'>**配置变更导致页面销毁重建**</font>**之后依然也是存在的**

> 配置变更：横竖屏切换、分辨率调整、权限变更、系统字体样式变更...

### ViewModel的用法

- **常规用法**:存储的数据,仅仅只能当页面因为配置变更导致的销毁再重建时可复用。复用的是ViewModel的实例对象整体

```kotlin
class HiViewModel() : ViewModel() {
    val liveData = MutableLiveData<List<GoodsModel>>()
    fun loadInitData():LiveData<List<GoodsModel>> {
        //from remote
        //为了适配因配置变更而导致的页面重建, 重复利用之前的数据,加快新页面渲染，不再请求接口
        if(liveData.value==null){
           val remoteData = fetchDataFromRemote()
           liveData.postValue(remoteData)
        }
        return liveData
    }
}

//通过ViewModelProvider来获取viewmodel对象
val viewModel = ViewModelProvider(this).get(HiViewModel::class.java)

viewModel.loadPageData().observer(this,Observer{
     //渲染列表
})
```

- **进阶用法**:存储的数据,无论是配置变更、还是因内存不足、电量不足等系统原因导致页面被回收再重建。都可以复用。即便ViewModel不是同一个实例，它存储的数据也能做到复用
- 如果页面被正常关闭，这里的数据会被正常清理释放
- 需要主动引入依赖`savedstate`组件依赖

```groovy
api 'androidx.lifecycle:lifecycle-viewmodel-savedstate:2.2.0'
```

```kotlin
class HiViewModel(val savedState: SavedStateHandle) : ViewModel() {
    private val KEY_HOME_PAGE_DATA="key_home_page_data"
    private val liveData = MutableLiveData<List<GoodsModel>>()
    fun loadInitData():LiveData<List<GoodsModel>> {
        //1.from memory .
        if(liveData.value==null){
          val memoryData =savedState.get<List<GoodsModel>>(KEY_HOME_PAGE_DATA)
          liveData.postValue(memoryData)
          return liveData
      
        //2.from remote
          val remoteData = fetchDataFromRemote()
          savedState.set(KEY_HOME_PAGE_DATA,remoteData)
          liveData.postValue(remoteData)
          
        }
      return liveData
    }
}
```

- 跨页面的数据共享

```kotlin
//让Application实现ViewModelStoreOwner 接口
class MyApp: Application(), ViewModelStoreOwner {
    private val appViewModelStore: ViewModelStore by lazy {
        ViewModelStore()
    }

    override fun getViewModelStore(): ViewModelStore {
        return appViewModelStore
    }
}

val viewmodel = ViewProvider(application).get(HiViewModel::class.java)
```



### 配置变更ViewModel复用实现原理

- 准确点来说,应该是ViewModel如何做到在宿主销毁了，还能继续存在. 以至于页面恢复重建后，还能接着复用。<font color ='red'>**【肯定是前后获取到的是同一个ViewModel实例对象】**</font>
- 获取ViewModel实例的过程

```kotlin
val viewmodel = ViewModelProvider(activity).get(HiViewModel::class.java)
```

<img src="/imgs/jetpack/viewmodel_provider.png" />



- ViewModelProvider。本质是从传递进去的ViewModelStore来获取实例,如果没有，则利用factory去创建一个新的，并存储到ViewModelStore。

```java
class ViewModelProvider{
  private static final String DEFAULT_KEY =
            "androidx.lifecycle.ViewModelProvider.DefaultKey";

//根据传递的modelClass 构建一个默认的Key
public <T extends ViewModel> T get(Class<T> modelClass) {
        String canonicalName = modelClass.getCanonicalName();
        return get(DEFAULT_KEY + ":" + canonicalName, modelClass);
    }

//获取viewmodel实例时，也可以自行指定Key
public <T extends ViewModel> T get(String key, Class<T> modelClass) {
        ViewModel viewModel = mViewModelStore.get(key);
        if (mFactory instanceof KeyedFactory) {
            viewModel = ((KeyedFactory) (mFactory)).create(key, modelClass);
        } else {
            viewModel = (mFactory).create(modelClass);
        }
        mViewModelStore.put(key, viewModel);
        return (T) viewModel;
    }
}
```

- ViewModelStore 一个真正用来存储ViewModel实例的集合。本质上是HashMap<String,ViewModel>

```java
class ComponentActivity extends ViewModelStoreOwner{
    static final class NonConfigurationInstances {
        Object custom;
        ViewModelStore viewModelStore;
    }
    public ViewModelStore getViewModelStore() {
        if (mViewModelStore == null) {
            NonConfigurationInstances nc =
                    (NonConfigurationInstances) getLastNonConfigurationInstance();
            if (nc != null) {
                mViewModelStore = nc.viewModelStore;
            }
            if (mViewModelStore == null) {
                mViewModelStore = new ViewModelStore();
            }
        }
        
        return mViewModelStore;
    }
   }
}
```

- 因系统原因页面被回收时,会触发该方法, viewModelStore对象会被存储在ActivityThread中。在页面恢复重建时,会再次把这个object对象传递到Activity中。

```java
public final Object onRetainNonConfigurationInstance() {
        Object custom = onRetainCustomNonConfigurationInstance();
        
        NonConfigurationInstances nci = new NonConfigurationInstances();
        nci.custom = custom;
        nci.viewModelStore = viewModelStore;
        return nci;
    }
```

### ViewModel数据复用进阶SavedState

- SavedStateHandle的数据存储与恢复。即便ViewModel不是同一个实例，它存储的数据也能做到复用
- SavedStateRegistryController:用于创建SavedStatedRegistry
- SavedStatedRegistry数据存储、恢复中心。
- SavedStateHandle：单个ViewModel数据存储 、恢复。

<img src="/imgs/jetpack/savedstatehandle.png" />

</br>

- SavedStateRegistry 模型。一个总Bundle,key-value存储着每个ViewModel对应子bundle

<img src="/imgs/jetpack/savedStateRegistry.png" />

</br>

- SavedState数据存储流程.逐一调用每个SavedStateHandle保存自己的数据。汇总成一个总的Bundle,再存储到Activity的savedState对象中。

<img src="/imgs/jetpack/savedstate_save.png" />

</br>

- SavedState数据复用流程(1),从Activity的saveState恢复所有ViewModel的数据到SavedStateRegistry

<img src="/imgs/jetpack/savedstate_restore.png" />

</br>

- SavedState数据复用流程(2)，创建ViewModel并传递恢复的SavedStateHandle

<img src="/imgs/jetpack/savedstate_create.png" />



### 总结

- ViewModel`和`onSaveIntanceState方法有什么区别

  - onSaveIntanceState只能存储轻量级的key-value键值对数据, 非配置变更导致的页面被回收时才会触发，此时数据存储在ActivityRecord中
  - viewmodel可以存放任意Object数据。因配置变更导致的页面被回收才有效。此时存在ActivityThread#ActivityClientRecord中。

- SavedState本质是利用了onSaveIntanceState的时机。每个ViewModel的数据单独存储在一个bundle中，再合并成一个整体。再存放在Activity的outBundle中。

  
