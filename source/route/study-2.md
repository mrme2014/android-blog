---
title: Navigation component架构概述
---

<!--more-->

## 目录

- 接入navigation
- navigation基本用法
- navigation架构概述
- navigation进阶

<br/>

### 接入navigation

```java
api 'androidx.navigation:navigation-fragment:2.2.1'
api 'androidx.navigation:navigation-ui:2.2.1'
```

<br/>

### navigation基本用法

- 在mobile_navigation.xml中构建导航视图

```java
<navigation 
    android:id="@+id/mobile_navigation"
    app:startDestination="@id/navigation_dashboard">
    <fragment
        android:id="@+id/navigation_home"
        android:name="org.devio.as.hi.HomeFragment"
        android:label="@string/title_home" />

  <Activity
        android:id="@+id/navigation_home"
        android:name="org.devio.as.hi.HomeActivty"
        android:label="@string/title_home_activity" />
</navigation>
```



### navigation架构概述
<img src="/imgs/route/navigation.png" alt="navigation" style="zoom:50%;" width="2000">
<br></br>

### Navigation进阶实战

- 抛弃mobile_navigation.xml资源文件,改为注解标记
- 自动化构建可配置化的App主页面架构
