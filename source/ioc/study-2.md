---
title: 见微知著从源码到原理剖析Hilt核心知识点
---

<!--more-->

## 目录
- 案例演示
- 字节码分析
- 对象生命周期和作用域的实现

### 案例演示 

- 在Activity中使用Hilt 动态注入ILoginService的实现类对象

```kotlin
@AndroidEntryPoint
class MainActivity:AppcomponentActivity{
   @Inject
   lateinit var iLoginService:ILoginService
   fun onCreate(){
       iLoginService.login()
   }
}

@Module
@InstallIn(ApplicationComponent::class)//以下方法创建的对象生命周期跟applicaiton一样长
abstract class MainModule {

    @Binds
    @Singleton//该实例对象在applicaiton的生命周期内唯一
              //多次申请绑定，使用的是同一个对象
    abstract fun bindService(impl: LoginServiceImpl): ILoginService
}
    //@Providers
    //fun bindService():ILoginService{
    //由于构造函数需要context参数，在当前类无法拿到context,所以只能使用binds方式，提供对象实例
    //  return LoginServiceImpl(context)
    //}

interface ILoginService {
    fun login()
}


class LoginServiceImpl @Inject constructor(@ApplicationContext val context: Context) :
    ILoginService {
    override fun login() {
        Toast.makeText(context, "LoginServiceImpl:login", Toast.LENGTH_SHORT)
            .show()
    }
}
```



### 字节码分析

- Application对象注入

```kotlin
//1. 编译时生成Hilt_HiApplication,称之为依赖注入的入口类
     //负责创建applicationComponet组件对象
//2. 编译时把父类替换成Hilt_HiApplication
@HiltAndroidApp
class HiApplication :(Hilt_HiApplication) Application {
    override fun onCreate() {
        super.onCreate()-->//3.执行父类Hilt_HiApplication.onCreate() 
       //4.创建与Application生命周期关联的HiltComponents_ApplicationC组件,
             //并把application的Context对象与之关联
       //5.调用injectHiApplication为Application注入对象
    }
}
```

- MainActivity对象注入

```kotlin
//1. 编译时生成Hilt_MainActivity,称之为对象注入的入口类，
      //创建ActivityComponent对象
//2. 编译时把父类替换成Hilt_MainActivity
@AndroidEntryPoint
class MainActivity :(Hilt_MainActivity) AppcomponentActivity {
    override fun onCreate() {
        super.onCreate()-->//3.执行父类Hilt_MainActivity.onCreate() 
       //4.创建与Activity生命周期关联的HiltComponents_ActivityC组件
       //5.调用injectHiMainActivity为MainActivity注入对象
    }
}
```

- 共同点

  - 都会创建以`Hilt`为前缀，类名（`HiAppilication`）为后缀的对象注入管理类

  - 都会被替换掉父类为 `Hilt_HiApplication` ,`Hilt_MainActivity`

  - 都会在onCreate()时创建对应的component组件`HiltComponents_ApplicationC`,`HiltComponents_ActivityC`

  - 都会调用`injectxxx(MainActivity instance)`开始注入对象

    
  <img src="/imgs/ioc/components.png" />


```kotlin
public final class HiApplication_HiltComponents {
//ApplicationComponent，每一个组件关联多个module.
//根据module的InstallIn(ApplicationComponent::class)注解来决定的
@Component(
      modules = {
          ApplicationContextModule.class,
          ActivityRetainedCBuilderModule.class,
          ServiceCBuilderModule.class,
          MainModule.class
      }
  )
  @Singleton
  public abstract static class ApplicationC implements ApplicationComponent,HiApplication_GeneratedInjector {
    //void injectHiApplication(HiApplication hiApplication);
  }

//ActivityComponent
@Subcomponent(
      modules = {
          DefaultViewModelFactories.ActivityModule.class,
          FragmentCBuilderModule.class,
          ViewCBuilderModule.class,
          HiltWrapper_ActivityModule.class,
          ViewModelFactoryModules.ActivityModule.class
      }
  )
  @ActivityScoped
  public abstract static class ActivityC implements ActivityComponent,MainActivity_GeneratedInjector {
    @Subcomponent.Builder
    abstract interface Builder extends ActivityComponentBuilder {
    }
  }
}
```

- 组件间关系
  - 组件间存在嵌套关系，生命周期长的组件称之为生命周期短的组件的父容器，其中ApplicationC作为顶层容器
  - 目的是可以方便的从父容器获取实例对象(比如FragmentCImpl可以直接从ActivityCImpl中获取单例对象，实现共享)

<img src="/imgs/ioc/组件间关系.jpeg" alt="组件间关系" style="zoom:50%;" />



- 组件中的对象生命周期和作用域

```kotlin
public final class DaggerHiApplication_HiltComponents_ApplicationC extends HiApplication_HiltComponents.ApplicationC {
  private final ApplicationContextModule applicationContextModule;
  //MemoizedSentinel无意义，就是用于判断是否已创建过iLoginService的真实实例对象
  private volatile Object iLoginService = new MemoizedSentinel();

  private LoginServiceImpl getLoginServiceImpl() {
    return new LoginServiceImpl(ApplicationContextModule_ProvideContextFactory.provideContext(applicationContextModule));
  }

  private ILoginService getILoginService() {
    Object local = iLoginService;
    //1.如果在module的方法上标记了scope,那么在创建对象时，会首先判断
    //2.是否还没创建过了，那就创建它实例对象，并把applicationContext作为参数传递进去
    //3.如果已经创建过了，那就直接返回，就达到了组件生命周期内单一实例的目的
    //4. 由于组件（component）是在android类(application,activity,fragment)中开始被创建的。所以组件的对象的销毁，自然也和他们关联了起来。
    if (local instanceof MemoizedSentinel) {
      synchronized (local) {
        local = iLoginService;
        if (local instanceof MemoizedSentinel) {
          local = getLoginServiceImpl();
          iLoginService = DoubleCheck.reentrantCheck(iLoginService, local);
        }
      }
    }
    return (ILoginService) local;
  }
```

- 获取实例对象，并注入

```kotlin
private final class ActivityCImpl extends HiApplication_HiltComponents.ActivityC {
      //因为ActivityC实现了MainActivity_GeneratedInjector
      //所以需要复写injectMainActivity方法
      @Override
      public void injectMainActivity(MainActivity mainActivity) {
        injectMainActivity2(mainActivity);
      }
      private MainActivity injectMainActivity2(MainActivity instance) {
        MainActivity_MembersInjector.injectLoginService(instance, DaggerHiApplication_HiltComponents_ApplicationC.this.getILoginService());
        return instance;
      }
```

```kotlin
public final class MainActivity_MembersInjector implements MembersInjector<MainActivity> {
  public static void injectLoginService(MainActivity instance, ILoginService loginService) {
    //最后在这里完成对象的注入
    //所以private类型的无法完成对象注入，因为访问不到
    instance.loginService = loginService;
  }
}
```

- 能为主线程中的字段实现异步数据加载的依赖注入吗？

  ```kotlin
  class MainActivity{
      @Inject
      @jvmFiled
      var data:List<City>?=null
      
      //var data:LiveData<List<City>?>
  }
  
  @module
  @InstallIn(ActivityComponent::Class)
  object MainModule{
    @Providers
    fun loadData():List<City>?={
       HiExecutor.execute(Runnable{
           //加载数据库数据
       })
    }
    
    @Providers
    fun loadData():LiveData<List<City>?>={
       return LiveData<List<City>?>(){
         @override fun onActive(){
           HiExecutor.execute(Runnable{
                //加载数据库数据
                postValue(xxx)
            })
         }
       }
    }
  }
  ```

  
