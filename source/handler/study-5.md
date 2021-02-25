---
title: 架构师该如何选择导航框架
---

## 目录

- Android路由框架的诞生之路
- 传统路由方式
- 路由的最佳实践
- Navigation&Arouter横向对比
- 架构师技术选型

<br/>

### Android路由框架的诞生之路

<img src="Android路由.png" alt="Android路由" style="zoom:50%;" />

<br/>

### 传统路由方式

```java
//显性意图
startActivity(new Intent(this, HomeActivity.class));

//隐性意图
startActivity(new Intent("imooc://nativepage/home"));

//Eventbus
EventBus.getDefault().post(new HomePageEvent());

//静态方法调用
ActivityManager.openHomeActivity(Intent intent)
```

<br/>

### 路由的最佳实践

```java
@Path(path = "nativepage/home")
public class HomeActivity extend AppcompactActivity {
    
}

Router.for("nativepage/home")
       .withString("name", "888")
       .withInt("age", 11)
       .open();
```



<br/>

### Arouter&Navigation横向对比
<img src="/imgs/route/路由框架.png" style="zoom:50%;" />
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

### 如何做好技术选型

<img src="/imgs/route/技术选型.png" alt="技术选型" style="zoom:70%;" />
