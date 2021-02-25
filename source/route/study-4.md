---
title: 从架构师角度看ARouter实现原理
---

<!--more-->

## 目录
- Arouter配置与基于用法
- Arouter编译时原理分析
- Arouter运行时原理分析
- 路由组件技术选型

  
<img src="/imgs/route/Arouter.png" style="zoom:50%;" width="1600">
<br></br>

### Arouter配置与基本用法

- 依赖引入与配置

```java
defaultConfig{
 javaCompileOptions {
   annotationProcessorOptions {
   arguments = [AROUTER_MODULE_NAME:project_name,AROUTER_GENERATE_DOC:"enable"]
     }
   }
}

dependencies{
  api 'com.alibaba:arouter-api:1.5.0'
  kapt 'com.alibaba:arouter-compiler:1.2.2'
}
```

```java
//项目根目录build.gradle
dependencies {
  classpath "com.alibaba:arouter-register:1.0.2"
}
```

- Arouter基本方法

```java
module trade:
@Route(path = "/trade/detail",group="trade")
public class DetailActivity extends Activity{
  @AutoWired
  public String saleId
  @AutoWired
  public String shopId;
  
  public void onCreate(Bundle bundle){
     Arouter.getInstance().inject(this)
  }
}
```

```java
ARouter.getInstance().build("/trade/detail/activity")
                    .withString("saleId","1001")
                    .withString("shopId","1002")
                    .navigation()
```

- IProvider 跨模块Api调用,依赖解耦.服务管理
  
<img src="/imgs/route/iprovider.png" style="zoom:50%;" width="1600">
<br></br>
- 注解信息收集，分组写入文件
<img src="/imgs/route/arouter_processor.png" style="zoom:50%;" width="1600">
<br></br>

- 路由分组管理,按需加载
<img src="/imgs/route/Arouter路由分组管理.png" style="zoom:50%;" width="1600">
<br/><br/>

- Javapoet编译时文件写入

  javapoet依赖引入

  ```java
   implementation 'com.squareup:javapoet:1.8.0'
  ```

  ```java
  目标文件结构
  package com.example.helloworld;
  import java.util.Date;
  public final class HelloWorld {
    Date today() {
      return new Date();
    }
  }
  ```

  ```java
  javapoet
  MethodSpec today = MethodSpec.methodBuilder("today")
      .returns(Date.class)
      .addStatement("return new $T()", Date.class)
      .build();
  
  TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")
      .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
      .addMethod(today)
      .build();
  
  JavaFile javaFile = JavaFile.builder("com.example.helloworld", helloWorld)
      .build();
  
  javaFile.writeTo(outputFile);
  ```
<br></br>

### Arouter运行时时原理分析



- 初始化流程
<img src="/imgs/route/arouter_init.png" style="zoom:50%;" width="1600">
<br></br>

- 执行路由流程

```java
ARouter.getInstance().build("/trade/detail/activity")
                    .withString("sale_id","1001")
                    .withString("shop_id","1002")
                    .navigation()
```

<img src="/imgs/route/arouter_nav2.png" style="zoom:50%;" width="1600">
<br></br>

### 路由组件技术选型
<table border="1">
  <tr bgcolor="#999999">
    <th width="310">类型</th>
    <th> Navigation</th>
    <th> ARouter</th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td>跳转行为</td>
    <td>通过页面的action跳转,支持Activity,Fragment,Dialog</td>
    <td>支持标准URL跳转</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>模块间通信</td>
   <td>❎不支持，自行拓展</td>
   <td>@Route 注解配置，支持根据Path获取对应接口实现</td>
  </tr>
   <tr  bgcolor="#ffffff">
    <td>路由节点注册</td>
    <td>统一在navigation_mobile.xml中注册</td>
    <td>@Route注解</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>路由节点扩展</td>
   <td>一般</td>
   <td>一般 </td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>拦截器</td>
   <td>❎不支持 </td>
   <td>支持配置全局拦截器,可以自定义拦截顺序 </td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>转场动画</td>
   <td>支持</td>
   <td>支持</td>
  </tr>
   <tr  bgcolor="#ffffff">
   <td>降级策略</td>
   <td>❎不支持</td>
   <td>支持全局降级和局部降级 </td>
  </tr>
    <tr  bgcolor="#eeeeee">
   <td>跳转监听</td>
   <td>❎不支持</td>
   <td>支持全局和单次 </td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>跳转跳转参数监听</td>
   <td>支持基本类型和自定义类型</td>
   <td>支持基本类型和自定义类型</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>参数自动注入</td>
   <td>❎不支持</td>
   <td>@Autowired 注解的属性可被自动注入</td>
  </tr>
   <tr  bgcolor="#ffffff">
   <td>外部跳转控制</td>
   <td>deeplink页面直达 </td>
   <td>需要配置入口Acitity，支持的uri需要在Manifest中配置</td>
  </tr>
     <tr  bgcolor="#eeeeee">
   <td>回退栈管理</td>
   <td>支持逐个出栈，也支持直接回退到某个页面 </td>
   <td>❎不支持</td>
  </tr>
      <tr  bgcolor="#ffffff">
   <td>自动生成路由文档</td>
   <td>❎不支持 </td>
   <td>支持</td>
  </tr>
</table>
<br/>