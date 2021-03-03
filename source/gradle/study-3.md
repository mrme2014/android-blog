---
title: 实战：发布Gradle插件到JCenter
categories: 
  - jcenter
tags:
  - jcenter
  - bintray
---

<!--more-->

## 目录

- Api Key与仓库配置
- 发布前的准备
- 提交Bintray与发布Jcenter



### Api Key与仓库配置

- [账号注册](https://bintray.com/signup/oss)最好使用gmail邮箱

<center><img src="/imgs/gradle/bintary1.jpeg" alt="bintary1" style="zoom: 40%;" /></center>

- 登陆的`username`为上面注册时填写的`username`,使用邮箱无法登陆

<center><img src="/imgs/gradle/sigin.jpeg" alt="sigin" style="zoom:40%;"/></center>

- Api Key

  登陆之后点击右上角的`Edit Profile`进入个人主页，再点击左侧最后一个`API KEY`

  在plugin和library发布时需要用到

<center><img src="/imgs/gradle/bintary2.jpeg" alt="bintary2" style="zoom:40%;" /></center>

- repository仓库创建

  添加仓库名字，和仓库的类型为maven即可

<center><img src="/imgs/gradle/bintary3.jpeg" alt="bintary3" style="zoom:40%;" /></center>



### 发布前的准备

- 添加插件`com.novoda.bintray-release`
- [发布可配属性](https://github.com/novoda/bintray-release/wiki/Configuration-of-the-publish-closure)

```groovy
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.novoda:bintray-release:0.9'
    }
}

//在跟目下的local.properties配置 bintray网站的用户名和apikey
//不能够写死在build.gradle中
Properties properties = new Properties()
properties.load(file('../local.properties').newDataInputStream())

def user = properties.getProperty("bintray.user")
def key = properties.getProperty("bintray.apikey")

apply plugin: 'com.novoda.bintray-release'
publish {
    userOrg = 'qiaomu'  //组织，没有的话，就填用户名
    repoName = 'devio'  //在bintray上创建的仓库名称
    groupId = 'org.devio.as.proj.plugin'  //要发布的插件（库）的组，一般跟包名保持一致
    artifactId = 'devio-plugin'   //要发布的插件（库）的名称
    publishVersion = '1.0.0'      //版本号
    bintrayUser = user//用户名
    bintrayKey = key//bintray秘钥
    desc = 'Oh hi, this is a nice description for a project, right?'
    website = 'https://github.com/mrme2014/devio-plugin'
    repository ='https://github.com/mrme2014/devio-plugin.git'
    autoPublish=true
    dryRun =false
}
```

### 发布Bintray与发布Jcenter

- 执行命令

  但是此时还并没有发布到jcenter,只是提交到了 bintray。

```groovy
 ./gradlew clean build bintrayUpload
```

- 发布jcenter

  点击右侧的`add to Jcenter`.等待审核。审核结果会有邮件通知

<center><img src="/imgs/gradle/bintary4.jpeg" alt="bintary4" style="zoom:40%;" /></center>

- 如果通过审核了，右侧会显示`Linked to(1)`

<center><img src="/imgs/gradle/bintray5.jpeg" alt="bintray5" style="zoom:40%;" /></center>

- 使用

  在未审核通过之前使用的话，需要添加你在bintray上创建的仓库的地址

  ```groovy
    buildscript {
      repositories {
          google()
          jcenter()
          maven{url 'https://dl.bintray.com/qiaomu/devio'}
      }
      dependencies {
          //gradle需要从原来的3.5.0降级到3.4.2  bugly插件还没适配
          classpath 'com.android.tools.build:gradle:3.4.2'
          classpath 'org.devio.as.proj.plugin:devio-plugin:1.0.2'
      }
  }
  ```

  


