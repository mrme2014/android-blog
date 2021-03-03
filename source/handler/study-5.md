---
title: 集成Bugly实现热修复
---

## 目录
- 搭建补丁下发后台
- 后台下发补丁策略（全量，定量，开发机）
- 下发维度控制，有效控制补丁影响范围（补丁版本控制）
- 考虑补丁下载合成的时机（后台合成，空闲时合成，合成后要不要重启？）
- HTTPS及签名校验等机制保障补丁下发的安全性（如何校验补丁文件完整）

<img src="{{root}}/imgs/handler/tinker1.png" alt="tinker1" style="zoom:50" />

### gradle打包配置

- 项目根目录添加tinker-support gradle插件，提供编译时各种task，用于生产patch包

```groovy
dependencies {
        //gradle需要从原来的3.5.0降级到3.4.2  bugly插件还没适配
        classpath 'com.android.tools.build:gradle:3.4.2'
        classpath "com.tencent.bugly:tinker-support:1.1.5"
}
```

- 主项目APP 添加依赖配置签名和拆分包，应用tinker打包配置gradle文件

```groovy
apply from: '../tinker-support.gradle'

android {
    signingConfigs {
        config {
            storeFile file('../as_key_store')
            storePassword '123456'
            keyAlias 'key0'
            keyPassword '123456'
        }
    }
    defaultConfig {
        applicationId "org.devio.as.proj.main"
        //开启拆分包
        multiDexEnabled true
        //配置支持的动态库类型  
        ndk{
            abiFilters 'armeabi-v7a','x86'
        }
    }
    buildTypes {
        //配置 release debug的签名
        release {
            ...
            signingConfig signingConfigs.config
        }
        debug{
            signingConfig signingConfigs.config
        }
    }
    //开启jumboMode开关，tinker在diff时，可以使得dex文件更小
    dexOptions {
        jumboMode = true
    }
  
    compileOptions {
        sourceCompatibility = 1.8
        targetCompatibility = 1.8
    }
}

dependencies {
  //添加multidex
  implementation 'androidx.multidex:multidex:2.0.1'
   
   //添加对bugly  和 tinker，包含了crash捕获和热修复
   implementation 'com.tencent.bugly:crashreport_upgrade:1.3.6'
    implementation 'com.tencent.tinker:tinker-android-lib:1.9.9'
}
```

- Application 初始化

```java
public void onCreate() {
        Bugly.init(this, "10d86971eb", true);
        Bugly.setIsDevelopmentDevice(this, true);
}

protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        MultiDex.install(base);
        Beta.installTinker();
}
```

- manifest清单文件,添加权限以及BetaActivity

```xml
 <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
    <uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
    <uses-permission android:name="android.permission.READ_LOGS" />
    <uses-permission android:name="android.permission.READ_PHONE_STATE"/>
```

```xml
 <!-- bugly-->
    <activity
         android:name="com.tencent.bugly.beta.ui.BetaActivity"  
                                android:configChanges="keyboardHidden|orientation|screenSize|locale"
         android:theme="@android:style/Theme.Translucent" />
```



- tinker-support.gradle 打包配置文件，比原生tinker配置要简化的非常多

需要在项目根目录下创建该文件

```groovy
apply plugin: 'com.tencent.bugly.tinker-support'

//配置打包生成的apk的路径，定位在APP模块下的build/bakApk目录下
//形式如/user/**/desktop/org/.../main/app/bakApk
def bakPath = file("${project(":app").buildDir}/bakApk/")

//此处填写需要构建patch的发布包所在目录
//因为每次打包都会在上面的bakApk目录下生成一个以当前打包时间命名的文件件，文件件中存放着本次打包生成的apk,mapping.txt,R.txt.
def baseApkDir = 'app-0708-00-47-09'

//https://bugly.qq.com/docs/utility-tools/plugin-gradle-hotfix/
tinkerSupport {

    // 开启tinker-support插件，默认值true
    enable = true
  
    // 是否启用覆盖tinkerPatch配置功能，默认值false
    // tinkerSupport是bugly提供的配置属性
    // 如果为true ,下面tinkerPatch中配置的属性不会生效,推荐这种。tinker的配置太繁琐复杂
    overrideTinkerPatchConfiguration = true

    // 生成patch.dex目录，默认值当前module的子目录tinker
    autoBackupApkDir = "${bakPath}"

    // 编译补丁包时，必需指定基线版本的apk，也就是针对哪个线上版本生成patch
    baseApk = "${bakPath}/${baseApkDir}/app-release.apk"

    // 对应tinker插件applyMapping 混淆规则
    baseApkProguardMapping = "${bakPath}/${baseApkDir}/app-release-mapping.txt"

    // 对应tinker插件applyResourceMapping 资源id映射
    baseApkResourceMapping = "${bakPath}/${baseApkDir}/app-release-R.txt"

    //自动生成下面的tinker的值 时间戳形式0707-12-10-10
    autoGenerateTinkerId = true

    // 构建基准包和补丁包都要指定不同的tinkerId，并且必须保证唯一性
    tinkerId = "if autoGenerateTinkerId=true ,no need set here"

    // 是否开启反射Application模式
    enableProxyApplication = true

    // 是否支持新增非export的Activity（注意：设置为true才能修改AndroidManifest文件）
    supportHotplugComponent = true

}

/**
 * 一般来说,我们无需对下面的参数做任何的修改
 * 对于各参数的详细介绍请参考:
 * https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97
 */
tinkerPatch {
    //oldApk ="${bakPath}/${appName}/app-release.apk"
    ignoreWarning = true
    useSign = true
    dex {
        dexMode = "jar"
        pattern = ["classes*.dex"]
        loader = []
    }
    lib {
        pattern = ["lib/*/*.so"]
    }

    res {
        pattern = ["res/*", "r/*", "assets/*", "resources.arsc", "AndroidManifest.xml"]
        ignoreChange = []
        largeModSize = 100
    }

    packageConfig {
    }
    sevenZip {
        zipArtifact = "com.tencent.mm:SevenZip:1.1.10"
//        path = "/usr/local/bin/7za"
    }
    buildConfig {
        keepDexApply = false
        //tinkerId = "1.0.1-base"
        //applyMapping = "${bakPath}/${appName}/app-release-mapping.txt" //  可选，设置mapping文件，建议保持旧apk的proguard混淆方式
        //applyResourceMapping = "${bakPath}/${appName}/app-release-R.txt" // 可选，设置R.txt文件，通过旧apk文件保持ResId的分配
    }
}
```

<font color='red'>**需要把项目几个模块中的proguard-rules.pro**拷到你的工程里，否则混淆后将会报错</font>

### 验证热修复能力

- 构建基准包(可以理解为发布的包)。并且安装到你的手机，并联网打开，使得bugly上报该版本的tinkerId有人在使用，否则发布patch时，将会报错没有匹配到基准包tinkerId
  <img src="{{root}}/imgs/handler/tinker2.png" alt="tinker2" style="zoom:50%;" />

- 构建patch补丁

  随便找个地方修改几行代码。按照下图步骤构建patch补丁文件，最后将补丁文件上传到bugly补丁管理后台

<img src="{{root}}/imgs/handler/tinker3.png" alt="tinker3"/>

- 完整的测试流程

  - 打基准包安装并上报联网（注：填写唯一的tinkerId）
  - 对基准包的bug修复（可以是Java代码变更，资源的变更）
  - 修改基准包路径、修改补丁包tinkerId、mapping文件路径（如果开启了混淆需要配置）、resId文件路径
  - 执行buildTinkerPatchRelease打Release版本补丁包
  - 选择app/build/outputs/patch目录下的补丁包并上传（注：不要选择tinkerPatch目录下的补丁包，不然上传会有问题）
  - 编辑下发补丁规则，点击立即下发
  - 杀死进程并重启基准包，请求补丁策略（SDK会自动下载补丁并合成）
  - 再次重启基准包，检验补丁应用结果
  - 查看页面，查看激活数据的变化



### 注意事项

1. Tinker不支持修改AndroidManifest.xml，Tinker不支持新增四大组件(1.9.0支持新增非export的Activity)；
2. 由于Google Play的开发者条款限制，不建议在GP渠道动态更新代码；
3. 在Android N上，补丁对应用启动时间有轻微的影响；
4. 不支持部分三星android-21机型，加载补丁时会主动抛出`"TinkerRuntimeException:checkDexInstall failed"`；
5. 对于资源替换，不支持修改remoteView。例如transition动画，notification icon以及桌面图标
6. 每个项目的配置打包，构建补丁都会出现各种各样的问题。那此时，请注意观察控制台错误日志的输入。接着可以到bugly和tinker在github上的issues上面，搜一搜，看一看，目前还有哪些现存的问题。那么这些问题有可能就是你遇到的问题。

