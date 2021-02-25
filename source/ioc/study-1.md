---
title: 走进IOC架构世界
---

<!--more-->

## 目录
- 为什么需要IOC
- 什么是IOC & DI
- Hilt 大法
- Hilt VS dagger2

  

- #### 为什么需要IOC

```java
//模板代码，食之无味 弃之可惜，
public class MainActivity extends Activity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
     super.onCreate(savedInstanceState);
     TextView mTextView=(TextView) findViewById(R.id.mTextView);
     mTextView.setOnClickListener(new OnClickListener() {
        @Override
        public void onClick(View v) {

        }
     });
    
     String params1= getIntent().getString("params1")
     String params2= getIntent().getString("params2")
  }
}
```

- ButterKnife View注入，Arouter Intent参数自动提取注入

```java
public class MainActivity extends Activity {
 @bindView(R.id.text)
 public TextView textView;
 
 @Autowired
 public GoodsModel goodsModel
 @Override
 protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
     ButterKnife.inject(this);//运行时view注入
     Arouter.getInstance().inject(this)//运行时参数注入
  }
  
 @Click(R.id.text)
 void buttonClick() {
    
  }
}
```

- #### 两者实现原理简述. 

  编译时按照命名规则生成相应实现类，编织好`findViewById`的代码.

  运行时根据`MainActivity`的类名找到编译生成的实现类，反射构建实例，并调用inject方法触发`findViewById`

```java
//ButterKnife自动findViewById
class ButterKnife_MainActivity<MainActivity> implements Inject{
   @Override
   public void inject(MainActivity activity){
      activity.textView =activity.findViewById(R.id.text)
   }
}
```

```java
//arouter的参数自动提取
public class DetailActivity$$ARouter$$Autowired implements ISyringe {
  @Override
  public void inject(Object target) {
    DetailActivity substitute = (DetailActivity)target;
    substitute.goodsId = substitute.getIntent().getStringExtra("goodsId");
    substitute.goodsModel = substitute.getIntent().getParcelableExtra("goodsModel");
  }
}
```

- 对象注入

```java
//dagger2 hilt对象注入,不需要再new了，约定好规则，框架自动实现对象创建和赋值
@AndroidEntryPoint
public class MainActivity extends Activity {
 @Inject
 public EmptyView emptyView 
 @inject
 public User user          
 public void onCreate(Bundle savedInstance){

   }
}
```

- #### IOC框架的类型

  - **view注入:** Xutils，Android Annotations ,ButterKnife

  - **参数注入:** Arouter

  - **对象注入:** [koin](https://start.insert-koin.io/#/quickstart/kotlin),Dagger2，[<font color='red'>**Hilt**</font>](https://developer.android.com/training/dependency-injection/hilt-android)

- #### 什么是IOC(Inversion of Control)控制反转？

  它是一种思想不是一个技术实现。描述的是：Java 开发领域对象的创建以及管理的问题。

  - **控制** ：指的是对象创建（实例化、管理）的权力

  - **反转** ：控制权交给外部环境（Spring 框架、IoC 容器）

  - **传统的开发方式** ：一个类里面需要用到很多个成员变量，传统的写法，这些成员变量，都需要new出来！

  - **使用 IoC 思想的开发方式** ：**IOC的原则是：NO，我们不要new，这样耦合度太高(入参改变，所有引用都要改)**，而是通过 IOC 容器(Hilt,Dagger2 框架) 来帮助我们实例化对象并赋值。

  - **解决方案一：**配置xml文件，里面标明哪个类，里面用了哪些成员变量，等待加载这个类的时候，我帮你注入（new）进去(Spring 服务器开发常用ioc方案)

  - **解决方案二：**用注解在需要注入的成员变量上面加个注解，例如`@Inject`(android上常用的ioc实现方案,annotation+abstractProcessor)编译时生成相关实现类，运行时动态注入。

- #### 什么是DI（Dependency Injection）依赖注入?

  其实它们是同一个概念的不同角度描述，IOC是一种软件设计思想，DI是这种软件设计思想的一个具体的实现，相比于控制反转，依赖注入更容易理解。

  依赖注入指的是对象是通过外部注入的方式完成创建。

  　　

- #### IOC的优势

  - 对象之间的耦合度或者说依赖程度降低；
  - 对象实例的创建变的容易管理，很容易就可以实现一个全局，或局部共享单例
  - **场景：模板代码创建实例对象**，**全局或局部对象共享**

```kotlin
//1. 传统做法需要ILoginService = new LoginServiceImpl(),增加了对LoginServiceImpl的引用，
//倘若那天LoginServiceImpl不再是唯一实现类，那么所有引用的地方都需要相应改变

//2. 如果有100个地方都直接new创建LoginServiceImpl对象，
//如果入参改变了,那么这100个地方都需要相应改变。

//3.使用DI则没有这个问题,因为对象的创建将被统一管理了，统一实现了。
class MainActivty:AppcompatActivity{
  @Inject public ILoginService loginService;
}

class LoginServiceImpl(): ILoginService{
}

  

class CustomFragment:Fragment{
  //假如fragment需要访问Activity中的成员变量
  //共享对象的自动注入，在同一个作用域内(activity生命周期内，application生命周期内)，得到的都是同一个实例对象，不需要传来传去
   @Inject user:User
   fun  display(){
      //传统做法  ×强转×，或者set
      ((MainActivity)context).user.name
      ((SecondActivity)context).user.name
   }
}
```

- #### IOC的缺点

  - 代码可读性差，不知道对象在哪里被创建？实例创建的时机在哪里？入参是什么？
  - 增加新人学习成本
  - 加速触及65535方法数

- #### 是否有必要引入IOC？

  不是必须。能发挥多大作用取决于你的项目体量复杂度，如果简单的小项目用它适得其反，学习成本和收益不匹配.

- #### Hilt 相较于Dagger2的优势

  2019 dev submit大会，公开表示dagger2并没有真正解决大家真正遇到的问题，反而学习成本，入门门槛较高，现在已经停止开发新功能了。取而代之的是基于dagger2开发的Hilt组件，大大的降低了学习成本和上手复杂度。

  - Jetpack推荐的DI库,降低 Android 开发者使用依赖注入框架的上手成本,减少了在项目中进行手动依赖(简化使用姿势，dagger2需要大量配置，Hilt不需要)

  - Hilt内部有一套作用域的概念，只能在指定的作用域中使用这个对象，并且提供声明式对象生命周期的管理方式。原本dagger2是不能帮我们管理对象的使用范围，和生命周期，但是在hilt里面，这些问题都不存在了。

  

- #### Hilt如何使用

  基于注解的依赖注入框架，使得对象实例的创建更为简单,可管理。并自动管理它们的生命周期等。

  - 项目根目录`build.gradle`

  ```groovy
  classpath 'com.google.dagger:hilt-android-gradle-plugin:2.28-alpha'
  ```

  - 每个模块的`build.gralde`

  ```groovy
  apply plugin: 'kotlin-kapt'
  apply plugin: 'dagger.hilt.android.plugin'
  
  compileOptions {
          sourceCompatibility = 1.8
          targetCompatibility = 1.8
    }
  
   kotlinOptions {
          jvmTarget = "1.8"
   }
  
  dependencies{
    implementation "com.google.dagger:hilt-android:2.28-alpha"
  kapt "com.google.dagger:hilt-android-compiler:2.28-alpha"
  kapt 'androidx.hilt:hilt-compiler:1.0.0-alpha01'
    //在hilt中使用viewmodel，需要添加下面依赖
    implementation 'androidx.hilt:hilt-lifecycle-viewmodel:1.0.0-alpha01'
  }
  ```

  - step1.配置应用程序

  ```kotlin
  //在编译阶段他会生成顶层的components容器，
  //提供全局的application context。
  //为application提供对象注入的能力
  @HiltAndroidApp
  class HiApplication:Application {
    
  }
  ```

  - step2.配置需要依赖注入的类

  ```kotlin
  //声明该类是一个可以注入的入口类。 仅支持在以下类型中进行依赖注入：
  //this supports appcompatActivity, androidx.fragment, views, services, and broadcast-receivers.  
  //不支持contentprovider,不支持你自定义且不属于以上类型的类
  @AndroidEntryPoint
  class HomeActivity:AppcompatActivity(){
     //声明该对象需要被动态注入
     @inject
     lateinit var emptyView:EmptyView
    
     @inject
     @JvmField
     var iLoginService:ILoginService?=null
     override fun onCreate(bundle: Bundle?) {
          super.onCreate(bundle)
          //Hilt.inject(this)不需要
          //编译时生成Hilt_HomeActivity，在super前面实例化该类，进行成员变量赋值
     }
  }
  ```

  - step3.1 定义对象如何被创建` @Providers`

  ```kotlin
  // .用来告诉 Hilt 这个模块可以被那些组件应用，
  // .声明该模块中创建的对象的生命周期
  @InstallIn(ActivityComponent::class)
  @Module//声明这是一个模块,分组概念,里面的方法必须是public static
  object HomeModule{
    
     @Providers      //直接在方法给出具体的实现代码       
     @ActivityScoped //声明该对象的作用域，在Activity生命周期内共享实例
     fun newEmptyView(@ActivityContext context: Context):EmptyView{
          //@ActivityContext
          //声明context的类型，此处声明为Activity类型的context,
          //除此之外还可使用@ApplicationContext，声明为Application context
         val emptyView = EmptyView(context) emptyView.setDesc(HiRes.getString(R.string.list_empty_desc))
         emptyView.setIcon(R.string.list_empty)
         emptyView.layoutParams =LayoutParams(MATCH_PARENT,MATCH_PARENT)
         emptyView.backgroundColor=Color.white
         emptyView.
     }
  }
  ```

  - step3.2 使用`@binds`提供接口实现注入能力

  ```kotlin
  //接口注入
  @InstallIn(FragmentComponent::class)//不可以在activity中使用
  @Moudle 
  abstract class LoginServiceModule{
      //1. 函数返回类型告诉 Hilt 提供了哪个接口的实例
      //2. 函数参数告诉 Hilt 提供哪个实现
      //3. 没有指定作用域scope，则每次都会生成新的实例对象
      @Binds
      //@FragmentScoped 
      abstract bindLoginService(impl:LoginServiceImpl):ILoginService
  }
  ```

  - `@Binds`：需要在方法参数里面明确写明接口的实现类。
  - `@Provides`：不需要在方法参数里面明确指明接口的实现类，但需要给出具体的实现
  - 两者不能同时出现在一个module里面，因为`Provides`需要定义在`object`修饰的类，而`Binds`需要定义在`abstract`修饰的类。

- #### 限定符

  - 自定义限定符@Qualifier，**提供同一接口，不同的实现**

  ```kotlin
  @Module
  @Install(FragmentComponent::class)
  object MainModule{  
     @Qualifier
     @Retention(AnnotationRetention.RUNTIME）
     annotation class useLoginServiceImpl 
    
     @Qualifier
     @Retention(AnnotationRetention.RUNTIME)
     annotation class useLoginServiceImpl2
     
     @useLoginServiceImpl
     @Provides
     fun provideLoginServiceImpl(): ILoginService { 
         return LoginServiceImpl()
     }
  
     @useLoginServiceImpl2
     @Provides
     fun provideLoginServiceImpl2(): ILoginService { 
         return LoginServiceImpl2() 
     }
  }
         
  @AndroidEntryPoint
  class MainActivity{
     //此时便会使用provideLoginServiceImpl方法创建对象并注入
     @useLoginServiceImpl
     @inject
     lateinit var loginService:ILoginService
  }
  ```

- #### 作用域 scope

  - 默认情况下，Hilt中的所有对象实例都是**无作用域的**。这意味着每次请求绑定时，Hilt都会创建一个新的绑定实例。`@scopes` 的作用在指定作用域范围内(`Application`、`Activity` 等等) 提供相同的实例。
  - 在`@InstallIn`模块中声明实例生命周期的范围时，作用域的范围必须与[component](https://dagger.dev/hilt/components#component-lifetimes)的[作用域](https://dagger.dev/hilt/components#component-lifetimes)匹配。例如，`@InstallIn(ActivityComponent.class)`模块内的绑定只能用限制作用域`@ActivityScoped`

  ```kotlin
  @Module
  @InstallIn(ActivityComponent.class)
  object MainModule{
      @Providers
      @ActivityScoped  //在Activity生命周期内实例唯一
      fun providerLoginService():ILoginService=LoginServiceImpl
  }
  ```

  

| 组件类型                        | 对象生命周期  | 对象作用域               | 创建时机            | 销毁时机             |
| ------------------------------- | ------------- | ------------------------ | ------------------- | -------------------- |
| **`ApplicationComponent`**      | `Application` | `@Singleton`             | onCreate            | onDestroy()          |
| **`ActivityRetainedComponent`** | `ViewModel`   | `@ActivityRetainedScope` | onCreate()          | onDestroy()          |
| **`ActivityComponent`**         | `Activity`    | `@ActivityScoped`        | onCreate()          | onDestroy()          |
| **`FragmentComponent`**         | `Fragment`    | `@FragmentScoped`        | Fragment#onAttach() | Fragment#onDestroy() |
| **`ViewComponent`**             | `View`        | `@ViewScoped`            | View#super()        | `View` 销毁          |
| **`ServiceComponent`**          | `Service`     | `@ServiceScoped`         | onCreate()          | onDestroy()·         |

- 关键注解解释

| 注解                               | 作用                                                         | code some                                                    |
| ---------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| @HiltAndroidApp                    | 1.提供全局的applicationContext,生成相关实现类<br>2.生成components组件<br />3.为application提供依赖注入能力 | **@HiltAndroidApp <br>class..ExampleApplication:Application() { ... }** |
| @AndroidEntryPoint                 | 生成相关的实现类，为Activity,Fragment<br />,View,Service提供依赖注入能力 | **@AndroidEntryPoint <br>class..ExampleActivity:AppCompatActivity(){<br>..@Inject..lateinit..var..viewModel:AnalyticsAdapter <br> }** |
| @Inject                            | 标记需要注入的字段或参数，**其类型不能为private**            | **class LoginAdapter <br>@Inject constructor(val service:ILoginService ) { ... }** |
| @Install(FragmentComponent::class) | 1.用来告诉 Hilt 这个模块可以被那些组件应用<br>2.同时声明该模块中对象的生命周期 | **@InstallIn(FragmentComponent::class) @Module <br>object SearchModule { <br>  @Provides<br>..fun..bindAnalyticsService():AnalyticsService=AnalyticsServiceImpl() <br>}** |
|                                    |                                                              |                                                              |

#### Hint的局限性

Hilt 支持最常见的 Android 类 进行依赖注入

`Application`、`AppcompatActivity`、`androidx.Fragment`、`View`、`Service`、`BroadcastReceiver` 等等，但是我们可能需要在 Hilt 不支持的类中执行依赖注入，在这种情况下可以使用 `@EntryPoint` 注解

比如Hilt 不支持 `ContentProvider`，如果想在 `ContentProvider` 中让Hilt 完成对象的注入，你可以定义一个接口，并添加 `@EntryPoint` 注解，然后添加 `@InstallIn` 注解指定 对象的生命周期边界，代码如下所示。

```kotlin
@EntryPoint
@InstallIn(ApplicationComponent::class)
interface InitializerEntryPoint {
  
    fun injectWorkService(): WorkService
    companion object {
        fun resolve(context: Context): InitializerEntryPoint {
            //如果该模块的生命周期是application，在创建实例时需要使用fromApplication
            //对应的还有fromActivity,fromFragment,fromView
            //具体使用哪个方法需要和InstallIn指定的组件类型匹配
            return EntryPointAccessors.fromApplication(
                context,
                InitializerEntryPoint::class.java
            )
        }
    }
}
class WorkContentProvider : ContentProvider() {
    override fun onCreate(): Boolean {
         val service = InitializerEntryPoint.resolve(this).injectWorkService()
        return true
    }
}
```

#### 在 Dagger 中 依赖注入对比

```kotlin
public class Student {
    private int age;
    public Student(int age) {
        this.age = age;
    }
}

//1. 创建对象提供模型(@Module)
//除了提供带参的对象的提供者以外，还要有提供参数的提供者，二者缺一不可：
@Module
public class StudentModule {
    private int age;
     public StudentModule(int age) {
        this.age = age;
    }
     //提供参数的提供者
    @Provides
    public int ageProvider() {
        return age;
    }
    //创建对象提供者（@Provides）
    @Provides
    public Student studentProvider(int age) {
        return new Student(age);
    } 
}
//2.：创建注入器（@Component）
@Component(modules = StudentModule.class)////指定模型
public interface StudentComponent {
    void inject(MainActivity mainActivity);
}

//3.构建项目rebuild，生成注入器
//4.注入对象
public class MainActivity extends AppCompatActivity {
    @Inject
    Student student;
     @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //手动注入对象
        DaggerStudentComponent.builder().studentModule(new StudentModule(80)).build().inject(this);
    }
}
```

```kotlin
//hilt的写法
//你会发现 hilt不支持从外面传参，很遗憾
@Module
public class StudentModule {
    //创建对象提供者（@Provides）
    @Provides
    public Student studentProvider() {
        return new Student(80);
    } 
}
//3.构建项目rebuild，生成注入器
//4.注入对象
@AndroidEntryPoint
public class MainActivity extends AppCompatActivity {
    @Inject
    Student student;
     @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
    }
}
```
