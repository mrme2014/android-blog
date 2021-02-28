---
title: 深入浅出Gradle插件开发
---

<!--more-->

## 目录

- Gradle 构建生命周期
- Gradle Project工程树
- 自定义Task
- 自定义插件Transform
- 字节码插桩技术

### Gradle 构建脚本基础

<center><img src="/imgs/gradle/Gradle.png" alt="Gradle" style="zoom:60%;" /></center>

- **groovy语法:**继承自java语法，不需要专门去学习它。
- **gradle api:**类似于 android sdk一样的存在，在开发gradle插件时提供底层支持。
- **buildscript :**声明gradle脚本构建项目时所需要使用依赖项、仓库地址、第三方插件等。构建项目时会优先执行buildscript代码块中的内容。

### 项目构建生命周期

<center><img src="/imgs/gradle/项目构建.png" alt="项目构建" style="zoom:50%;" /></center>

- **初始化阶段:**会执行项目根目录下的<font color='red'>`Settings.gradle`</font>文件，分析哪些project参与本次构建

  > include ':app'
  >
  > include ':biz_home','biz_detial','biz_order'

- **配置阶段:**加载所有参与本次构建项目下的<font color='red'>`build.gradle`</font>文件，会将`build.gradle`文件解析并实例化为一个`Gradle的Project`对象，然后分析`Project`之间的依赖关系，分析`Project`下的`Task`之间的依赖关系，生成有向无环拓扑结构图`TaskGraph`

  > 每个 gradle 项目都会有一个 build.gradle 文件，该文件是该 项目 构建的入口，可以在这里针对该 项目 进行配置，比如配置版本，需要哪些插件，依赖哪些库等。解析完成后会生成与之对应的Project对象

- **执行阶段:**这是`Task`真正被执行的阶段，Gradle会根据依赖关系决定哪些`Task`需要被执行，以及执行的先后顺序。

  > Task是Gradle中的最小执行单元，我们所有的构建，编译，打包，debug都是执行了某一个(多个)Task，一个Project可以有多个task，Task之间可以互相依赖。

```groovy
// 1.在Settings.gradle配置如下代码监听构建生命周期阶段回调
gradle.buildStarted {
    println "项目构建开始..."
}
// 2.初始化阶段完成
gradle.projectsLoaded {
    println "从settings.gradle解析完成参与构建的所有项目"-->
}

//3.配置阶段开始 解析每一个project/build.gradle
gradle.beforeProject { proj ->
    println "${proj.name} build.gradle解析之前"
}
gradle.afterProject { proj ->
    println "${proj.name} build.gradle解析完成"
}
gradle.projectsEvaluated {
    println "所有项目的build.gradle解析配置完成"-->2.配置阶段完成
}

gradle.getTaskGraph().addTaskExecutionListener{
    //某任务开始执行前
  void beforeExecute(Task task){}
    //某任务执行完成
  void afterExecute(Task task, TaskState state){}
}

gradle.buildFinished {
    println "项目构建结束..."
}
```

![tasks](/imgs/gradle/tasks.png)

- **打印构建阶段task依赖关系及输入输出**

```groovy
afterEvaluate { project ->
    //收集所有project的 task集合
    Map<Project, Set<Task>> allTasks = project.getAllTasks(true)
    // 遍历每一个project下的task集合
    allTasks.entrySet().each { projTasks ->
        projTasks.value.each { task ->
            //输出task的名称 和dependOn依赖
            System.out.println(task.getName());
            for (Object o : task.getDependsOn()) {
                System.out.println("dependOn--> " + o.toString());
            }

//打印每个任务的输入 ， 输出
//for (File file : task.getInputs().getFiles().getFiles()) {
//    System.out.println("input--> " + file.getAbsolutePath());
//}
//
//for (File file : task.getOutputs().getFiles().getFiles()) {
//    System.out.println("output--> " + file.getAbsolutePath());
//}
            System.out.println("----------------------------------");
        }
    }
}
```

```groovy
build
dependOn--> check
dependOn--> assemble
----------------------------------
check
dependOn--> lint
dependOn--> test
----------------------------------
checkDebugManifest
dependOn--> preDebugBuild
----------------------------------
clean
----------------------------------
compileDebugJavaWithJavac
dependOn--> generateDebugSources
dependOn--> preDebugBuild
```

#### Project 工程树

<center><img src="/imgs/gradle/RootProject(ASProj).png" alt="RootProject(ASProj)" style="zoom:40%;" /></center>

每个 项目 都会有一个` build.gradle` 文件，在配置阶段会生成与之对应的`Project`对象，`RootProject `也不例外。而且`RootProject` 可以获取所有的子项目`subProject`，所以可以在 `RootProject` 的 `build.gradle` 文件里对所有 `subProject` 统一配置，比如应用的插件，依赖项等等

```groovy
ASProj/build.gradle
//allprojects:中定义的属性会被应用到所有 moudle 中
//但是为了保证每个项目的独立性，我们一般不会在这里面操作太多共有的东西。
allprojects {
    repositories {
        google()
        jcenter()
    }
}
//应用于所有子项目,可以把所有子项目通用的配置在这里统一设置
//对所有子项目进行遍历，我们可以在闭包里配置、打印、输出或者修改 子Project的属性。
subprojects {project->
  android{
    compileOptions {
        sourceCompatibility = 1.8
        targetCompatibility = 1.8
    }

    kotlinOptions {
        jvmTarget = "1.8"
    }
  }
  dependencies{
     //implementation(...)
  }
}
```

### Gradle 的 Task 和 Transform

#### **Task定义和配置**

 `Task` 是 `Gradle` 项目构建的最小执行单元，Gradle 通过将一个个 Task 串联起来完成具体的构建任务，每个 `Task` 都属于一个 `Project`。
`Gradle` 在构建的过程中，会构建一个 `TaskGraph`任务依赖图，这个依赖图会保证各个 Task 的执行顺序关系。

|    配置项    |                             描述                             |   默认值    |
| :----------: | :----------------------------------------------------------: | :---------: |
|     type     |       基于一个存在的 Task 来创建，和我们的类继承差不多       | DefaultTask |
|    action    |             添加到任务的一个 Action 或者一个闭包             |             |
| description  |                           任务描述                           |             |
|    group     |                           任务分组                           |             |
|   enabled    |            是否可用,如果为false,则该任务会被跳过             |    true     |
|  dependsOn   |   配置任务**依赖关系**，如果A依赖B,则必须B执行完才能执行A    |             |
| mustRunAfter | 设置任务**执行顺序**。 如果任务 A和 B **都存在的场景下, 任务 A 必须先于任务B 执行** |             |
| finalizedBy  | 为任务A添加一个当前**任务结束**后立马执行的任务B。跟**dependsOn**相反 |             |

- **Task Propterty**属性配置

```groovy
//第一种配置方法，创建的时候就配置task的group和description
//description就是个说明，类似对注释
task tinyPngTask(group:'imooc',description:'compress images'){
    println 'this is TinyPngTask'
}

project.tasks.create(name:'tinyPngTask'){
    //第二种配置方式：直接在闭包中配置
    setGroup('imooc')
    setDescription('compress images')
    println 'this is TinyPngTask'
}

//第三种是继承自DefaultTask,需要向project中注册
//一般开发插件的时候会这么写
class TinyPngTask extends DefaultTask{
  @TaskAction
  void run(){
    //.....
  }
}
```

- **Task Actions**执行动作

  `doFirst`给 Task 添加一个执行动作，在该Task的执行阶段，是最先执行

  `doLast`给 Task 添加一个执行动作，在该Task的执行阶段，是最后执行

```groovy
task tinyPngTask(group:'imooc',description:'compress images'){
    println 'this is TinyPngTask'   //1.直接写在闭包里面的，是在配置阶段就执行的
    doFirst {
        println 'task in do first1'  //3.运行任务时，后于first2执行
    }
    doFirst {
        println 'task in do first2' //2.运行任务时，所有actions第一个执行
    }
    doLast {
        println 'task in do last'   //4.运行任务时，会最后一个执行
    }
}
```

- **Task Denpendency**任务关系

  - **dependsOn** - 设置任务**依赖关系**。 执行任务 B 需要任务 A 首先被执行。

  ```groovy
  ASProj/build.gradle
  ---------------------
  task taskA {
      doFirst {
          println 'taskA'
      }
  }
  task taskB {
      doFirst {
          println 'taskB'
      }
  }
  taskB.dependsOn taskA   //B在执行时，会首先执行它的依赖任务A
  ------------------------
  terminal:./gradlew taskB ===>taskA taskB
  ------------------------
  terminal:./gradlew taskA taskB ===>taskA taskB
  ```

  - **mustRunAfter** - 设置任务**执行顺序**。 执行任务 B 不需要执行任务 A. 但如果任务 A和 B **都存在的场景下, 任务 A 必须先于任务B 执行**。

  ```groovy
  ASProj/build.gradle
  ---------------------
  task taskA {
      doFirst {
          println 'taskA'
      }
  }
  task taskB {
      doFirst {
          println 'taskB'
      }
  }
  taskB.mustRunAfter taskA //在一次任务执行流程中，A,B都存在，这里设置的执行顺序才有效
  ----------------------
  terminal:./gradlew taskB ===>taskB
  ----------------------
  terminal:./gradlew taskA taskB ===>taskA taskB
  ```

  - **finalizedBy** - 为任务A添加一个当前**任务结束**后立马执行的任务B。跟**dependsOn**相反

  ```groovy
  ASProj/build.gradle
  ---------------------
  task taskA {
      doFirst {
          println 'taskA'
      }
  }
  task taskB {
      doFirst {
          println 'taskB'
      }
  }
  taskB.finalizedBy taskA   //B执行完成，会执行A
  ------------------------
  terminal:./gradlew taskA taskB ===>taskA taskB
  ------------------------
  terminal:./gradlew taskB ===>taskA taskB
  ```

#### **Transform的定义与配置**

```
A Transform that processes intermediary build artifacts.
For each added transform, a new task is created.
```

如果我们想对编译时产生的 Class 文件，在转换成Dex 之前做一些处理（字节码插桩，替换父类...）。
我们可以通过 `Gradle` 插件来注册我们编写的 `Transform`。注册后的 `Transform` 也会被 Gradle 包装成一个 Gradle Task，这个 `Transform Task` 会在` java compile Task `执行完毕后运行。
一般我们使用 `Transform` 会有下面两种场景

- 我们需要对编译生成的 class 文件做自定义的处理。
- 我们需要读取编译产生的 class 文件，做一些其他事情，但是不需要修改它。
- 比如 Hilt DI框架中会修改 superclass 为特定的 class，
- Hugo耗时统计库会在每个方法中插入代码来统计方法耗时....
- InstantPatch热修复，在所有方法前插入一个预留的函数，可以将有bug的方法替换成下发的方法。
- CodeCheck代码检查，都是使用 transform 来做的。

![transform](/imgs/gradle/transform.png)

#### 如何自定义插件

- 输入输出

  - **TransformInput**是指输入文件的一个抽象，包括：

    **DirectoryInput**集合
     是指以源码的方式参与项目编译的所有目录结构及其目录下的源码文件

    **JarInput**集合
     是指以jar包方式参与项目编译的所有本地jar包和远程jar包（包括aar）

  - **TransformOutputProvider** 通过它可以获取到输出路径等信息

```groovy
public class MyTransform extends Transform {
    @Override
    public String getName() {
        return "MyTransform";
    }
    @Override
    public Set<QualifiedContent.ContentType> getInputTypes() {
        //该transform接受那些内容作为输入参数
        //CLASSES(0x01),
        //RESOURCES(0x02)，assets/目录下的资源，而不是res下的资源
        return TransformManager.CONTENT_CLASS;
    }
    @Override
    public Set<? super QualifiedContent.Scope> getScopes() {
        //该transform工作的作用域--->见下表
        return TransformManager.SCOPE_FULL_PROJECT;
    }
    @Override
    public boolean isIncremental() {
        //是否增量编译
        //基于Task的上次输出快照和这次输入快照对比，如果相同，则跳过
        return false;
    }
    @Override
    public void transform(TransformInvocation transformInvocation) {

      //注意：当前编译对于Transform是否是增量编译受两个方面的影响：
      //（1）isIncremental() 方法的返回值；
      //（2）当前编译是否有增量基础（clean之后的第一次编译没有增量基础，之后的编译有增量基础）
      def isIncremental =transformvocation.isIncremental&&!isIncremental()
       //获取一个能够获取输出路径的工具
      def outputProvider= transformInvocation.outputProvider
      if(!isIncremental){
        //不是增量更新则必须删除上一次构建产生的缓存文件
        outputProvider.deleteAll()
      }
transformInvocation.inputs.each { input ->
            //遍历所有目录
            input.directoryInputs.each { dirInput ->
                println("MyTransform:" + dirInput.file.absolutePath)


                File dest = outputProvider.getContentLocation(directoryInput.getName(),
directoryInput.getContentTypes(), directoryInput.getScopes(), Format.DIRECTORY);
                //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了
                FileUtils.copyDirectory(dirInput.getFile(), dest);
            }

            //遍历所有的jar包(包括aar)
           input.jarInputs.each{file,state->
                  println("MyTransform:" + file.absolutePath)

             File dest = outputProvider.getContentLocation(
                        jarInput.getFile().getAbsolutePath(),
                        jarInput.getContentTypes(),
                        jarInput.getScopes(),
                        Format.JAR);
                //将修改过的字节码copy到dest，就可以实现编译期间干预字节码的目的了
                FileUtils.copyFile(jarInput.getFile(), dest);
            }
        }
    }
}
```

- **Transform Scope作用域**

| 作用域类型         | 描述                                       |
| ------------------ | ------------------------------------------ |
| PROJECT            | 只处理当前项目下的文件                     |
| SUB_PROJECTS       | 只处理子项目下的文件                       |
| EXTERNAL_LIBRARIES | 只处理外部的依赖库                         |
| PROVIDED_ONLY      | 只处理本地或远程以provided形式引入的依赖库 |
| TESTED_CODE        | 测试代码                                   |

- **自定义Transform工作流程**

<img src="/imgs/gradle/transform2.jpeg" alt="transform2" style="zoom:50%;" />

- **插件开发模式**

| 类型         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| Buildscript  | 把插件写在 build.gradle 文件中，一般用于简单的逻辑，只在该 build.gradle 文件中可见 |
| buildSrc模块 | 将插件源代码放在 /buildSrc/src/main/groovy 中，只对该项目中可见，适用于逻辑较为复杂 |
| **独立项目** | 独立的 Groovy 和 Java 项目，可以把这个项目打包成 Jar 文件包，一个 Jar 文件包还可以包含多个插件入口，将文件包发布到托管平台上，供其他人使用 |

![buildsrc](/imgs/gradle/buildsrc.jpeg)

```groovy
apply plugin: 'groovy'
repositories {
    google()
    jcenter()
}
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    //3.使用android plugin.相当于使用jetpack库
    implementation 'com.android.tools.build:gradle:3.4.2'
    //2.使用gradle api，相当于使用android sdk
    implementation gradleApi()
    //1.使用groovy库
    implementation localGroovy()

    implementation 'org.javassist:javassist:3.27.0-GA'
}
sourceCompatibility = "1.8"
targetCompatibility = "1.8"
```

#### Extension扩展配置

它的作用就是通过实现自定义的 bean对象，可以在 Gradle 脚本中增加类似 'android' 这样命名空间的配置，Gradle 可以识别这种配置，并读取里面的配置内容。通常使用ExtensionContainer来创建并管理Extension

```groovy
class TinyPngExt{
    List<Integer> whiteList;
    String apiKey;
}

class TinyPngPlugin implements Plugin<Project> {
    @Override
    void apply(Project project) {
        //1.获取ExtensionContainer对象
        def extContainer = project.extensions

      //2.注册扩展对象
      //name：要创建的Extension的名字，可以是任意符合命名规则的字符串，不能与已有的重复，否则会抛异常；
      //type：该Extension的类class类型；
      //constructionArguments：类的构造函数参数值
      extContainer.create("tinyPng", TinyPngExt.class, Object... constructionArguments))

    }
}
```

#### gradle动态传参

这有很多用途，比如可以控制要不要集成`HiDebugTool`开发者工具

```groovy
dependencies{
  //读取gradle参数 决定要不要集成hi_debugtool
  //我们需要哪些功能，就可以通过打包的参数来决定
  if(hasProperty('debugTool')&&getProperty('debugTool')=='1'){
      implementation project(path: ':hi_debugtool')
  }
}
```

我们判断的条件是是否有 `debugTool==1` 属性

```groovy
//不会集成debugTool
./gradlew assembleDebug
------------------------
//会集成debugToo
//命令行中 -P 的意思是为 Project 指定 K-V 格式的属性键值对，使用格式为 -PK=V
./gradlew  -PdebugTool=1 assembleDebug
```

#### 常用的构建命令

```groovy
./gradlew clean    //移除所有的编译输出文件，比如apk

./gradlew build  //构建任务，相当于同时执行了check任务和assemble任务

./gradlew check   //执行lint检测编译。

./gradlew assemble //可以编译出release包和debug包

./gradlew assembleRelease  //打 release 包

./gradlew assembleDebug  //app module 打 debug 包

./gradlew installDebug  //安装 app 的 debug 包到手机上
```

#### 字节码技术三剑客 AspectJ,Javassist,Asm

<img src="/imgs/gradle/apo.png" alt="apo" style="zoom:50%;" />

- Apt :代表框架：DataBinding,Dagger2, ButterKnife, EventBus3 、DBFlow。通过继承自`AbstractProcessor`，在`Process`方法内来完成类的生成
- [AspectJ](https://msd.misuland.com/pd/3181438578597039256)：防止重复点击
- [ASM](https://www.jianshu.com/p/eb15847cae80):实现方法耗时检测
- [javassist使用指南](https://www.jianshu.com/p/886f9b2cbfdb)



####  字节码插桩技术

- 向工程内所有Activity的onCreate函数内插入Toast代码块
- 工程内源码参与编译的.class，以jar(aar)参与编译.class。 都需要修改

```groovy
void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        def outputProvider = transformInvocation.outputProvider
        //outputProvider.deleteAll()

        transformInvocation.inputs.each { input ->
            input.directoryInputs.each { dirInput ->
                //遍历处理没一个文件夹下面的class文件
                handleDirectory(dirInput.file)
                // 将输入的所有目录复制到output指定目录
                def dest = outputProvider.getContentLocation(dirInput.name, dirInput.contentTypes, dirInput.scopes, Format.DIRECTORY)
                FileUtils.copyDirectory(dirInput.file, dest)
            }

            input.jarInputs.each { jarInput ->
                // 重新计算输出文件jar的名字（同目录 copyFile 会冲突）
                def jarName = jarInput.name
                def md5Name = DigestUtils.md5Hex(jarInput.file.getAbsolutePath())
                if (jarName.endsWith(".jar")) {
                    jarName = jarName.substring(0, jarName.length() - 4)
                }

                // 生成输出路径
                def dest = outputProvider.getContentLocation(md5Name + jarName, jarInput.contentTypes, jarInput.scopes, Format.JAR)
                //处理jar包
                def output = handleJar(jarInput.file)
                // 将输入内容复制到输出
                FileUtils.copyFile(output, dest)
            }
        }
        pool.clearImportedPackages()
    }

    static File handleJar(File jarFile) {
        //添加类搜索路径,否则下面pool.get()查找类会找不到
        pool.appendClassPath(jarFile.absolutePath)
        //通过JarFile，才能获取得到jar包里面一个个的子文件
        def inputJarFile = new JarFile(jarFile)
        //通过它进行迭代遍历
        Enumeration enumeration = inputJarFile.entries()

        //jar包里面的class修改之后，不能原路写入jar。会破坏jar文件结构
        //需要写入单独的一个jar中
        def outputJarFile = new File(jarFile.parentFile, "temp_" + jarFile.name)
        if (outputJarFile.exists()) outputJarFile.delete()
        JarOutputStream jarOutputStream = new JarOutputStream(new BufferedOutputStream(new FileOutputStream(outputJarFile)))

        while (enumeration.hasMoreElements()) {
            JarEntry jarEntry = (JarEntry) enumeration.nextElement()
            String entryName = jarEntry.getName()
            InputStream inputStream = inputJarFile.getInputStream(jarEntry)

            //构建一个个需要写入的JarEntry，并添加到jar包
            JarEntry zipEntry = new JarEntry(entryName)
            jarOutputStream.putNextEntry(zipEntry)

            println("entryName:" + entryName)
            if (!shouldModifyClass(entryName)) {
                //如果这个类不需要被修改，那也需要向output jar 写入数据，否则class文件会丢失
              jarOutputStream.write(IOUtils.toByteArray(inputStream))
                inputStream.close()
                continue
            }
            def ctClass = modifyClass(inputStream)
            def byteCode = ctClass.toBytecode()
            ctClass.detach()
            inputStream.close()

            println("code length:" + byteCode.length)
            //向output jar 写入数据
            jarOutputStream.write(byteCode)
            jarOutputStream.flush()
        }
        inputJarFile.close()
        jarOutputStream.closeEntry()
        jarOutputStream.flush()
        jarOutputStream.close()

        return outputJarFile
    }

    void handleDirectory(File dir) {
        //将当前路径加入类池,不然找不到这个类
        println("handleDirectory:" + project.rootDir)
        pool.appendClassPath(dir.absolutePath)
        if (dir.isDirectory()) {
            dir.eachFileRecurse { File file ->
                String filePath = file.name
                println("filePath:" + file.absolutePath)
                //确保当前文件是class文件，并且不是系统自动生成的class文件
                if (shouldModifyClass(filePath)) {
                    def inputStream = new FileInputStream(file)
                    def ctClass = modifyClass(inputStream)
                    ctClass.writeFile(filePath)
                    ctClass.detach()
                }
            }
        }
    }

    //通过文件输入流，使用javassist加载出ctclass类
    //从而操作字节码
    static CtClass modifyClass(InputStream inputStream) {
        //之所以使用inputStream ，是为了兼顾jar包里面的class文件的处理场景
        //从jar包中获取一个个的文件，只能通过流
        ClassFile classFile = new ClassFile(new DataInputStream(new BufferedInputStream(inputStream)))
        println("found activity:" + classFile.name)
        CtClass ctClass = pool.get(classFile.name)
        //解冻，意思是指如果该class之前已经被别人加载并修改过了。默认不允许再次被编辑修改
        if (ctClass.isFrozen()) {
            ctClass.defrost()
        }

        //构造onCreate方法的Bundle入参
        CtClass bundle = pool.getCtClass("android.os.Bundle")
        CtClass[] params = Arrays.asList(bundle).toArray()
        //通过ctClass
        def method = ctClass.getDeclaredMethod("onCreate", params)
        def message = classFile.name
      //向方法的最后一行插入toast，message为当前类的全类名
    method.insertAfter("android.widget.Toast.makeText(this," + "\"" + message + "\"" + ",Toast.LENGTH_SHORT).show();")

        return ctClass
    }

//检验文件的路径以判断是否应该对它修改
//不是我们包内的class，不是activity.class 都不修改
    static boolean shouldModifyClass(String filePath) {
        return (filePath.contains("org/devio/as/proj")
                && filePath.endsWith("Activity.class")
                && !filePath.contains('R$')
                && !filePath.contains('$')
                && !filePath.contains('R.class')
                && !filePath.contains("BuildConfig.class"))
    }
```