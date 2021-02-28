---
title: 实战：基于Jenkins实现持续集成与自动打包
---

<!--more-->

## 目录
- 传统包构建方式
- Jenkins实现持续集成与自动打包
- 自定义gradle打包脚本
- 自动上传蒲公英并钉钉群通知

### 传统的包构建方式

- **传统包构建步骤**
  1. git pull origin master 拉取分支最新代码
  2. ./gradlew assembleDebug 构建debug包
  3. build/output/apk/app-debug.apk 部署并运行或发送给测试同学。
- **传统包构建的问题**
  1. 显而易见这些是机械、重复且浪费时间的工作
  2. 不利于多人协作

### Jenkins持续集成与自动打包构建

一切重复的工作皆可自动化，大厂里面都有自动化包构建平台，大多是基于 Jenkins 持续集成方案来定制的。

Jenkins是一个开源软件项目，是基于Java开发的一种[持续集成](https://baike.baidu.com/item/持续集成/6250744)工具，使软件的持续集成变成可能Jenkins提供数百个插件来支持构建，部署和自动化任何项目。

- **[Jenkins软件包安装与服务管理](https://www.jenkins.io/zh/)**

  - 安装：`brew install jenkins-lts`
  - 开启服务：`brew services start jenkins-lts`
  - 重启服务： `brew services restart jenkins-lts`
  - 升级服务：`brew upgrade jenkins-lts`

  > 安装homebrew
  >
  > /usr/bin/ruby -e "$(curl -fsSL [https://raw.githubusercontent.com/Homebrew/install/master/install](https://links.jianshu.com/go?to=https%3A%2F%2Fraw.githubusercontent.com%2FHomebrew%2Finstall%2Fmaster%2Finstall))"

- **服务启动。这个过程首次可能需要4~5分钟。耐心等待**

<center><img src="/imgs/gradle/jenkins1.jpeg" alt="jenkins1" style="zoom:40%;" /></center>

- **以管理身份登陆。在terminal中打开红色字体的文件，把密码输入以继续**

  `open /Users/你电脑名/.jenkins/secrets/initialAdminPassword`

<center><img src="/imgs/gradle/jenkins2.jpeg" alt="jenkins2" style="zoom:40%;" /></center>

- **插件安装。这里选择`安装推荐的插件`，过程需要30分钟左右**

<center><img src="/imgs/gradle/jenkins3.jpeg" alt="jenkins3" style="zoom:40%;" /></center>

- **Jenkins实例实例，默认即可**

<center><img src="/imgs/gradle/jenkins5.jpeg" alt="jenkins5" style="zoom:40%;" /></center>

- **账户创建。这里我们不创建新用户，点击右下角的`使用admin账户继续`。而admin账户的密码就存储在上面红色字体文件中**

<center><img src="/imgs/gradle/jenkins6.jpeg" alt="jenkins6" style="zoom:50%;" /></center>

### Jekins配置

![jenkins7](/imgs/gradle/jenkins7.png)

#### 安装额外的插件

- [Multiple SCMs plugin](https://plugins.jenkins.io/multiple-scms)--多仓库构建
- [Groovy Postbuild](https://plugins.jenkins.io/groovy-postbuild)-- groovy脚本
- [build user vars plugin](https://plugins.jenkins.io/build-user-vars-plugin)--获取当前登录用户新
- [Upload to pgyer](https://plugins.jenkins.io/upload-pgyer)--apk上传蒲公英
- [Locale plugin](https://plugins.jenkins.io/locale) --本地语言

#### 配置(jdk,gradle,android_sdk,git)

1. **Manage Jenkins -->Global Tool Configuration**

   ![jdk](/imgs/gradle/jdk.jpeg)

   ![git](/imgs/gradle/git.jpeg)

   ![gradle](/imgs/gradle/gradle.jpeg)

   > Mac和Windows快速查看git安装目录
   >
   > mac: $which git
   > windows: $where git

2. **Manage Jenkins -->Configure System-->Android SDK**

   ANDROID_HOME的路径必须注入到系统的环境变量中

   ![android_sdk](/imgs/gradle/android_sdk.jpeg)

#### 创建Job任务

- **选择freestyle project类型**

<center><img src="/imgs/gradle/job.jpeg" alt="job" style="zoom:40%;"/></center>

- **常规配置：勾选`this project is paramterized`增加构建选项，可以选择构建debug,release包**

  <center><<img src="/imgs/gradle/job2.jpeg" alt="job2" style="zoom:50%;" />>

- **参与构建项目的源码管理**

  这里我们需要把`ASProj`,`hi-ui`,`hi-library`的仓库都设置进来参与编译。

  这需要安装[Multiple SCMs plugin](https://plugins.jenkins.io/multiple-scms),需要增加则可以点击左小角的`add scm`选择`git`

<center><img src="/imgs/gradle/job3.jpeg" alt="job3" style="zoom:50%;" /></center>

<center><img src="/imgs/gradle/job4.jpeg" alt="job4" style="zoom:50%;" />


<center><img src="/imgs/gradle/job5.jpeg" alt="job5" style="zoom:50%;" /></center>

- **打包环境**

<center><img src="/imgs/gradle/job6.jpeg" alt="job6" style="zoom:50%;" /></center>

- **设置构建Task**

<center><img src="/imgs/gradle/job7.jpeg" alt="job7" style="zoom:50%;" /></center>

- **打包参数透传**

<center><img src="/imgs/gradle/job8.jpeg" alt="job8" style="zoom:50%;" />


### [获取蒲公英的apikey](https://www.pgyer.com/doc/view/api)

### [钉钉自定义消息格式](https://ding-doc.dingtalk.com/doc#/serverapi2/qf2nxq)

### 自定义打包脚本上传apk到蒲公英与钉钉群通知

```groovy
import groovy.json.JsonSlurper
import java.text.SimpleDateFormat

// 打包测试环境 apk 上传蒲公英 钉钉群通知
task packageApk {
    // 让packageApk 依赖于assembleRelease(Debug). 包构建成功后
    //才会执行uploadApk()
    dependsOn("assemble" + BUILD_TYPE.capitalize())
    doLast {
        uploadApk()
    }
}

def uploadApk() {
    // 查找上传的 apk 文件, 这里需要换成自己 apk 路径"
    def apkDir = new File("./app/build/outputs/apk/${BUILD_TYPE}")
    def apk = null
    for (int i = apkDir.listFiles().length - 1; i >= 0; i--) {
        File file = apkDir.listFiles()[i]
        if (file.name.endsWith(".apk")) {
            apk = file
            break
        }
    }
    if (apk == null || !apk.exists()) {
        throw new RuntimeException("apk file not exists!")
    }
    println "*************** upload start ***************"

    String BOUNDARY = UUID.randomUUID().toString(); // 边界标识 随机生成
    String PREFIX = "--", LINE_END = "\r\n";
    String CONTENT_TYPE = "multipart/form-data"; // 内容类型

    try {
        URL url = new URL("https://www.pgyer.com/apiv2/app/upload");
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setReadTimeout(30000);
        conn.setConnectTimeout(30000);
        conn.setDoInput(true); // 允许输入流
        conn.setDoOutput(true); // 允许输出流
        conn.setUseCaches(false); // 不允许使用缓存
        conn.setRequestMethod("POST"); // 请求方式
        conn.setRequestProperty("Charset", "UTF-8"); // 设置编码
        conn.setRequestProperty("connection", "keep-alive");
        conn.setRequestProperty("Content-Type", CONTENT_TYPE + ";boundary=" + BOUNDARY);
        DataOutputStream dos = new DataOutputStream(conn.getOutputStream());

        StringBuffer sb = new StringBuffer();
        sb.append(PREFIX).append(BOUNDARY).append(LINE_END);//分界符
        sb.append("Content-Disposition: form-data; name=\"" + "_api_key" + "\"" + LINE_END);
        sb.append("Content-Type: text/plain; charset=UTF-8" + LINE_END);
        //sb.append("Content-Transfer-Encoding: 8bit" + LINE_END);
        sb.append(LINE_END);
        sb.append("2b3cef360604cc6024d3db76546e6e70");
        sb.append(LINE_END);//换行！


        if (apk != null) {
            /**
             * 当文件不为空，把文件包装并且上传
             */

            sb.append(PREFIX);
            sb.append(BOUNDARY);
            sb.append(LINE_END);
            /**
             * 这里重点注意： name里面的值为服务器端需要key 只有这个key 才可以得到对应的文件
             * filename是文件的名字，包含后缀名的 比如:abc.png
             */
            sb.append("Content-Disposition: form-data; name=\"file\"; filename=\"" + apk.getName() + "\"" + LINE_END);
            sb.append("Content-Type: application/octet-stream; charset=UTF-8" + LINE_END);
            sb.append(LINE_END);
            dos.write(sb.toString().getBytes())

            InputStream is = new FileInputStream(apk)
            byte[] bytes = new byte[1024];
            int len = 0;
            while ((len = is.read(bytes)) != -1) {
                dos.write(bytes, 0, len);
            }
            is.close();
            dos.write(LINE_END.getBytes());
            byte[] end_data = (PREFIX + BOUNDARY + PREFIX + LINE_END).getBytes();

            dos.write(end_data);
            dos.flush();
            /**
             * 获取响应码 200=成功 当响应成功，获取响应的流
             */
            int res = conn.getResponseCode();
            if (res == 200) {
                println("Upload request success");
                BufferedReader br = new BufferedReader(new InputStreamReader(conn.getInputStream()))
                StringBuffer ret = new StringBuffer();
                String line
                while ((line = br.readLine()) != null) {
                    ret.append(line)
                }
                String result = ret.toString();
                println("Upload result : " + result);

                def resp = new JsonSlurper().parseText(result)
                println result
                println "*************** upload finish ***************"
                sendMsgToDing(resp.data)
            } else {
                //发送钉钉 消息--构建失败
            }
        }
    } catch (MalformedURLException e) {
        e.printStackTrace();
    } catch (IOException e) {
        e.printStackTrace();
    }
}

def sendMsgToDing(def data) {
     //括号中的参数需要替换成你自己的webhook url
    def conn = new URL("https://oapi.dingtalk.com/robot/send?access_token=3deb233b815f19f412f44bd42c5ddf7e1ed27b3cf313594c45a4c6755fbbdd50").openConnection()
    conn.setRequestMethod('POST')
    conn.setRequestProperty("Connection", "Keep-Alive")
    conn.setRequestProperty("Content-type", "application/json;charset=UTF-8")
    conn.setConnectTimeout(30000)
    conn.setReadTimeout(30000)
    conn.setDoInput(true)
    conn.setDoOutput(true)
    def dos = new DataOutputStream(conn.getOutputStream())

    //获取蒲公英返回的response中的buildShortcutUrl，短链地址
    //组合成项目的下载主页
    def downloadUrl = "https://www.pgyer.com/" + data.buildShortcutUrl
    def qrCodeUrl = "![](" + data.buildQRCodeURL + ")"
    def detailLink = "[项目地址](${BUILD_URL})"

    def _title = "${JOB_NAME}构建成功"
    def _content = new StringBuilder()
    _content.append("## ${JOB_NAME}构建成功")
    _content.append("\n\n构建版本:${BRANCH_NAME}")
    _content.append("\n\n构建类型:${BUILD_TYPE}")
    _content.append("\n\n下载地址:" + downloadUrl)
    _content.append("\n\n" + qrCodeUrl)
    _content.append("\n\n构建用户:${BUILD_USER}")
    _content.append("\n\n构建时间:" + getNowTime())
    _content.append("\n\n查看详情:" + detailLink)
    def json = new groovy.json.JsonBuilder()
    json {
        msgtype "markdown"
        markdown {
            title _title
            text _content.toString()
        }
        at {
            atMobiles([])
            isAtAll false
        }
    }
    dos.writeBytes(json.toString())
    println("*************** 钉钉消息已发送 ***************")
}
```
