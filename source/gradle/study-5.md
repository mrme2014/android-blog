---
title: APK打包与安全
categories: 
  - Apk安全
tags:
  - 打包原理
  - aapt2
  - 包加固
  - 反编译
---

<!--more-->

## 目录
- aapt2 命令行实现apk打包
- apk反编译
- apk加固原理
- 360加固实战

### aapt2 命令行实现apk打包

#### **apk文件结构**

<img src="/imgs/gradle/apk.jpeg" />

- **classes.dex**：Dex是**D**alvikVM **ex**ecutes的缩写，即Android Dalvik执行文件

- **AndroidManifest.xml**：Project中AndroidManifest.xml编译后得到的二进制xml文件 
- **META-INF**：主要保存各个资源文件的SHA1 hash值，用于校验资源文件是否被篡改，防止二次打包时资源文件被替换，该目录下主要包括下面三个文件：
  - MANIFEST.MF：保存版本号以及对每个文件（包括资源文件）整体的SHA1 hash
  - CERT.SF：保存对每个文件头3行的SHA1 hash
  - CERT.RSA：保存签名和公钥证书

- **res**：Project中res目录下资源文件编译后得到的二进制xml文件
- **resources.arsc**：包含了所有资源文件的映射，可以理解为资源索引，通过该文件能找到对应的资源文件信息

#### aapt2打包流程

<center><img src="/imgs/gradle/aapt4.jpeg" alt="aapt4" style="zoom:80%;" /></center>



<center><img src="/imgs/gradle/aapt3.png" alt="aapt3" style="zoom:25%;" /></center>

1. **通过aapt2打包res资源文件:**生成R.java、resources.arsc和res文件（二进制 & 非二进制如res/raw和pic保持原样）
2. **通过Javac编译R.java、Java接口文件、Java源文件:**生成.class文件
3. **通过d8命令:**将.class文件和第三方库中的.class文件处理生成classes.dex
4. **通过aapt2工具:**将aapt生成的`resources.arsc`和`res`文件、未编译的资源`assets`文件和`classes.dex`一起打包生成apk
5. **通过zipalign工具:**将未签名的apk进行对齐处理。
6. **通过apksigner工具:**对上面的apk进行debug或release签名

| 工具名称                                                     | 功能介绍                     | 工具所在路径                                 |
| ------------------------------------------------------------ | ---------------------------- | -------------------------------------------- |
| [aapt2](https://developer.android.com/studio/command-line/aapt2) | android资源打包工具          | ${android_sdk}/build-tools/29.0.0/aapt2      |
| [Kotlinc](https://www.jianshu.com/p/fc86e386210f)            | 把.kt文件编译成.class文件    | ${android studio}/plugins/Kotlin/kotlinc     |
| [javac](https://www.jianshu.com/p/b6b96f78f470)              | 把.java编译成.class          | /usr/bin/javac                               |
| [d8](https://developer.android.google.cn/studio/command-line/d8) | 新一代.class转化成.dex的工具 | ${android_sdk}/build-tools/29.0.0/aidl       |
| [zipaligin](https://developer.android.google.cn/studio/command-line/zipalign) | 字节码对齐工具               | ${android_sdk}/build-tools/29.0.0/zipalign   |
| [apksigner](https://developer.android.google.cn/studio/command-line/apksigner) | apk签名工具                  | ${android_sdk}/build-tools/29.0.0/apksiginer |

#### [aapt2](https://developer.android.com/studio/command-line/aapt2)命令行实现打包

- 使用`appt2 compie`编译资源文件 ,把`*.xml`文件压缩成二进制文件使之体积更小，运行时解析更快。drawable文件默认会被压缩，raw/目录下文件原封不动
  - --dir 资源文件的目录
  - -o 编译后的资源输出位置

```powershell
//对单个文件编译
aapt2 compile app/src/main/res/values/strings.xml -o apk/
//对目录下的所有资源打包编译  
aapt2 compile --dir app/src/main/res/ -o apk/res.zip   
```

- 使用`aapt2 link`为资源分配id,并生成R.java文件和resources.arsc资源索引表文件
  - -I 参与资源链接的android.jar的路径
  - --java 指定生成的R.java文件的路径
  - --manifest 指定参与资源链接的manifest文件的路径
  - -o 指定资源链接后 输出的.apk的路径（此时不包含.dex文件，也没签名）

```powershell
aapt2 link apk/res.zip \
-I /Users/timian/Library/Android/sdk/platforms/android-29/android.jar \
--java apk/ \
--manifest app/src/main/AndroidManifest.xml \
-o apk/res.apk
```

- 使用[javac](https://www.jianshu.com/p/b6b96f78f470)编译.java文件，生成.class文件
  - -encoding 指定编译的格式
  - -target 指定参与编译的jdk版本
  - -bootclasspath 指定参与编译的android.jar的路径
  - -d 指定输出的.class文件的路目录
  - --classpath指定参与编译的类路径资源(appcompat,recyclerview)

```powershell
javac -encoding utf-8 \
-target 1.8 \
-bootclasspath /Users/timian/Library/Android/sdk/platforms/android-29/android.jar \
app/src/main/java/org/devio/as/proj/aapt/*.java apk/org/devio/as/proj/aapt/R.java \
-d apk/
```

- 使用[d8](https://developer.android.google.cn/studio/command-line/d8)命令把.class文件转换成.dex文件
  - 输入为aapt目录下所有.class文件
  - --lib指定参与编译的android.jar的路径
  - --output指定输出的.dex文件的路径,这里我们需要把它指定到项目根目录下，下一步里需要把dex和res合并,否则会把dex文件的文件夹也带进去

```powershell
d8 apk/org/devio/as/proj/aapt2/*.class \
--lib /Users/timian/Library/Android/sdk/platforms/android-29/android.jar \
--output ./
```

- 使用aapt吧`classes.dex`添加到`res.apk`里面去
  - 原本这步是通过 apkbuilder 工具，现在改成用 aapt 命令来做。

```powershell
aapt add apk/res.apk classes.dex
```

- 使用[zipaligin](https://developer.android.google.cn/studio/command-line/zipalign)工具对.apk字节码进行4字节对齐

```powershell
zipalign 4 apk/res.apk apk/app-unsigned-aligned.apk
```

- 使用[apksigner](https://developer.android.google.cn/studio/command-line/apksigner)对.apk进行签名
  - --ks key密钥文件路径
  - --out 签名后的.apk文件路径 
  - 最后参与签名的.apk文件的路径

```powershell
apksigner sign --ks key --out apk/app-release.apk apk/app-unsigned-aligned.apk
```

#### gradle构建工具打包

assembleDebug打包流程中关键的 Task 所对应的实现类与含义

| Task                       | 对应实现类                          |                             作用                             |
| :------------------------- | :---------------------------------- | :----------------------------------------------------------: |
| preBuild                   | AppPreBuildTask                     |    预先创建的 task，用于做一些 application Variant 的检查    |
| generateDebugBuildConfig   | GenerateBuildConfig                 |             生成与构建目标相关的 BuildConfig 类              |
| javaPreCompileDebug        | JavaPreCompileTask                  |            用于在 Java 编译之前执行必要的 action             |
| generateDebugResValues     | GenerateResValues                   |               生成 Res 资源类型值(xml中定义id)               |
| compileDebugAidl           | AidlCompile                         |                        编译 AIDL 文件                        |
| processDebugManifest       | ProcessApplicationManifest          |                      处理 manifest 文件                      |
| mergeDebugResources        | MergeResources                      |                   使用 AAPT2 合并资源文件                    |
| processDebugResources      | LinkApplicationAndroidResourcesTask |               用于处理资源并生成 R.class 文件                |
| compileDebugJavaWithJavac  | AndroidJavaCompile                  |                   用于执行 Java 源码的编译                   |
| processDebugJavaRes        | ProcessJavaResTask                  |                      处理 Java Res 资源                      |
| checkDebugDuplicateClasses | CheckDuplicateClassesTask           |            用于检测工程外部依赖，确保不包含重复类            |
| dexBuilderDebug            | D8DexArchiveBuilder                 | 用于将 .class 文件转换成 dex archives，即 DexArchive，Dex 存档，可以通过 addFile 添加一个 DEX 文件 |
| mergeDebugJavaResource     | MergeJavaResourceTask               |               合并来自多个 moudle 的 Java 资源               |
| packageDebug               | PackageApplication                  |                           打包 APK                           |
| assembleDebug              | Assemble                            |                       空 task，锚点使                        |

### APK安全攻守道

| 风险种类                 | 风险描述                                                     | 解决方案                                                     |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| App防止反编译            | 被反编译的暴露客户端逻辑，加密算法，密钥，等等               | 加固阿里（聚安全），腾讯（乐加固），360（加固宝）等方式加固自身的应用。 |
| 资源文件泄露风险         | 获取图片，js文件等文件，通过植入病毒，钓鱼页面获取用户敏感信息 | 资源混淆(AndResGuard)，加固等等                              |
| so文件破解风险           | 导致核心代码逻辑泄漏                                         | so加固                                                       |
| 测试开关的代码被打包发布 | 通过测试的Url，测试账号等对正式服务器进行攻击。测试开关的代码被打包发布 | 把内网测试的日志清除，或者测试服务器和生产服务器不要使用同一个 |
| Root设备运行风险         | 已经root的手机通过获取应用的敏感信息等                       | 检测是否是root的手机禁止应用启动                             |
| 模拟器运行风险           | 刷单，模拟虚拟位置等                                         | 禁止在虚拟器上运行                                           |
| 截屏攻击风险             | 对APP运行中的界面进行截图或者录制来获取用户信息              | 添加属性getWindow().setFlags(FLAG_SECURE)不让用户截图和录屏  |
| 输入监听风险             | 用户输入的信息被监听或者按键位置被监听造成用户信息泄露等     | 自定义键盘                                                   |

#### 反编译

- [dex2jar](https://github.com/pxb1988/dex2jar/releases/tag/2.0)把dex文件反编译成jar包

  ```powershell
  sudo chmod +x d2j_invoke.sh//取消d2j_invoke.sh文件的权限
  
  sh dex2jar/d2j-dex2jar.sh apk/classes.dex -o classes.jar //执行命令
  ```

- [jd-gui](http://java-decompiler.github.io/#jd-gui-download) jar包class查看工具

#### Apk加固

- **dex加密并与壳dex生成加固后的dex**
  - 把原apk文件中提取出来的classes.dex文件通过加密程序进行加密
  - 将加密后的dex文件追加在壳dex文件后面，生成新的dex文件

<center><img src="/imgs/gradle/dex加壳2.png" alt="dex加壳2" style="zoom:50%;" /></center>

- **修改原始apk并重新打包签名**

  - 将上一步生成的`加固后dex`替换掉原始的dex

  - 由于程序运行的时候，需要先加载StubApplication类。所以，我们需要修改`AndroidManifest.xml`文件，指定application为`StubApplication`

<img src="/imgs/gradle/dex加壳3.png" />

- #### 脱壳

  - 在attachBaseContext方法里，主要做两个工作：
    - 释放原始dex文件并解密
    - 然后使用自定义的DexClassLoader加载解密后的原dex文件
    - 反射ActivityThread中的loadedApk中的类加载器为自定义的类加载器

  - 在onCreate方法中，主要做两个工作：
    - 创建原Application对象，并调用原Application的onCreate方法启动原程序
    - 通过反射修改ActivityThread类，并将Application指向原dex文件中的Application

   <img src="/imgs/gradle/dex加壳4.png" />

- [360加固保](https://jiagu.360.cn/#/global/download)
