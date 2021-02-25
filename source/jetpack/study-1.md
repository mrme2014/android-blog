---
title: 走进Jetpack架构组件库
---

<!--more-->

## 目录

- 什么是Jetpack
- Jetpack的优势
- Jetpack组件库众多组件介绍
- 使用Jetpack架构开发模式



### 什么是Jetpack

- 面试概括：Jetpack是众多优秀组件的集合。是谷歌推出的一套引领Android开发者逐渐统一开发规范的架构
- 系统介绍：



### Jetpack的优势

- 基于生命周期感知的能力,可以减少NPE崩溃、内存泄漏、模板代码。为我们开发出健壮且高质量的程序提供强力保障。
- 组件可以单独使用，也可以搭配使用，搭配 Kotlin 语言特性可进一步加速开发。



### Jetpack组件库介绍

![Jetpack核心组件](/imgs/jetpack/Jetpack核心组件.png)

- ##### Navigation:为单Activity架构而生的端内路由

  - 特性：Activity,Fragment,Dialog提供路由能力的组件、导航时可携带参数、指定转场动画、支持deepline页面直达、fragment回退栈管理能力。
  - 不足之处：十分依赖xml文件，不利于模块化，组件化开发。

  - 添加依赖

  ```groovy
  api "android.arch.navigation:navigation-fragment:1.0.0"
  api "android.arch.navigation:navigation-ui:1.0.0"
  ```

  - 路由跳转,可携带参数,指定转场动画

  ```java
  NavController navController;
  navController.navigate(int resId,Bundle args, NavOptions navOptions) 
  ```

  - deepLink实现页面直达能力

  ```java
  navController.handleDeepLink("/detail/main")
  ```

  - 管理Fragment回退栈

  ```java
  navController.popBackStack(int destinationId, boolean inclusive)
  ```

  

  

- ##### 2.Lifecycle:具备宿主声明后期感知能力的组件。

  - 特性：它持有组件（如 Activity 或 Fragment）生命周期状态的信息，并且允许其他对象观察此状态。

  - 添加依赖

  ```groovy
  api 'androidx.appcompat:appcompat:1.1.0'
  api 'androidx.lifecycler:lifecycle-common:2.1.0'
  ```

  - 经典用法

  ```java
  public class Fragment implements xxx, LifecycleOwner {
  LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);
      @Override
      public Lifecycle getLifecycle() {
          return mLifecycleRegistry;
      }
  }
  
  class MyPresenter extends LifecycleObserver{
      @OnLifecycleEvent(Lifecycle.Event.ON_CREATE)
      void onCreate(@NotNull LifecycleOwner owner){}
  
      @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
      void onDestroy(@NotNull LifecycleOwner owner){
        
      }
  }
  
  getLifecycle().addObserver(mPresenter);//注册观察者,观察宿主生命周期状态变化
  ```

- ##### 3.ViewModel:具备生命周期感知能力的数据存储组件。

  - 特性：页面配置更改数据不丢失、生命周期感应、数据共享

  ```groovy
  api 'androidx.appcompat:appcompat:1.1.0'
  api 'androidx.lifecycler:lifecycle-viewmodel:2.0.0'
  api 'androidx.lifecycle:lifecycle-livedata:2.0.0'
  ```

  - ViewModel的数据存储、生命周期感知

  ```java
  //1.创建MyViewModel 类
  class MyViewModel extends ViewModel{
      @Override
      protected void onCleared() {
            super.onCleared();
            //宿主销毁时,回执行到这里,可自我清理释放资源
      }
      
     MutableLiveData<User> userInfoData = new MutableLiveData();
     public void getUserInfo(LifecycleOwner owner, Observer<User> observer){
           userInfoData.observe(owner,observer);
           //.....query user info from room database
           userInfoData.setValue(user);
       }
  }
  ```

  - ViewModel数据共享

  ```java
  //2.构建ViewModel实例对象，需要使用ViewModelProvider,来获取viewmodel对象。
  //如果ViewModelStore中已存在则返回共用实例,如不存在，则使用factory创建并缓存。
  
  //不同fragment中获取同一个ViewModel实例，从而实现数据共享
  class FragmentA extends Fragment{
   public void onCreate(Bundle bundle){
   MyViewModel myViewModel= new  ViewModelProvider(getActivity().getViewModelStore(),
                  new ViewModelProvider.NewInstanceFactory())
                  .get(MyViewModel.class)
    }
  }
  ```

  

- ##### 4.LiveData: 具备生命周期感知能力的数据订阅、分发组件。

  - 特性：支持共享资源、支持黏性事件的分发、不再需要手动处理生命周期、确保界面符合数据状态。
  - 不足：黏性事件不支持取消

  ```java 
  MutableLiveData<T> liveData = new MutableLiveData();
  //注册一个跟宿主生命周期绑定的观察者，宿主销毁了，会自动解除注册
  observe(LifecycleOwner owner,Observer<? super T> observer)
  //观察不到宿主生命周期，不会自动解除注册  
  observeForever(Observer<? super T> observer) 
  
  //以下俩方法都是分发数据给所有的观察者
  //只能用在主线程  
  setValue(T value) 
  //子线程，主线程都可以使用  
  postValue(T value)
  ```

- ##### 5.Room：轻量级orm数据库,本质上是一个SQLite抽象层。

  - 特性：使用更加简单（类似于Retrofit库），通过注解的方式实现相关功能。 编译时自动生成实现类impl

  - 引入依赖

  ```java
  api "android.arch.persistence.room:runtime:1.1.1"
  kapt "android.arch.persistence.room:compiler:1.1.1" 
  ```

  - Room数据库读写

  ```java
  //1. 创建一个操作数据库数据的实体层
  @Dao
  public interface UserDao {
      @Query("SELECT * FROM user")
      List<User> getAll();
  
      @update("update table user set desc=:desc and where user_id = :userId")
      User updateUser(String desc,String userId);
  
      @Insert
      void insertAll(User... users);
  
      @Delete
      void delete(User user);
  }
  ```

  ```java
  //2. 创建数据库
  @Database(entities = {User.class}, version = 1)
  public abstract class HiDatabase extends RoomDatabase {
      public static HiDatabase hiAppDb;
      static{
          hiAppDb = Room.databaseBuilder(getApplicationContext(),
          AppDatabase.class, "database-name").build();
      }
    
      public abstract UserDao userDao();
  }
  
  //3. 通过数据库单例对象,获取userDao数据操作对象.访问数据库
  hiAppDb.userDao().getAll();
  ```

- ##### 6.DataBinding: dataBinding只是一种工具，它解决的是View和数据之间的双向绑定。MVVM是一种架构模式，两者是有本质区别的。

  - 特性：数据与视图双向绑定、数据绑定空安全、减少模板代码、释放Activity/Fragment

  - 开启dataBinding

  ```groovy
  android{
      ...
      dataBinding {
          enabled = true
      }
  }
  ```

  - 布局中绑定数据

  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <layout xmlns:android="http://schemas.android.com/apk/res/android"
      <data>
          <variable
              name="user"
              type="com.mooc.ppjoke.model.User" />
          <import type="com.mooc.ppjoke.user.UserManager"></import>
      </data>
      <androidx.constraintlayout.widget.ConstraintLayout>
          <TextView
              android:id="@+id/tvName"
              android:layout_width="200dp"  //不能使用dataBinding动态绑定
              android:text="@{user.name}"  //单向绑定数据变更自动通知UI
              android:textSize="@{@dimen/16sp}"//资源引用
              android:text="@{user.name+@string/suffix}"  //字符串的拼接需要引用资源 
              android:text="@{UserManager.getUserName()}" //调用静态方法,类必须先导入
              android:onClick="@{()-> UserManager.login()}"
            />
  
        <EditText
              //双向绑定数据变更自动更新UI，UI变更了也能自动更新user中name的数据，比单向绑定多个=
              android:text="@={user.name}" />
      </androidx.constraintlayout.widget.ConstraintLayout>
  </layout>
  
  <layout_user_info_item>
  LayoutUserInfoItemBinding binding = LayoutUserInfoItemBinding.inflate(....)
  ```

  

- ##### 7.WorkManager:新一代后台任务管理组件,功能十分强悍。service能做的事情它都能做,

  - 支持周期性任务调度、链式任务调度、丰富的任务约束条件、即便程序即便退出，依旧能保证任务的执行

  - 添加依赖

  ```java
  api "android.arch.work:work-runtime:1.0.1"
  ```

  - 执行任务

  ```java
  //#1.构建任务
  class UploadFileWorker extends Worker{
     public Result doWork() {
       //执行文件上传的任务
       return Result.success()
     }
  }
  
  //#2.构建 执行任务的request对象
  OneTimeWorkRequest request = new OneTimeWorkRequest
        .Builder(UploadFileWorker.class)
        .build()
    
  //#3.把任务加入调度队列  
  WorkContinuation workContinuation = WorkManager.getInstance(context).beginWith(request); 
  workContinuation.then(workB).then(workC).enqueue()
  ```

- #####  8.Paging:列表分页组件，可以轻松完成分页预加载以达到无限滑动的效果

  - 巧妙融合LiveData、提供多种数据源加载方式

  - 不足之处，不支持列表数据增删改。列表添加HeaderView,FooterView定位不准确。

  - 添加依赖

  ```groovy
  api 'android.arch.paging:runtime:1.0.0'
  ```

  - 使用paging实现页面初始化与分页数据加载

  ```java
  //1.构建PagedList.Config对象,用以声明以何种方式分页
  PagedList.Config config = new PagedList.Config.Builder()
                  .setPageSize(10)//指定每次分页加载的条目数量
                  .setInitialLoadSizeHint(12)//指定初始化数据加载的条目数量
                  .build();
  //2.创建数据源工厂类,用来创建数据提供者
  DataSource.Factory factory = new DataSource.Factory() {
          @NonNull
          @Override
          public DataSource create() {
             return new ItemKeyedDataSource();
          }
      };
  
  //3.构建一个能够触发加载页面初始化数据的LiveData对象,并且把上面创建的DataSource.Factory和PagedList.Config 传递进去
  LiveData<PagedList<T>> pageData = new LivePagedListBuilder(factory, config)
                  .build();//最后通过build方法 构建出LiveData对象。请注意它的 泛型是<PagedList<T>>
  
  //4.最后我们只需要拿到前面构建出来的LiveData对象注册一个Observer观察者,仅仅如此就可以触发页面初始化数据的加载了
  mViewModel.getPageData().observe(this, pagedList -> submitList(pagedList));
  
  class MyItemKeyedDataSource extends ItemKeyedDataSource<GoodsModel>{
    public abstract void loadInitial(LoadInitialParams<Key> params,LoadInitialCallback<Value> callback){
       //页面初始化数据加载
       callback.onResult(list)
    }
   public void loadAfter(@NonNull LoadParams<Key> params,LoadCallback<Value> callback){
     //向后分页数据加载...
      callback.onResult(list)
   }
   public void loadBefore(LoadParams<Key> params,LoadCallback<Value> callback){
     //向前分页数据加载...
     callback.onResult(list)
   }
  }
  ```

  

### 使用Jetpack架构开发模式

<img src="/imgs/jetpack/arch.png" alt="arch" style="zoom: 87%;" />
