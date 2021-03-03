---
title: 如何高效的构建渠道包？
categories: 
  - gradle
tags:
  - gradle自动化
  - 多渠道打包
  - V1、V2签名
---

<!--more-->

## 目录
- 什么是多渠道
- 为什么要提供多渠道包
- gradle实现多渠道打包
- V1打包签名原理概述
- V2打包签名原理概述
- 商用多渠道打包方式实战

### **什么是多渠道包**

 渠道包就是要在安装包apk中添加渠道信息，也就是channel,对应不同的渠道，例如:小米市场、360市场、应用宝市场等
 需要为每个apk包设定一个可以区分渠道的标识，这个为apk包设定应用市场标识的过程就是多渠道打包。

### **为什么要提供多渠道包**

 国内存在着有众多的应用市场，产品在不同的渠道可能有不同的统计需求，为此Android开发人员需要为每个应用市场发布一个安装包,在安装包中添加不同的标识，应用在请求网络的时候携带渠道信息，方便后台做运营统计。

### **通过配置gradle实现多渠道打包的方式**

其核心原理就是在编译时通过gradle修改AndroidManifest.xml中的meta-data占位符的内容，执行N次打包流程，然后就可以在java中通过API获取对应的数据。

**补充知识:** **BuildTypes、Flavors、BuildVariants三个定义**

 1、BuildTypes : 构建类型，gradle组件默认提供给了`debug`,`release`构件类型。
 2、Flavors : 产品渠道，可以根据productFlavors，针对不同的渠道生产个性化apk
 3、BuildVariants：每一个buildtype和flavor组成一个buildvariant

- Manifest配置渠道占位符

```xml
<meta-data
            android:name="channel"
            android:value="${channel_value}"/> //yyb,360
```

- app模块的build.gradle

```groovy
android{
    //产品维度，没有实际意义，但gradle要求
    flavorDimensions 'default'
    //productFlavors是android节点的一个节点
    productFlavors {
        baidu {}
        xiaomi { }
        yyb{}
    }
    //如果需要在不同渠道统一配置，使用productFlavors.all字段
    //替换manifest里面channel_value的值
    productFlavors.all { flavor ->
        manifestPlaceholders = [channel_value: name]}

    //设置渠道包名称-> howow-xiaomi-release-1.0.apk
    applicationVariants.all { variant ->
        variant.outputs.all { output ->
            //output即为该渠道最终的打包产物对象
            //设置它的outputFileName属性
            outputFileName = "hwow_${variant.productFlavors[0].name}_${variant.buildType.name}_${variant.versionName}.apk"
        }
    }
}
```

- 运行时获取渠道信息

```java
private String getChannel() {
       PackageManager pm = getPackageManager();
       ApplicationInfo appInfo = pm.getApplicationInfo(getPackageName(), PackageManager.GET_META_DATA);
       return appInfo.metaData.getString("channel_value");
}
```

**但是，这种方式存在一些缺点**：

1. 每生成一个渠道包，都要重新执行一遍构建流程，效率太低，只适用于渠道较少的场景。
2. Gradle会为每个渠道包生成一个不同的BuildConfig.java类，记录渠道信息，导致每个渠道包的dex的CRC值都不同

```java
public final class BuildConfig {
  public static final boolean DEBUG = Boolean.parseBoolean("true");
  public static final String APPLICATION_ID = "org.devio.as.proj.biz_home";
  public static final String BUILD_TYPE = "debug";
  public static final int VERSION_CODE = 1;
  public static final String VERSION_NAME = "1.0";
  //增加了productflavor之后，这里的值，每个apk都不同
  public static final String FLAVOR = "";
}
```

**如果使用了微信的Tinker热补丁方案，那么就需要为不同的渠道包打不同的补丁，因为Tinker是通过对比基础包base.apk和新包new.apk生成差分补丁patch.apk，然后再把补丁patch.apk和已安装的基础包old_base.apk一起合成新的apk。**

**这就要求用于生成差分补丁的基础包base.apk里面的DEX和已安装的基础包old_base.apk里面的DEX是完全一致的，就是说手机上安装的是应用宝渠道包，那必须使用应用宝渠道包来生成补丁patch.apk，才能合并成功）**

### ApkTool

- ApkTool是一个逆向分析工具，可以<font color='red'>**把APK解开，添加渠道信息文件，重新打包重新签名**</font>因此，基于ApkTool的多渠道打包方案分为以下几步：

  1.通过ApkTool工具，解压apk

  2.删除已有签名信息

  3.在meta-inf文件夹下添加渠道信息文件

  4.通过ApkTool工具，重新打包生成新APK

  5.重新签名

但是每生成一个新的渠道包时，都需要重新解包、打包和签名，而这几步操作又是相对机械且耗时。但是若需要同时生成上百个渠道包，则需要几个小时，显然不适合渠道非常多的业务场景

#### **<font color='red'>多渠道打包的切入点原则：附加信息不能影响签名验证</font>**

- **Android Apk的签名是什么？**

  签名就是**apk文件摘要信息使用keystore里面的私钥加密**后的产物，一旦内容被篡改，摘要就会改变，签名是摘要的加密结果，摘要改变，签名也会失效。 APK签名也是这个道理，如果APK签名跟内容对应不起来，系统就认为APK内容被篡改了，从而拒绝安装，以保证系统的安全性。目前Android有三种签名V1、V2（N）、V3（P）

  > 消息摘要（message digest）：将长度不固定的数据作为输入，执行特定的hash函数，生成固定长度的输出，输出的hash值就称为该消息的消息摘要。MD5、SHA、CRC都是hash函数的算法

### V1签名方案概述

<img src="/imgs/gradle/签名流程.png">

- **MANIFEST.MF**：记录apk中每一个文件名与文件SHA1摘要，主要作用是保证每个文件的完整性）
- **CERT.SF :** 记录了对MANIFEST.MF文件整体进行摘要的信息，和每一块摘要的二次摘要，防止MAINIFEST.MF被篡改
- **CERT.RSA：**记录了keystore中存储的公钥证书信息，以及对CERT.SF文件的签名(经keystore中的私钥加密)

> 不同的keystore进行签名，.MF和.SF生成的文件是一致的，不同的是.RSA文件。.MF和.SF保证完整性，.RSA保证来源。
>
> 使用的是jarsigner，位于JDK/bin/jarsigner

- MANIFEST.MF文件格式

```java
Name: AndroidManifest.xml
SHA1-Digest: tgjWjmR6NMAcnuU2pdSuRfmoMZs=

Name: res/mipmap-xxhdpi-v4/ic_launcher_round.png
SHA1-Digest: YXgfepaazMNsoZ37OKGoetkmKeo=

Name: res/mipmap-xxxhdpi-v4/ic_launcher.png
SHA1-Digest: XxxxMiGehaRuVFr2hSLLsenGp6I=
```

- CERT.SF文件格式

```java
Signature-Version: 1.0
SHA1-Digest-Manifest: m4hofJv2im9b2HQo/h6VPKRnzqE=//对MANIFEST.MF文件整体的摘要
Created-By: 1.0 (Android)

Name: AndroidManifest.xml
SHA1-Digest: FvHt2P2c5TfxN/y7CE8O9rhmeyU=

ame: res/mipmap-xxhdpi-v4/ic_launcher_round.png
SHA1-Digest: RtjvorkRz5hy9Ob0K77vtBjnZaA=

Name: res/mipmap-xxxhdpi-v4/ic_launcher.png
SHA1-Digest: 6MThBayhsogaBpVVf8HH/GtKop4=
```

- CERT.RSA文件格式

```java
Version: 3 (O×2) //版本
serial Number: 1461033915 (6x57159bbb) //序列号
Signature Algorlthm: sha1WithRSAEncryption //签名算法
Issuer: cn=shang  //发行者

Validity
Not Beforb: Apr 1902:45:152016 GHT //有效时间
Not After Apr 1302:45:152041 GHT
Subj ect: CN=shang
Sub Ject Public Key Info:
Public Key Algorithm: rsa Encryption
Public-Key: (1024 bit)               //公钥
Modulus:
ee: a2: f1: b3:11: fc: a4:49:02: d2:2c: 5a: 2b: d3: bc
c7:9f: 88:3d: ea: a7:29:85:16:08:41: ad: 40:19:4f
5f: 5b: oa: 6b: 76:6a: b5:45: f7: Bb: 40: bb: ab: 14:26
17:4a: se: a3: e4:4e: b1:7f: a4: f1:, 6f: 6b: e9: ca: 77
42: c4: la: 34:6f: 44:21: cc: 07:26: dd: 62:65:6a: af
9a: e7:69: c1:82:9f: 1f: ce: e9
```

- **V1签名如何防止篡改**

  假如攻击者修改了其中某一个文件，那么他必须修改 **MANIFEST.MF** 中对应文件的摘要，否则这个文件校验不通过； 接着还得修改 **CERT.SF** 中的摘要，否则摘要校验不过； 还得重新计算 **CERT.SF** 的签名，否则签名校验不通过； 但是计算签名需要私钥，私钥在开发者手中，攻击者没有私钥，所以无法签名。

- **V1签名存在的问题**

  校验速度慢： 校验过程中需要对apk中所有文件进行摘要计算，在apk资源很多、性能较差的机器上签名校验会花费较长时间，导致安装速度慢

  完整性不够：V1 签名只会校验 apk 文件中的部分文件，例如 **META-INF** 文件夹就不会参与校验. META-INF目录用来存放签名，自然此目录本身是不计入签名校验过程的，可以随意在这个目录中添加文件，比如一些快速批量打包方案就选择在这个目录中添加渠道文件



### V2 签名方案概述

- **V2 签名是在 Android7.0 之后引入的，它解决了 V1 签名校验时速度慢的问题，同时对apk完整性的校验扩展到整个安装包**

  > 使用的是apkSinger,位于 /build-tools/29/apksigner.bat

- Zip文件存储结构

<center><img src="/imgs/gradle/zip.png" alt="zip" style="zoom:40%;" /></center>

<img src="/imgs/gradle/apk.png">

- **分块计算摘要**

  v2签名不针对单个文件校验了，而是针对APK进行校验，将APK分成多个1M体积的数据块，对每个块计算数据摘要，之后针对所有摘要再次进行摘要。

  V2摘要签名分两级，第一级分块摘要，第二级是对第一级的摘要集合进行摘要，然后利用秘钥进行签名。安装的时候，块摘要可以并行处理，这样可以提高校验速度。

  <img src="/imgs/gradle/chunk.png">

- 签名之后的apk
  <img src="/imgs/gradle/signblock.png">

- 在这个 APK 文件结构下，只有 块2记录apk 签名信息的区域，不是全部参与 v2 签名的校验，所以大家都在这部分做文章。

- APK Signing Block 中包含了多个 "ID-值" 的键值对，而 v2 签名的信息，会记录在 ID 为 `0x7109871a` 中，而不会校验其他 ID 下的值。

- 因此，基于 v2 签名的多渠道方案就这样诞生了：**在 APK 签名块中添加一个额外的 ID-值，用于记录渠道信息**。
- 文件隐写：LSB图像最低位有效算法 和 盲水印技术。

- 可商用的多渠道自动化打包方案

| 打包工具             | 腾讯vasDolly | 美团Walle | packer-ng-plugin |
| -------------------- | ------------ | --------- | ---------------- |
| v1签名               | 支持         | 不支持    | 支持             |
| v2签名               | 支持         | 支持      | 支持             |
| 根据已有包生成渠道包 | 支持         | 不支持    | 不支持           |
| 多线程打包加速       | 支持         | 不支持    | 不支持           |
| 签名校验             | 支持         | 支持      | 支持             |

- 如何集成vasDolly

```groovy
//项目根目录添加插件
dependencies {
    classpath 'com.leon.channel:plugin:2.0.3'
}
```

- 项目根目录新建渠道列表`flavor_channel.txt`

```
360    //每行一个渠道
yyb
xiaomi
huawei
baidu
```

- 根目录新建`flavor_channel.gradle`，并在app模块中应用

```groovy
apply plugin: 'channel'
dependencies{
    //多渠道
    api 'com.leon.channel:helper:2.0.3'
}
channel {
    //指定渠道文件
    channelFile = file("../flavor_channel.txt")
    //多渠道包的输出目录，默认为new File(project.buildDir,"channel")
    baseOutputDir = new File(project.buildDir, "channel")
    //多渠道包的命名规则，默认为：${appName}-${versionName}-${versionCode}-${flavorName}-${buildType}
    apkNameFormat = '${appName}-${versionName}-${flavorName}-${buildType}'
    //快速模式：生成渠道包时不进行校验（速度可以提升10倍以上，默认为false）
    isFastMode = false
    //buildTime的时间格式，默认格式：yyyyMMdd-HHmmss
    buildTimeDateFormat = 'yyyyMMdd-HH:mm:ss'
    //低内存模式（仅针对V2签名，默认为false）：只把签名块、中央目录和EOCD读取到内存，不把最大头的内容块读取到内存，在手机上合成APK时，可以使用该模式
    lowMemory = false
}
/**
 *apkNameFormat支持以下字段：
 *
 * appName ： 当前project的name
 * versionName ： 当前Variant的versionName
 * versionCode ： 当前Variant的versionCode
 * buildType ： 当前Variant的buildType，即debug or release
 * flavorName ： 当前的渠道名称
 * appId ： 当前Variant的applicationId
 * buildTime ： 当前编译构建日期时间，时间格式可以自定义，默认格式：yyyyMMdd-HHmmss
 **/
```

- 读取渠道信息

通过helper类库中的`ChannelReaderUtil`类读取渠道信息。

```
String channel = ChannelReaderUtil.getChannel(getApplicationContext());
```
