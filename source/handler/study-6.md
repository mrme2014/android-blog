---
title: Tinker实现原理剖析
---

## 目录
- Tinker工作原理说明
- 差量包全量合成与dex2oat在10.0的适配原理
- 全量包动态加载与版本适配原理



### Tinker工作原理简述

- **简述热修复实现原理**
  - 阶段1.使用bsdiff对新旧apk做差分异，得到差量化产物patch.apk补丁文件
  - 阶段2.使用dexpatch把下载的差量patch.apk补丁文件中的dex,so,res和基线版本做全量合并，dex和so文件全量合并后得到完整的tinker_NClass.apk,res文件全量合并后得到完成的resource.apk
  - 阶段3.动态加载tinker_NClass.apk进行dex插队实现类修复，动态加载resource.apk并反射替换context中的assetManger实现资源文件的更新
  - 拓展:[bsdiff & bspatch实现APP增量更新](https://www.jianshu.com/p/f70e31755bcd)

<img src="{{root}}/imgs/handler/tinker4.png" alt="tinker4" />

- **patch.apk补丁包结构**

  包含差量后的dex文件，res文件，还有libs/so文件，除此之外还有些补丁的描述信息文件。`dex_meta.txt`记录了补丁包中所有dex文件的信息。`res_meta.txt`记录了资源文件的信息。`package_meta.txt`记录了补丁包的信息。

  ```java
  Sun Jul 12 12:58:57 CST 2020
  platform=all
  NEW_TINKER_ID=tinker_id_patch2-1.0.0
  TINKER_ID=tinker_id_patch-1.0.0
  is_protected_app=0
  patchVersion=1.0
  ```

<center><img src="{{root}}/imgs/handler/tinker5.png" alt="tinker5" style="zoom: 50%;" /></center>

- **增量升级**
  - 增量升级是以两个应用版本之间的差异来生成补丁的，所以必须对所发布的每一个线上版本都和最新的版本作差分，以便所有版本的用户都可以差分升级。
  - 增量升级成功的前提是，用户手机端必须存在完整apk文件。如果运行时无法获取已安装的apk文件，则无法进行增量升级；如果已安装apk内容有过修改(比如破解版apk)，这样也是无法进行增量升级的，为了防止补丁合成错误。最好通过md5 ，version,crc(数据完整性校验，可判断文件数据是否被修改过) 对patch包和已安装包进行完整性和一致性的校验。

### patch补丁合成与10.0适配

- **开始把下发下来的patch.apk中包含的 dex,so与已安装包的base.apk中相应的文件分别进行全量合成操作，合成新的tinker_NClass.apk**.入口方法

```java
TinkerInstaller.onReceiveUpgradePatch(getApplicationContext(), Environment.getExternalStorageDirectory().getAbsolutePath() + "/patch_signed_7zip.apk");
```

- **开始做合成之前的一堆条件校验**

```java
class DefaultPatchListener{
   public int onPatchReceived(String path) {
        //path = patch.apk的文件地址
        final File patchFile = new File(path);
        final String patchMD5 = SharePatchFileUtil.getMD5(patchFile);  
        //1. 判断当前安装包是否开启tinkerEnable,也就代表是否可以进行patch操作
        //2. 判断patch文件是否存在
        //3. 判断当前运行是否在主进程
        //4. 判断当前patchService服务进程是否运行中
        //5. 判断当前是否开启了davik jit 即时编译
        //6. 判断当前patch是否已经合成过了 
        //7. 判断当前pacth合成失败，是否以达到最大次数   
        final int returnCode = patchCheck(path, patchMD5);
        if (returnCode == ShareConstants.ERROR_PATCH_OK) {
            runForgService();
            TinkerPatchService.runPatchService(context, path);
        } else {
            Tinker.with(context).getLoadReporter().onLoadPatchListenerReceiveFail(new File(path), returnCode);
        }
        return returnCode;
    }
}
```

- **启动新的TinkerPatchService进程专门用于合并patch文件，防止影响主进程**

```java
class TinkerPatchService extends IntentService{
  @Override
    protected void onHandleIntent(Intent intent) {
        doApplyPatch(this, intent);
    }
  
  private static void doApplyPatch(Context context, Intent intent) {
        //该service运行在别进程，无法debug,可以在源码中键入下面的语句
        //当程序运行到这里的时候，就会停下来，等待debug attach
        Debug.waitForDebugger();

        if (!sIsPatchApplying.compareAndSet(false, true)) {
            //cas操作， 如果正在进行patch，则不处理该次请求
            return;
        }

        Tinker tinker = Tinker.with(context);

        //storage/emulated/0/patch_signed_7zip.apk
        String path = getPatchPathExtra(intent);
        File patchFile = new File(path);
        //记录开始的时间值，这个值即便系统休眠了，也可以累加
        long begin = SystemClock.elapsedRealtime();
        PatchResult patchResult = new PatchResult();
        //进一步 开启合成,result代表文件合成成功，or 失败
       
        boolean result = upgradePatchProcessor.tryPatch(context, path, patchResult);
         //计算本次合成花费的时间
        long  cost = SystemClock.elapsedRealtime() - begin;
        
        patchResult.isSuccess = result;
        patchResult.rawPatchFilePath = path;
        patchResult.costTime = cost;

        //开启新的SampleTinkerResultService服务,用于关闭pacth服务进程，处理pacth结果，删除pacth文件，以及判断需不需要重启主进程
        AbstractResultService.runResultService(context, patchResult, getPatchResultExtra(intent));
   
        //变更状态
        sIsPatchApplying.set(false);
    }
  
  //该放在是在TinkerApplication中初始化的时候调用，传递过来的
  //他俩的值分别为开启新的UpgradePatch.class,SampleTinkerResultService.class,
  public static void setPatchProcessor(AbstractPatch upgradePatch, Class<? extends AbstractResultService> serviceClass) {
        upgradePatchProcessor = upgradePatch;
        resultServiceClass = serviceClass;
    }
}
```

- **UpgradePatch，又是一顿补丁文件校验， 最后的tryRecoverDexFiles开始dex文件的合成**

```java
public class UpgradePatch extends AbstractPatch {
    private static final String TAG = "Tinker.UpgradePatch";

    @Override
    public boolean tryPatch(Context context, String tempPatchPath, PatchResult patchResult) {
        Tinker manager = Tinker.with(context);

        final File patchFile = new File(tempPatchPath);
        //再一次检查已安装的APP，是否开启了tinkerEnable也就意味着
        //是否可以进行patch操作
        if (!manager.isTinkerEnabled() || !ShareTinkerInternals.isTinkerEnableWithSharedPreferences(context)) {
            return false;
        }
         //检验patch.apk是否存在，是否可读，并且文件内容不为0
        if (!SharePatchFileUtil.isLegalFile(patchFile)) {
            return false;
        }
        //构建检查patch文件信息的对象
        ShareSecurityCheck signatureCheck = new ShareSecurityCheck(context);

        //校验patch包的签名和 本地安装包的签名是否一致
        int returnCode = ShareTinkerInternals.checkTinkerPackage(context, manager.getTinkerFlags(), patchFile, signatureCheck);
        if (returnCode != ShareConstants.ERROR_PACKAGE_CHECK_OK) {
            return false;
        }

        //md5= e08e3479895cae19572e26e799dcabb9
        String patchMd5 = SharePatchFileUtil.getMD5(patchFile);
        //use md5 as version
        patchResult.patchVersion = patchMd5;
      
        //check ok, we can real recover a new patch
        //准备合成后的文件需要的目录
        ///data/user/0/tinker.sample.android/tinker
        final String patchDirectory = manager.getPatchDirectory().getAbsolutePath();
      
        //创建文件锁
        //data/user/0/tinker.sample.android/tinker/info.lock
        File patchInfoLockFile = SharePatchFileUtil.getPatchInfoLockFile(patchDirectory);
        //合成成功后，用来记录补丁信息，因为合成后补丁会被删除
        ///data/user/0/tinker.sample.android/tinker/patch.info
        File patchInfoFile = SharePatchFileUtil.getPatchInfoFile(patchDirectory);

        //从补丁包的assets/package_meta.txt读取出以下信息
        //TINKER_ID -> tinker_id_baseApk-1.0.0
        //patchVersion -> 1.0
        //NEW_TINKER_ID -> tinker_id_patch-1.0.0
        //is_protected_app -> 0
        //platform -> all
        final Map<String, String> pkgProps = signatureCheck.getPackagePropertiesIfPresent();
        
        //有没有在gradle中配置 声明自己是个加固后的APP的开关
        final String isProtectedAppStr = pkgProps.get(ShareConstants.PKGMETA_KEY_IS_PROTECTED_APP);
        final boolean isProtectedApp = (isProtectedAppStr != null && !isProtectedAppStr.isEmpty() && !"0".equals(isProtectedAppStr));


        //it is a new patch, so we should not find a exist
        SharePatchInfo newInfo = new SharePatchInfo("", patchMd5, isProtectedApp, false, Build.FINGERPRINT, ShareConstants.DEFAULT_DEX_OPTIMIZE_PATH, false);

        //md5 =e08e3479895cae19572e26e799dcabb9
        //patch-name= patch-e08e3479
        final String patchName = SharePatchFileUtil.getPatchVersionDirectory(patchMd5);

        final String patchVersionDirectory = patchDirectory + "/" + patchName;

        ShareTinkerLog.i(TAG, "UpgradePatch tryPatch:patchVersionDirectory:%s", patchVersionDirectory);

        //copy file
        ///data/user/0/tinker.sample.android/tinker/patch-e08e3479/patch-e08e3479.apk
        File destPatchFile = new File(patchVersionDirectory + "/" + SharePatchFileUtil.getPatchVersionFile(patchMd5));

        try {
            // check md5 first
            //把patch.apk拷贝到内部目录
            if (!patchMd5.equals(SharePatchFileUtil.getMD5(destPatchFile))) {
                SharePatchFileUtil.copyFileUsingStream(patchFile, destPatchFile);
               
            }
        } catch (IOException e) {
            return false;
        }

        //开始合并dex文件，并进行dex优化
        if (!DexDiffPatchInternal.tryRecoverDexFiles(manager, signatureCheck, context, patchVersionDirectory, destPatchFile)) {
            return false;
        }
        
        //dex 与so 文件合并完成后会生成一个完整的新的apk文件 ///data/user/0/tinker.sample.android/tinker/patch-e08e3479/tinker_NClass.apk，下次重启时，拿它做dex插队就行了。
          ...... 
            
        //resource资源合并，合并生成resource.apk，重启后反射替换文件加载路径
        return true;
    }
```



### 动态加载dex补丁

- **该类在SampleApplication中被指定为加载补丁的loader类，在运行时会发射触发这里的tryLoad方法**

```java
 class TinkerLoader extends AbstractTinkerLoader {
  @Override
    public Intent tryLoad(TinkerApplication app) {
        Intent resultIntent = new Intent();
      
        tryLoadPatchFilesInternal(app, resultIntent);
      
        return resultIntent;
    }
}
```

```java
class TinkerLoader extends AbstractTinkerLoader{
  private void tryLoadPatchFilesInternal(TinkerApplication app, Intent resultIntent) {
    final int tinkerFlag = app.getTinkerFlags();

        //再次校验当前安装的APP 是否开启了 tinkerEnable,也就是是否可以进行patch操作
        if (!ShareTinkerInternals.isTinkerEnabled(tinkerFlag)) {
            return;
        }
        //检查当前不是在patch进程，因为开启新的tinkerpatchservice之后，也会触发application的初始化，
        // 也会进入这里，但是补丁的加载，只能在主进程。所以return
        if (ShareTinkerInternals.isInPatchProcess(app)) {
            ShareTinkerLog.w(TAG, "tryLoadPatchFiles: we don't load patch with :patch process itself, just return");
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_DISABLE);
            return;
        }
        //获取 合成后的补丁文件所在的tinker目录
        //tinker  /data/user/0/tinker.sample.android/tinker
        File patchDirectoryFile = SharePatchFileUtil.getPatchDirectory(app);
        if (patchDirectoryFile == null) {
            ShareTinkerLog.w(TAG, "tryLoadPatchFiles:getPatchDirectory == null");
            //treat as not exist
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_DIRECTORY_NOT_EXIST);
            return;
        }
        //tinker  /data/user/0/tinker.sample.android/tinker
        String patchDirectoryPath = patchDirectoryFile.getAbsolutePath();

        //目录文件不存在说明之前没有 patch过，那肯定不能合成
        //check patch directory whether exist
        if (!patchDirectoryFile.exists()) {
            ShareTinkerLog.w(TAG, "tryLoadPatchFiles:patch dir not exist:" + patchDirectoryPath);
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_DIRECTORY_NOT_EXIST);
            return;
        }

        //tinker/patch.info
        File patchInfoFile = SharePatchFileUtil.getPatchInfoFile(patchDirectoryPath);

        //check patch info file whether exist
        //检查合成后 记录 patch补丁文件信息的 patch.info是否存在，如果不存在也是不需要合成的
        if (!patchInfoFile.exists()) {
            ShareTinkerLog.w(TAG, "tryLoadPatchFiles:patch info not exist:" + patchInfoFile.getAbsolutePath());
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_INFO_NOT_EXIST);
            return;
        }
        //old = 641e634c5b8f1649c75caf73794acbdf
        //new = 2c150d8560334966952678930ba67fa8
        File patchInfoLockFile = SharePatchFileUtil.getPatchInfoLockFile(patchDirectoryPath);

        //读取之前合成的补丁文件的信息
        patchInfo = SharePatchInfo.readAndCheckPropertyWithLock(patchInfoFile, patchInfoLockFile);
        if (patchInfo == null) {
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_INFO_CORRUPTED);
            return;
        }

        final boolean isProtectedApp = patchInfo.isProtectedApp;
        resultIntent.putExtra(ShareIntentUtil.INTENT_IS_PROTECTED_APP, isProtectedApp);

        String oldVersion = patchInfo.oldVersion;
        String newVersion = patchInfo.newVersion;
        String oatDex = patchInfo.oatDir;

        //补丁文件信息 无效，则return
        if (oldVersion == null || newVersion == null || oatDex == null) {
            //it is nice to clean patch
            ShareTinkerLog.w(TAG, "tryLoadPatchFiles:onPatchInfoCorrupted");
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_INFO_CORRUPTED);
            return;
        }

        
                SharePatchInfo.rewritePatchInfoFileWithLock(patchInfoFile, patchInfo, patchInfoLockFile);
                ShareTinkerInternals.killProcessExceptMain(app);
                String patchVersionDirFullPath = patchDirectoryPath + "/" + patchName;
                SharePatchFileUtil.deleteDir(patchVersionDirFullPath + "/" + ShareConstants.INTERPRET_DEX_OPTIMIZE_PATH);
            }
        }

        resultIntent.putExtra(ShareIntentUtil.INTENT_PATCH_OLD_VERSION, oldVersion);
        resultIntent.putExtra(ShareIntentUtil.INTENT_PATCH_NEW_VERSION, newVersion);

        boolean versionChanged = !(oldVersion.equals(newVersion));
        //oat模式，也就是使用pms#registerDexModule触发的dex2oat 编译
        boolean oatModeChanged = oatDex.equals(ShareConstants.CHANING_DEX_OPTIMIZE_PATH);

        //得到oat之后生成的.odex 文件路径
        oatDex = ShareTinkerInternals.getCurrentOatMode(app, oatDex);
        resultIntent.putExtra(ShareIntentUtil.INTENT_PATCH_OAT_DIR, oatDex);

        String version = oldVersion;
        if (versionChanged && mainProcess) {
            version = newVersion;
        }
        if (ShareTinkerInternals.isNullOrNil(version)) {
            ShareTinkerLog.w(TAG, "tryLoadPatchFiles:version is blank, wait main process to restart");
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_INFO_BLANK);
            return;
        }

        //patch-641e634c
        String patchName = SharePatchFileUtil.getPatchVersionDirectory(version);
        if (patchName == null) {
            ShareTinkerLog.w(TAG, "tryLoadPatchFiles:patchName is null");
            //we may delete patch info file
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_VERSION_DIRECTORY_NOT_EXIST);
            return;
        }
        //tinker/patch-641e634c
        //下面是获取patch补丁文件，先判断它的目录是否存在
        String patchVersionDirectory = patchDirectoryPath + "/" + patchName;
        File patchVersionDirectoryFile = new File(patchVersionDirectory);
        if (!patchVersionDirectoryFile.exists()) {
            ShareTinkerLog.w(TAG, "tryLoadPatchFiles:onPatchVersionDirectoryNotFound");
            //we may delete patch info file
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_VERSION_DIRECTORY_NOT_EXIST);
            return;
        }


        //tinker/patch-641e634c/patch-641e634c.apk
        //得到patch补丁文件的路径
        final String patchVersionFileRelPath = SharePatchFileUtil.getPatchVersionFile(version);
        File patchVersionFile = (patchVersionFileRelPath != null ? new File(patchVersionDirectoryFile.getAbsolutePath(), patchVersionFileRelPath) : null);

        //补丁文件 是否是个无效的文件
        if (!SharePatchFileUtil.isLegalFile(patchVersionFile)) {
            ShareTinkerLog.w(TAG, "tryLoadPatchFiles:onPatchVersionFileNotFound");
            //we may delete patch info file
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_VERSION_FILE_NOT_EXIST);
            return;
        }

        ShareSecurityCheck securityCheck = new ShareSecurityCheck(app);

        //再次校验补丁文件签名和当前安装包的签名是否一致
        int returnCode = ShareTinkerInternals.checkTinkerPackage(app, tinkerFlag, patchVersionFile, securityCheck);
        if (returnCode != ShareConstants.ERROR_PACKAGE_CHECK_OK) {
            ShareTinkerLog.w(TAG, "tryLoadPatchFiles:checkTinkerPackage");
            resultIntent.putExtra(ShareIntentUtil.INTENT_PATCH_PACKAGE_PATCH_CHECK, returnCode);
            ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_PACKAGE_CHECK_FAIL);
            return;
        }
        ......
        //now we can load patch jar
        if (!isArkHotRuning && isEnabledForDex) {
            boolean loadTinkerJars = TinkerDexLoader.loadTinkerJars(app, patchVersionDirectory, oatDex, resultIntent, isSystemOTA, isProtectedApp);

            if (isSystemOTA) {
                //加载成功后更新 patch补丁文件的信息，并写入文件
                // update fingerprint after load success
                patchInfo.fingerPrint = Build.FINGERPRINT;
                patchInfo.oatDir = loadTinkerJars ? ShareConstants.INTERPRET_DEX_OPTIMIZE_PATH : ShareConstants.DEFAULT_DEX_OPTIMIZE_PATH;
                // reset to false
                oatModeChanged = false;

                if (!SharePatchInfo.rewritePatchInfoFileWithLock(patchInfoFile, patchInfo, patchInfoLockFile)) {
                    ShareIntentUtil.setIntentReturnCode(resultIntent, ShareConstants.ERROR_LOAD_PATCH_REWRITE_PATCH_INFO_FAIL);
                    ShareTinkerLog.w(TAG, "tryLoadPatchFiles:onReWritePatchInfoCorrupted");
                    return;
                }
                // update oat dir
                resultIntent.putExtra(ShareIntentUtil.INTENT_PATCH_OAT_DIR, patchInfo.oatDir);
            }
        }

        //加载合并生成的resource.apk中的全量资源文件
        //now we can load patch resource
        if (isEnabledForResource) {
            boolean loadTinkerResources = TinkerResourceLoader.loadTinkerResources(app, patchVersionDirectory, resultIntent);
            if (!loadTinkerResources) {
                ShareTinkerLog.w(TAG, "tryLoadPatchFiles:onPatchLoadResourcesFail");
                return;
            }
        }

        // Init component hotplug support.
        // 注册activity
        if ((isEnabledForDex || isEnabledForArkHot) && isEnabledForResource) {
            ComponentHotplug.install(app, securityCheck);
        }
        ......
  }
}
```

- **dex文件动态加载**

```java
class TinkerDexLoader{
  public static boolean loadTinkerJars(final TinkerApplication application, String directory, String oatDir, Intent intentResult, boolean isSystemOTA, boolean isProtectedApp) {
         ......
        //dexPath = /data/user/0/tinker.sample.android/tinker/patch-e48bf916/dex/
        SystemClassLoaderAdder.installDexes(application, classLoader, optimizeDir, legalFiles, isProtectedApp);
        ......
        return true;
    }
}
```

- 根据不同的平台执行不同的dex加载策略，但实际上都脱不开dex插队的思想

```java
class SystemClassLoaderAdder{
  public static void installDexes(Application application, ClassLoader loader, File dexOptDir, List<File> files, boolean isProtectedApp){
        if (!files.isEmpty()) {//files=[/data/user/0/tinker.sample.android/tinker/patch-e48bf916/dex/tinker_classN.apk]
            files = createSortedAdditionalPathEntries(files);
            ClassLoader classLoader = loader;
            if (Build.VERSION.SDK_INT >= 24 && !isProtectedApp) {
                //对于7.0及以上，且没有加固的APP
                //自己构造tinkerClassLoader负责类加载
                //但实际上，我测试过，使用下面的dex插队，也是可以成功加载补丁类的
                classLoader = NewClassLoaderInjector.inject(application, loader, dexOptDir, files);
            } else {
                //because in dalvik, if inner class is not the same classloader with it wrapper class.
                //it won't fail at dex2opt
                if (Build.VERSION.SDK_INT >= 23) {
                    //dex插队
                    V23.install(classLoader, files, dexOptDir);
                } else if (Build.VERSION.SDK_INT >= 19) {
                    //dex插队，只是比6.0的 makeDexElement参数的类型略有不同
                    V19.install(classLoader, files, dexOptDir);
                } else if (Build.VERSION.SDK_INT >= 14) {
                    //dex 插队只是比6.0的 makeDexElement少1个参数
                    V14.install(classLoader, files, dexOptDir);
                } else {
                    //也是dex插队
                    V4.install(classLoader, files, dexOptDir);
                }
            }
        }
    }
}
```

- **7.1及以上dex动态加载实现方式**

```java
final class NewClassLoaderInjector {
    //files=[/data/user/0/tinker.sample.android/tinker/patch-e48bf916/dex/tinker_classN.apk]
    //dexPotDir= /data/user/0/tinker.sample.android/tinker/patch-e48bf916/odex
    public static ClassLoader inject(Application app, ClassLoader oldClassLoader, File dexOptDir, List<File> patchedDexes) throws Throwable {

        final ClassLoader newClassLoader = createNewClassLoader(app, oldClassLoader, dexOptDir, patchedDexPaths);
        doInject(app, newClassLoader);
        return newClassLoader;
    }

    private static ClassLoader createNewClassLoader(Context context, ClassLoader oldClassLoader,File dexOptDir, String... patchDexPaths) throws Throwable {
        final Field pathListField = findField(
                Class.forName("dalvik.system.BaseDexClassLoader", false, oldClassLoader),
                "pathList");
        final Object oldPathList = pathListField.get(oldClassLoader);

        //把需要动态加载的dex 文件的路径拼装起来，使用；分割，但实际上这里只有一个tinker_NClass.apk文件
        final StringBuilder dexPathBuilder = new StringBuilder();
        final boolean hasPatchDexPaths = patchDexPaths != null && patchDexPaths.length > 0;
        if (hasPatchDexPaths) {
            for (int i = 0; i < patchDexPaths.length; ++i) {
                if (i > 0) {
                    dexPathBuilder.append(File.pathSeparator);
                }
                dexPathBuilder.append(patchDexPaths[i]);
            }
        }

        final String combinedDexPath = dexPathBuilder.toString();


        //找出pathclassloader中已经加载的动态库的 路径
        final Field nativeLibraryDirectoriesField = findField(oldPathList.getClass(), "nativeLibraryDirectories");
        List<File> oldNativeLibraryDirectories = null;
        if (nativeLibraryDirectoriesField.getType().isArray()) {
            oldNativeLibraryDirectories = Arrays.asList((File[]) nativeLibraryDirectoriesField.get(oldPathList));
        } else {
            oldNativeLibraryDirectories = (List<File>) nativeLibraryDirectoriesField.get(oldPathList);
        }
        final StringBuilder libraryPathBuilder = new StringBuilder();
        boolean isFirstItem = true;
        for (File libDir : oldNativeLibraryDirectories) {
            if (libDir == null) {
                continue;
            }
            if (isFirstItem) {
                isFirstItem = false;
            } else {
                libraryPathBuilder.append(File.pathSeparator);
            }
            libraryPathBuilder.append(libDir.getAbsolutePath());
        }

        final String combinedLibraryPath = libraryPathBuilder.toString();

        //构建一个新的classloader 来接管类加载
        final ClassLoader result = new TinkerClassLoader(combinedDexPath, dexOptDir, combinedLibraryPath, oldClassLoader);
        findField(oldPathList.getClass(), "definingContext").set(oldPathList, result);

        return result;
    }

    private static void doInject(Application app, ClassLoader classLoader) throws Throwable {
        Thread.currentThread().setContextClassLoader(classLoader);

        //如想要生效，还要把刚才创建的tinkerClassLoader替换掉 loadedApk中的 classloader对象
        final Context baseContext = (Context) findField(app.getClass(), "mBase").get(app);
        final Object basePackageInfo = findField(baseContext.getClass(), "mPackageInfo").get(baseContext);
        findField(basePackageInfo.getClass(), "mClassLoader").set(basePackageInfo, classLoader);

        if (Build.VERSION.SDK_INT < 27) {
            //在8.0上，还需要替换resource中的classloader对象
            final Resources res = app.getResources();
            try {
                findField(res.getClass(), "mClassLoader").set(res, classLoader);

                //还需要替换resource中mDrawableInflater，中的classloader对象
                final Object drawableInflater = findField(res.getClass(), "mDrawableInflater").get(res);
                if (drawableInflater != null) {
                    findField(drawableInflater.getClass(), "mClassLoader").set(drawableInflater, classLoader);
                }
            } catch (Throwable ignored) {
                // Ignored.
            }
        }
}
```



