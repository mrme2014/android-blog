---
title: 走进Android热修复世界
---

## 目录

- 主流热修复方式
- 动态加载dex实现热修复

### 主流热修复方式
<table border="1">
  <tr bgcolor="#999999">
    <th width="310">实现方式</th>
    <th width="310">实现方式</th>
    <th width="310">代表</th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td>底层替换</td>
    <td>底层替换方案限制颇多，但时效性最好，加载轻快，立即见效</td>
    <td>AndFix、Sophix</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>类加载方案</td>
   <td>类加载方案时效性差，需要重新冷启动才能见效，但修复范围广</td>
   <td>QZone、Tinker</td>
  </tr>
</table>
<br></br>
<table border="1">
  <tr bgcolor="#999999">
    <th width="310">能力\方案</th>
    <th width="310">Tinker</th>
    <th width="310">QZone</th>
    <th width="310">AndFix</th>
    <th width="310">Sophix</th>
  </tr>
  <tr  bgcolor="#ffffff">
    <td>Dex修复</td>
    <td>冷启动</td>
    <td>冷启动</td>
    <td>不支持</td>
    <td>即时，冷启动</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>资源替换</td>
   <td>差量包，需要合成</td>
   <td>差量包，需要合成</td>
   <td>不支持</td>
   <td>差量包，不用合成</td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>So替换</td>
   <td>替换接口，开发不透明</td>
   <td>插桩实现，开发透明</td>
   <td>不支持</td>
   <td>插桩实现，开发透明</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>成功率</td>
   <td>高</td>
   <td>较高</td>
   <td>一般</td>
   <td>高</td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>全版本支持</td>
   <td>支持</td>
   <td>支持</td>
   <td>支持</td>
   <td>支持</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>性能损耗</td>
   <td>较大</td>
   <td>较大</td>
   <td>较大</td>
   <td>较小</td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>补丁包大小</td>
   <td>较小</td>
   <td>较大</td>
   <td>较大</td>
   <td>较小</td>
  </tr>
  <tr  bgcolor="#eeeeee">
   <td>接入复杂度</td>
   <td>复杂</td>
   <td>较大</td>
   <td>较高</td>
   <td>较小</td>
  </tr>
  <tr  bgcolor="#ffffff">
   <td>SDK体积</td>
   <td>较大</td>
   <td>较小</td>
   <td>较小</td>
   <td>较小</td>
  </tr>
</table>
<br></br>

### 动态加载dex实现热修复

<center><img src="/imgs/handler/hotfix.png" style="zoom:50%;" width="1600"></center>

- step1 找到classloader中的dexPathList对象

```java
 ClassLoader loader = context.getClassLoader();
 Field pathListField = findField(loader, "pathList");
 Object dexPathList = pathListField.get(loader); 
```

- step2 执行dexPathList.makeDexElements方法，生成包含dex的element数组

```java
private static Object[] makeDexElements(Object dexPathList, ArrayList<File> files,ClassLoader loader) {
        Method makeDexElements = findMethod(dexPathList, "makeDexElements", 
                     List.class, File.class, List.class, ClassLoader.class);
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
        Object[] combined = (Object[]) Array.newInstance(original.getClass().getComponentType(),
                           original.length + extraElements.length);
        System.arraycopy(extraElements, 0, combined, 0, extraElements.length);
        System.arraycopy(original, 0, original, extraElements.length, original.length);
        dexElementsFiled.set(instance, combined);
    }

```

- **class打包成dex 命令**
  - 在你的环境变量中添加dx脚本的路径`/Users/***/Library/Android/sdk/build-tools/30.0.0`
  - 然后在as 中打开terminal.输入一下命令，output是指输出的dex文件路径及名称，如下指定会生成在项目根目录下，后面跟的是需要打包的class文件，或者目录。

```java
dx --dex --no-strict --output=patch.dex 
app/build/intermediates/javac/debug/classes/org/devio/as/proj/hi_handler/HotFixTest.class

```
