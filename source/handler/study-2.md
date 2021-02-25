---
title: 深入理解Android类加载机制
---

## 目录
- 什么是双亲委派
- 双亲委派下的Class文件加载流程
- Android中主要的类加载器
- PathClassLoader & DexClassLoader 到底有何异同？
- Class文件不同的加载方式
- Class文件加载源码分析

  

### 双亲委派

- **什么是双亲委派**

  - 加载`.class`文件时，以递归的形式逐级向上委托给父加载器parentClassLoader去加载，如果加载过了，就不用再加载一遍。

  - 如果父加载器也没加载过，则继续委托给父加载器去加载，一直到这条链路的顶级，顶级classloader判断如果没加载过，则尝试加载，加载失败,则逐级向下交还调用者来加载。

<center><img src="/img/handler/classloader2.png" style="zoom:50%;"></center>

- **双亲委派是如何实现的**

  ```java
  val customClassloader = new ClassLoader(){
      @override
      public Class<?> loadClass(String name) throws ClassNotFoundException {
      }
  }
  
  class ClassLoader{
      protected ClassLoader() {
           this(checkCreateClassLoader(),getSystemClassLoader());//pathclassloader
      }
      protected ClassLoader(ClassLoader parent) {
           this(checkCreateClassLoader(), parent);
      }
    
      private static ClassLoader createSystemClassLoader() {
          String classPath = System.getProperty("java.class.path", ".");
          String librarySearchPath = System.getProperty("java.library.path", "");
          return new PathClassLoader(classPath, librarySearchPath, BootClassLoader.getInstance());
      }
    
      protected Class<?> loadClass(String name, boolean resolve)throws ClassNotFoundException{
              //先检查是否已经加载过--findLoaded
               Class<?> c = findLoadedClass(name);
               //如果自己没加载过，存在父类，则委托父类
               if (parent != null) {
                     c = parent.loadClass(name, false);
                   } 
               if (c == null) {
                    //如果父类也没加载过，则尝试本级classLoader加载
                     c = findClass(name);
                   }
              }   
              return c;
      }
  }
  ```

- **双亲委派的作用**

  - 防止同一个`.class`文件重复加载。
  - 对于任意一个类确保在虚拟机中的唯一性。由**加载它的类加载器**和这个**类的全类名一同确立其在Java虚拟机中的**唯一性。
  - 保证系统类`.class`文件不能被篡改。通过委托方式可以保证系统类的加载逻辑不被篡改。



### Android中主要的类加载器

- PathClassLoader，复杂的加载系统类和应用程序的类，通常用来加载已安装的apk的dex文件，实际上外部存储的dex文件也能加载。
- DexClassLoader:可以加载dex文件以及包含dex的压缩文件（apk,dex,jar,zip)。
- BaseDexClasssLoader： 实现应用层类文件的加载，而真正的加载逻辑委托给PathList来完成。

<center><img src="/img/handler/lassloader.png" style="zoom:50%;"></center>

**PathClassLoader & DexClassLoader有什么不同?**

```java
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
         super(dexPath, null, librarySearchPath, parent);
    }
}
```

```java
public class DexClassLoader extends BaseDexClassLoader {
    //dexPath：dex文件以及包含dex的apk文件或jar文件的路径，多个路径用文件分隔符分隔，默认文件分隔符为‘：’
    //optimizedDirectory：Android系统将dex文件进行优化后所生成的ODEX文件的存放路径，该路径必须是一个内部存储路径。
                         //PathClassLoader中使用默认路径“/data/dalvik-cache”，
                         //而DexClassLoader则需要我们指定ODEX优化文件的存放路径。
   //librarySearchPath：所使用到的C/C++库存放的路径
   //parent：这个参数的主要作用是保留java中ClassLoader的委托机制
   public DexClassLoader(String dexPath, String optimizedDirectory, String librarySearchPath, 
   ClassLoader parent) {
        super(dexPath, null, librarySearchPath, parent);
    }
}
```



### 双亲委派下的Class文件加载流程

```java
class BaseDexClassLoader extends ClassLoader{
  public BaseDexClassLoader(String dexPath, File optimizedDirectory,String librarySearchPath, 
                            ClassLoader parent){
     //会从我们给定的dexPath加载出所有的dex文件,存储起来
     this.pathList = new DexPathList(this, dexPath, librarySearchPath, null, false);
  }
  
  //可以说baseDexClassLoader是实现了APP中class文件的加载工作
  protected Class<?> findClass(String name) throws ClassNotFoundException {
        //而它又将具体的实现委托给了pathList
        Class c = pathList.findClass(name, suppressedExceptions);
        if (c == null) {
            //如果没有从dex文件中 找到class,则抛出非常 熟悉的异常
            //class not found on DexPathList[[zip file "/data/app/***-1/base.apk"],
            // nativeLibraryDirectories=[/vendor/lib64, /system/lib64]
            ClassNotFoundException cnfe = new ClassNotFoundException(
                    "Didn't find class \"" + name + "\" on path: " + pathList);
            throw cnfe;
        }
        return c;
    }
}
```

```java
class DexPathList{
  public DexPathList(ClassLoader definingContext, String dexPath,
                     String librarySearchPath, File optimizedDirectory, boolean isTrusted) {
     //dexPath="/data/data/**/classes.dex:/data/data/**/class1.dex/data/data/**/class2.dex"
      //根据传递的dexpath加载出所有dex文件路径
      this.dexElements = makeDexElements(splitDexPath(dexPath), optimizedDirectory,
                                 suppressedExceptions, definingContext, isTrusted);
      //加载APP的动态库
      this.nativeLibraryDirectories = splitPaths(librarySearchPath, false); 
      //加载系统的动态库
      this.systemNativeLibraryDirectories =splitPaths(System.getProperty("java.library.path"), true);
      ......
  }
    
    private static Element[] makeDexElements(List<File> files,  optimizedDirectory, 
                      List<IOException> suppressedExceptions, ClassLoader loader, boolean isTrusted) {
      Element[] elements = new Element[files.size()];
      int elementsPos = 0;
      for (File file : files) {
          if (file.isDirectory()) { 
              elements[elementsPos++] = new Element(file);
          } else if (file.isFile()) {
              String name = file.getName();
              DexFile dex = null;
              //如果文件路径以.dex结尾，则直接加载文件内容
              if (name.endsWith(DEX_SUFFIX)) {
                  try {
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                      if (dex != null) {
                          elements[elementsPos++] = new Element(dex, null);
                      }
                  } catch (IOException suppressed) {
                      System.logE("Unable to load dex file: " + file, suppressed);
                      suppressedExceptions.add(suppressed);
                 }
              } else {
                  try {
                     //如果是jar,zip等文件类型，则需要先
                      dex = loadDexFile(file, optimizedDirectory, loader, elements);
                  } catch (IOException suppressed) {
                      suppressedExceptions.add(suppressed);
                  }

                   if (dex == null) {
                      elements[elementsPos++] = new Element(file);
                  } else {
                      elements[elementsPos++] = new Element(dex, file);
                  }
              }
              if (dex != null && isTrusted) {
                dex.setTrusted();
              }
          } else {
              System.logW("ClassLoader referenced unknown path: " + file);
          }
      }
      if (elementsPos != elements.length) {
          elements = Arrays.copyOf(elements, elementsPos);
      }
      return elements;
    }
    //从这里可以看出 optimizedDirectory 不同,  DexFile对象构造方式不同
    // 我们继续看看 optimizedDirectory 在 DexFile 中的作用：
    private static DexFile loadDexFile(File file, File optimizedDirectory)
        throws IOException {
    if (optimizedDirectory == null) {
        return new DexFile(file);
    } else {
        String optimizedPath = optimizedPathFor(file, optimizedDirectory);
        return DexFile.loadDex(file.getPath(), optimizedPath, 0);
      }
   }  
}
```

```java
class DexFile{
  public DexFile(File file) throws IOException {
    this(file.getPath());
}
/**
 * 从给定的File对象打开一个DEX文件。这通常是一个ZIP/JAR 文件，其中包含“classes.dex”。
 * VM将在/data/dalvik-cache中生成相应文件的名称，然后打开它。
 * @param fileName 引用实际DEX文件的File对象
 * @throws IOException 找不到文件或*打开文件缺少访问权限，回抛出io异常
 */
public DexFile(String fileName) throws IOException {
    mCookie = openDexFile(fileName, null, 0);
    mFileName = fileName;
    guard.open("close");
    //System.out.println("DEX FILE cookie is " + mCookie + " fileName=" + fileName);
}

/**
 * 打开一个DEX文件。返回的值是VM cookie
 * @param sourceName Jar或APK文件包含“ classes.dex”。
 * @param outputName 包含优化形式的DEX数据的文件。
 * @param flags 启用可选功能。
 */
private DexFile(String sourceName, String outputName, int flags) throws IOException {
    if (outputName != null) {
        try {
            String parent = new File(outputName).getParent();
            if (Libcore.os.getuid() != Libcore.os.stat(parent).st_uid) {
                throw new IllegalArgumentException("Optimized data directory " + parent
                        + " is not owned by the current user. Shared storage cannot protect"
                        + " your application from code injection attacks.");
            }
        } catch (ErrnoException ignored) {
            // assume we'll fail with a more contextual error later
        }
    }
    mCookie = openDexFile(sourceName, outputName, flags);
    mFileName = sourceName;
    guard.open("close");
}

static public DexFile loadDex(String sourcePathName, String outputPathName,
    int flags) throws IOException {
    return new DexFile(sourcePathName, outputPathName, flags);
}

//打开dex的native方法，/art/runtime/native/dalvik_system_DexFile.cc
private static native long openDexFileNative(String sourceName, String outputName, int flags);
//打开一个DEX文件。返回的值是VM cookie。 失败时，将引发IOException。
private static long openDexFile(String sourceName, String outputName, int flags) throws IOException {
    // Use absolute paths to enable the use of relative paths when testing on host.
    return openDexFileNative(new File(sourceName).getAbsolutePath(),
                                (outputName == null) ? null : new File(outputName).getAbsolutePath(),
                                flags);
}
}
```

从注释当中就可以看到 `new DexFile(file)` 的dex输出路径只能为 `/data/dalvik-cache`，而 `DexFile.loadDex()` 的 `dex` 输出路径为自己输入的`optimizedDirectory` 路径。

<center><img src="/img/handler/dexfile.png" style="zoom:50%;"></center>

### Class文件加载

类的加载指的是将类的.class文件中的二进制数据读入到内存中，将其放在运行时数据区的方法区内，然后在堆区创建一个java.lang.Class对象，用来封装类在方法区内的数据结构,并且提供了访问方法区内的数据结构的方法。

- 通过Class.forName()方法动态加载
- 通过ClassLoader.loadClass()方法动态加载

类加载分为三个步骤 :1.装载（Load）2.链接（Link）3.初始化（Initialize）

<center><img src="/img/handler/classloader3.png" style="zoom:50%;"></center>

- **1.装载（load）查找并加载类的二进制数据（查找和导入Class文件)**

  - 通过一个类的全限定名来获取其定义的二进制字节流。
  - 将这个字节流转化为方法区的运行时数据结构。
  - 在Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口。

- **2.链接**

  - **验证**：确保被加载的类的正确性

    - 文件格式验证：验证字节流是否符合Class文件格式的规范；例如：是否以0xCAFEBABE开头、主次版本号是否在当前虚拟机的处理范围之内、常量池中的常量是否有不被支持的类型。
    - 元数据验证：对字节码描述的信息进行语义分析，以保证其描述的信息符合Java语言规范的要求；例如：这个类是否有父类，除了java.lang.Object之外。
    - 字节码验证：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。

  - **准备**：为类的静态变量分配内存，并将其初始化为默认值

    - 这时候进行内存分配的仅包括类变量（static），而不包括实例变量，实例变量会在对象实例化时随着对象一块分配在Java堆中。
    - 这里所设置的初始值通常情况下是数据类型默认的零值（如0、0L、null、false等），而不是被在Java代码中被显式地赋予的值。

    ```java
    class MainActivity{
       //在准备阶段他得值为默认值0，初始化阶段才会被赋值为3.
       
       //因为把value赋值为3的putlic static语句在编译后的指令是在类构造器<clinit>（）方法之中被调用的
       // 所以把value赋值为3的动作将在初始化阶段才会执行。
       static int value = 3；//0x0001
       
       int value2=3;//随着对象实例化的时候，才会被赋值
      
      static void test(){
          value2 = 100;//静态方法为什么不能访问非静态变量？
      }
    }
    ```

  - **解析**：把类中的符号引用转换为直接引用

    - 解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程，解析动作主要针对类、接口、字段、类方法、接口方法、方法类型。符号引用就是一组符号来描述目标，可以是任何字面量。
      **直接引用**就是直接指向目标的内存地址指针

- 3.**初始化：执行类的<clint>方法，对类的静态变量，静态代码块执行初始化操作**（不是必须的）

  - **类的初始化**
    - 创建类的实例，也就是new一个对象
    - 访问某个类或接口的静态变量，或者对该静态变量赋值
    - 调用类的静态方法
    - 反射Class.forName("android.app.ActivityThread")
    - 初始化一个类的子类（会首先初始化子类的父类）
    - JVM启动时标明的启动类，即文件名和类名相同的那个类

### Class.forName & ClassLoader.loadClass有何不同

- `class.forName()`前者除了将类的`.class`文件加载到jvm中之外，还会对类进行解释，执行类中的static块。注意这里的静态块指的是在类初始化时的一些数据，但是`classloader`却没有。`Class.forName()`方法实际上也是调用的CLassLoader来实现的, 内部实际调用的方法是

```java
class Class<T>{
  public static Class<?> forName(String className)
                throws ClassNotFoundException {
        Class<?> caller = Reflection.getCallerClass();
        //如果实在MainActivity中调用的Class.forName,那么caller就等于MainActivity的class对象
        //ClassLoader.getClassLoader(caller)：得到当前类的加载器，
        //到这你会发现forNanme其实也是使用的ClassLoader类加载器加载的。
        return forName(className, true, ClassLoader.getClassLoader(caller));
    }
}
  //className：表示我们要加载的类名android.app.ActivityThread
  //initialize：指Class被加载后是不是必须被初始化。
  //不初始化就是不执行static的代码即静态代码，在这里默认为true，也就是默认执行类的初始化。
  //加载该类的classloader
  @CallerSensitive
  public static Class<?> forName(String name, boolean initialize,ClassLoader loader)
        throws ClassNotFoundException
    {
        if (loader == null) {
            loader = BootClassLoader.getInstance();
        }
        Class<?> result;
        try {
            result = classForName(name, initialize, loader);
        } catch (ClassNotFoundException e) {
            Throwable cause = e.getCause();
            if (cause instanceof LinkageError) {
                throw (LinkageError) cause;
            }
            throw e;
        }
        return result;
    }
static native Class<?> classForName(String className, boolean shouldInitialize,
                       ClassLoader classLoader) throws ClassNotFoundException; 
```

```java
Class BaseDexClassLoader extends ClassLoader{
  protected Class<?> findClass(String name) throws ClassNotFoundException {
    return  pathList.findClass(name, suppressedExceptions);
  }
}
```






