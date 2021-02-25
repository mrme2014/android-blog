---
title: Kotlin开发环境搭建技巧
---

## 目录

- 必备条件：
  - Java开发环境
  - Kotlin编辑器
- 创建一个基于Kotlin的Android项目
- 关于Kotlin的包大小
- 为已有基于Java的Android项目添加Kotlin支持
- 将Java 文件转成kotlin文件

Kotlin的运行依赖于JVM，所以首先我们要确保我们的电脑已经安装JDK；然后呢我们需要一个Kotlin的IDE：

- IntelliJ IDEA
- Android Studio
- Eclipse

考虑到大家开发主要用的是Android Studio，接下来就以Android Studio为例来讲解如何创建基于Kotlin的Android项目。

## 创建一个基于kotlin的Android项目

Android Studio 3.0 及更高版本提供全面的Kotlin 支持，如果你的AS版本低于3.0则需要更新AS。接下来就让我们来创建我们的Kotlin项目吧。

>第一步：

打开 Android Studio，在欢迎页面点击 `Start a new Android Studio project` 或者 `File | New | New project`。

>第二步：

选择一个定义应用程序行为的 activity 。对于第一个 "Hello world" 应用程序，选择仅显示空白屏幕的` Empty Activity`，然后点击 `Next`。

![Choosing empty activity](/imgs/kotlin/0-create-new-project.png)

在下一个对话框中，填写工程的详细信息：

![Project configuration](/imgs/kotlin/1-create-new-project.png)

**开发语言：选择 Kotlin**。


完成这些步骤后，Android Studio 会创建一个项目。 该项目已包含用于构建可在 Android 设备或模拟器上运行的应用程序的所有代码和资源。

## 关于Kotlin的包大小

Kotlin有着极小的运行时文件体积：整个库的大小约 1298 KB（1.3.61 版本）。这意味着 Kotlin 对 apk 文件大小影响微乎其微。

>就对比 Kotlin 与 Java所编写的程序而言，Kotlin 编译器所生成的字节码看上去几乎无差异。


## 为已有基于Java的Android项目添加Kotlin支持

为已有基于Java的Android项目添加Kotlin支持有两种方式：

- 通过AS的工具添加：
- 手动添加：

### 通过AS的工具添加

用Android Studio打开已有的Android项目，然后菜单栏上的`Tools`选项 -> `Kotlin` -> `configure kotlin in project`：

![configure-kotlin-with-android-with-gradle](/imgs/kotlin/configure-kotlin-with-android-with-gradle.jpg)

### 手动添加


>your project/build.gradle：

```groovy
buildscript {
    + ext.kotlin_version = '1.3.61'
    repositories {
        google()
        jcenter()

    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.3'
        + classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"

        // NOTE: Do not place your application dependencies here; they belong
        // in the individual module build.gradle files
    }
}
...
```

>app/build.gradle：

```groovy
apply plugin: 'com.android.application'
+ apply plugin: 'kotlin-android-extensions'
+ apply plugin: 'kotlin-android'
...
dependencies {
   ...
    + implementation "androidx.core:core-ktx:+"
    + implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
}
repositories {
    mavenCentral()
}
```

至此，我们已经完成了为原有项目添加kotlin的支持，接下来我们来看下如何将Java 文件转成kotlin文件？

## 将Java 文件转成kotlin文件

打开一个Java文件，然后选择菜单栏上的`Code`选项 -> `convert java file to kotlin file`：

```java
package org.devio.as.hi.kotlin.demo;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

转换后：

```kotlin
package org.devio.`as`.hi.kotlin.demo

import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
    }
}
```

以上便是`Kotlin开发环境搭建`部分的所有内容，接下来呢我们来。


## 参考

- [以 IntelliJ IDEA 入门](https://www.kotlincn.net/docs/tutorials/getting-started.html)

