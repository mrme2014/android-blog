---
title: 走进Android线程世界
---

<!--more-->

## 目录

## Navigation component案例应用

- 本节目标

- 自定义注解处理器

- 自定义navigator

- 配置化的APP主页架构

  

<br/>

### 本节目标

- 摒弃mobile_navigation.xml，支持开发时注解标记路由节点

- 自定义Fragment 导航器，实现页面hide-show的切换方式

- 配置化的APP主页架构

  <img src="/imgs/route/Navigation案例.png" style="zoom:50%;" width="1600">

  <br/>
  <br/>

### 自定义注解处理器

- gradle配置

```java
api 'com.alibaba:fastjson:1.2.59'
api 'com.google.auto.service:auto-service:1.0-rc6'
annotationProcessor 'com.google.auto.service:auto-service:1.0-rc6'
```

<br/>

- 注解处理器基本用法

```java
@AutoService(Processor.class)
@SupportedSourceVersion(SourceVersion.RELEASE_8)
@SupportedAnnotationTypes({"org.devio.as.hi.nav-annotation.FragmentDestination"})
public class NavProcessor extends AbstractProcessor {
   
   @Override
    void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
      //处理器被初始化的时候被调用
    }
     
    
    boolean process(Set annotations, RoundEnvironment roundEnv) 
      //处理器处理自定义注解的地方
      return false
}
```

- 注解处理器的引用

```java
kapt project(path:'nav-compiler')
api project(path:'nav-annotations')
```

- Java中的几种Element 类型
<table border="1">
  <tr bgcolor="#999999">
    <th width="310">Element</th>
    <th width="310"作用域</th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td>TypeElement</td>
    <td>代表该元素是一个类或者接口</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>VariableElement</td>
   <td>代表该元素是字段，方法入参，枚举常量</td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>PackageElement</td>
   <td>代表该元素是包级别</td>
  </tr>
   <tr  bgcolor="#eeeeee">
   <td>ExecutableElement</td>
   <td>代表方法，构造方法</td>
  </tr>
    </tr>
   <tr  bgcolor="#eeeeee">
   <td>TypeParameterElement</td>
   <td>代表类型，接口，方法入参的泛型参数</td>
  </tr>
</table>
<br></br>

### 自定义Fragment 导航器

```java
@Navigator.Name("HiFragment")
public class HiFragmentNavigator extends Navigator<HiFragmentNavigator.Destination>{
  
public NavDestination navigate(Destination destination, Bundle args) {
    
  }
  
class Destination extends NavDestination{
  public Destination(Navigator navigator){
        
  }
 }
}
```



<br/>

### 配置化的APP主页架构
