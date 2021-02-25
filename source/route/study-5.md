---
title: 实战：基于ARouter实现登录拦截&全局降级策略
---

<!--more-->

## 目录
- 需求分析
- 成果展示
- 疑难点分析
- Coding实现

### 需求分析

  - 利用Arouter拦截页面跳转，实现全局页面降级
    <video src="/imgs/route/arouter_video.mp4" width =360 height=480 controls></video>


### 成果展示
```java
//个人中心页 声明了进入该页面需要登陆
@route(path="/profile/detail",extras=flag_login)
class ProfileDetailActivity extends AppcompactActivity{
  
}

//充值页 声明了进入该页面需要实名认证
@route(path="/profile/detail",extras=flag_authentication)
class ChargeActivity extends AppcompactActivity{
  
}

//会员权益页 声明了进入该页面需要成为会员
@route(path="/profile/detail",extras=flag_vip)
class MemberBenefitsActivity extends AppcompactActivity{
  
}
```



```java
@Interceptor(name = "biz_interceptor")
public class BizInterceptor implements IInterceptor {
    @Override
    public void process(Postcard postcard, InterceptorCallback callback) {
      //得到目标页在route注解中标记的extras字段的值
      int flag = postcard.getExtras();
      
        if((flag&flag_login)!=0){
           //目标页拥有flag_login属性
          login()
        }else if((flag&flag_authentication)!=0){
           //目标页拥有 flag_authentication 属性属性
          authentication()
        }else if((flag&flag_vip)!=0){
           //目标页拥有 flag_vip 属性属性
          becomeVip()
        }else{
          ......
        }
}
```



### 疑难点分析

```java
@route(path="/profile/detail",extras=flag_login|flag_vip)
```

- 为什么可以使用router注解的extras参数，为目标页指定属性

- int型数值,在内存中占4字节，每字节占8位,所以利用extras字段我们可以为目标页指定32个开关

<img src="/imgs/route/arouter_extra.png" style="zoom:50%;" width="1600">

### Coding实现

- 引入Arouter组件到主项目

  ```java
  //模块build.gradle
  apply plugin: 'com.alibaba.arouter'
  apply plugin: 'kotlin-kapt'
  
  android{
    defaultConfig{
      javaCompileOptions {
      annotationProcessorOptions {
          arguments = [AROUTER_MODULE_NAME: project.getName()]
       }
      }
    }
  }
  dependencies{
    api 'com.alibaba:arouter-api:1.5.0'
    kapt 'com.alibaba:arouter-compiler:1.2.2'
  }
  ```

  ```java
  //项目build.gradle
  classpath "com.alibaba:arouter-register:1.0.2"
  ```

  
