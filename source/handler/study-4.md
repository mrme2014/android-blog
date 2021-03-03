---
title: 热修复实战
---
## 走进Android热修复世界

- 主流热修复方式
- 动态加载dex实现热修复hexo

### 主流热修复方式

| 实现方式   | 优劣势                                                   | 代表           |
| ---------- | -------------------------------------------------------- | -------------- |
| 底层替换   | 底层替换方案限制颇多，但时效性最好，加载轻快，立即见效   | AndFix、Sophix |
| 类加载方案 | 类加载方案时效性差，需要重新冷启动才能见效，但修复范围广 | QZone、Tinker  |

| 能力\方案  | Tinker               | QZone              | AndFix | Sophix             |
| ---------- | -------------------- | ------------------ | ------ | ------------------ |
| Dex修复    | 冷启动               | 冷启动             | 不支持 | 即时，冷启动       |
| 资源替换   | 差量包，需要合成     | 差量包，需要合成   | 不支持 | 差量包，不用合成   |
| So替换     | 替换接口，开发不透明 | 插桩实现，开发透明 | 不支持 | 插桩实现，开发透明 |
| 成功率     | 高                   | 较高               | 一般   | 高                 |
| 全版本支持 | 支持                 | 支持               | 支持   | 支持               |
| 性能损耗   | 较大                 | 较大               | 较小   | 较低               |
| 补丁包大小 | 较小                 | 较大               | 较大   | 较小               |
| 接入复杂度 | 复杂                 | 较低               | 较高   | 较低               |
| SDK体积    | 较大                 | 较小               | 较小   | 较小               |
|            |                      |                    |        |                    |

### 动态加载dex实现热修复

<img src="/imgs/handler/hotfix.png">

- step1 找到classloader中的dexPathList对象

```java
 ClassLoader loader = context.getClassLoader();
 Field pathListField = findField(loader, "pathList");
 Object dexPathList = pathListField.get(loader); 
```

- step2 执行dexPathList.makeDexElements方法，生成包含dex的element数组

```java
private static Object[] makeDexElements(Object dexPathList, ArrayList<File> files,ClassLoader loader) {
        Method makeDexElements = findMethod(dexPathList, "makeDexElements", List.class, File.class, List.class, ClassLoader.class);
        return (Object[]) makeDexElements.invoke(dexPathList, files, null, exceptions, loader);
    }
```

- Step3 向dexPathList合并新加载进来dex数组 

```java
private static void expandFieldArray(Object instance, String fieldName,Object[] extraElements) {
       Field dexElementsFiled = findField(instance, "dexElements");
        Object[] original = (Object[]) dexElementsFiled.get(instance);
         //构建新的element数组，先把修复bug的dexelement数组放进去，再把原来的放进去，最后更改到dexpathList对象的dexelement
        //getComponentType得到数组中元素的类型 ，我们知道是Element,但是无法直接访问到该类。
        Object[] combined = (Object[]) Array.newInstance(original.getClass().getComponentType(), original.length + extraElements.length);
        System.arraycopy(extraElements, 0, combined, 0, extraElements.length);
        System.arraycopy(original, 0, original, extraElements.length, original.length);
        dexElementsFiled.set(instance, combined);
    }

```

- **class打包成dex 命令**
  - 在你的环境变量中添加dx脚本的路径`/Users/***/Library/Android/sdk/build-tools/30.0.0`
  - 然后在as 中打开terminal.输入一下命令，output是指输出的dex文件路径及名称，如下指定会生成在项目根目录下，后面跟的是需要打包的class文件，或者目录。

```java
dx --dex --no-strict --output=patch.dex app/build/intermediates/javac/debug/classes/org/devio/as/proj/hi_handler/HotFixTest.class

```



