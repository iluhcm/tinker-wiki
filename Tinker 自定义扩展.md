Tinker 自定义扩展      
====================================
## 自定义Application类
程序启动时会加载默认的Application类，这导致我们补丁包是无法对它做修改了。如何规避？在这里我们并没有使用类似InstantRun `hook Application`的方式，而是通过代码框架的方式来避免，这也是为了尽量少的去反射，提升框架的兼容性。

这里我们要实现的是完全将原来的Application类隔离起来，即其他任何类都不能再引用我们自己的Application。我们需要做的其实是以下几个工作：

1. 将我们自己Application类以及它的继承类的所有代码拷贝到自己的ApplicationLike继承类中，例如SampleApplicationLike。你也可以直接将自己的Application改为继承ApplicationLike;
2. Application的`attachBaseContext`方法实现要单独移动到`onBaseContextAttached`中；
3. 对ApplicationLike中，引用application的地方改成`getApplication()`;
4. 对其他引用Application或者它的静态对象与方法的地方，改成引用ApplicationLike的静态对象与方法；

更详细的事例，大家可以参考下面的一些例子以及[SampleApplicationLike](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/app/SampleApplicationLike.java)的做法。

**如果你不愿意改造自己的应用，可以尝试TinkerPatch的一键傻瓜式接入，具体的可参考文档[TinkerPatch 平台介绍](http://tinkerpatch.com/Docs/intro)。**

### Application代理类
为了使真正的Application实现可以在补丁包中修改，我们把Appliction类的所有逻辑移动到[ApplicationLike](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-loader/src/main/java/com/tencent/tinker/loader/app/ApplicationLike.java)代理类中。

```java
-public class YourApplication extends Application {
+public class SampleApplicationLike extends DefaultApplicationLike 
```

**在1.7.6版本之前，我们需要手动同时将gradle的dex loader中的Application改为新的YourApplication。在1.7.6版本之后，tinker-build-plugin将会自动写入，我们无须手动填写。**

```xml
dex {
loader = ["com.tencent.tinker.loader.*",
         //warning, you must change it with your application
          "tinker.sample.android.YourApplication",       
}
```

具体实现可参考[SampleApplicationLike](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/app/SampleApplicationLike.java), 其中对Application类的调用可以修改成：

```java
public void registerActivityLifecycleCallbacks(Application.ActivityLifecycleCallbacks callback) {
    application.registerActivityLifecycleCallbacks(callback);
}
```

若你的应用代理Application的ClassLoader、Resource以及AssetsManger，可以使用以下方法设置。

```java
applicationLike.setResources(res);
applicationLike.setClassLoader(classloader);
applicationLike.setTAssets(assets);
```

事实上，你也可以在你的Application类加入代理，但是在Application中尽量不要引用自己的类，将真正的实现放在外面。

```java
public class YourrApplication extends Application {
	ActivityLifecycleCallbacks activityLifecycleCallbacks;
	public void setTinkerActivityLifecycleCallbacks(ActivityLifecycleCallbacks activityLifecycleCallbacks) {
		this.activityLifecycleCallbacks = activityLifecycleCallbacks;
	}
```

### 修改你的Application类
然后将你的Application类继承[TinkerApplication.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-loader/src/main/java/com/tencent/tinker/loader/app/TinkerApplication.java)。`除了构造方法之外，你最好不要引入其他的类，这将导致它们无法通过补丁修改。`

```java
public class SampleApplication extends TinkerApplication {
    public SampleApplication() {
      super(
        //tinkerFlags, tinker支持的类型，dex,library，还是全部都支持！
        ShareConstants.TINKER_ENABLE_ALL,
        //ApplicationLike的实现类，只能传递字符串 
        "tinker.sample.android.app.SampleApplicationLike",
        //Tinker的加载器，一般来说用默认的即可
        "com.tencent.tinker.loader.TinkerLoader",
        //tinkerLoadVerifyFlag, 运行加载时是否校验dex与,ib与res的Md5
        false);
    }  
}
```

具体的数值含义如下：

| 参数               | 默认值      | 描述       | 
| ----------------- | ---------  | ---------  | 
| tinkerFlags       |   TINKER_DISABLE     | tinker运行时支持的补丁包中的文件类型:<br> 1. ShareConstants.TINKER_DISABLE:不支持任何类型的文件；<br> 2. ShareConstants.TINKER_DEX_ONLY:只支持dex文件；<br> 3. ShareConstants.TINKER_LIBRARY_ONLY:只支持library文件;<br> 4. ShareConstants.TINKER_DEX_AND_LIBRARY:只支持dex与res的修改；<br> 5. ShareConstants.TINKER_ENABLE_ALL:支持任何类型的文件，也是我们通常的设置的模式。        |
| delegateClassName    | "com.tencent.tinker.loader<br>.app.DefaultApplicationLike"  | Application代理类的类名，这里只能使用字符串，`不能使用class.getName()`。 |
| loaderClassName     | "com.tencent.tinker.<br>loader.TinkerLoader" | 加载Tinker的主类名，对于特殊需求可能需要使用自己的加载类。需要注意的是：<br>`这个类以及它使用的类都是不能被补丁修改的，并且我们需要将它们加到dex.loader[]中`。<br>一般来说，我们使用默认即可。|
| tinkerLoadVerifyFlag  | false | 由于合成过程中我们已经校验了各个文件的Md5，并将它们存放在/data/data/..目录中。默认每次加载时我们并不会去校验tinker文件的Md5,但是你也可通过开启loadVerifyFlag强制每次加载时校验，但是这会带来一定的时间损耗。|

**Warning: 这里务必不能写成SampleApplicationLike.class.getName()，只能通过传递字符串的方式。为了减少错误的出现，推荐使用Annotation生成Application类**

### 使用Annotation生成Application类
为了隐藏你的Application类，我们更加推荐你使用`tinker-android-anno`在运行时生成你的Application类。这样保证你无法修改你的Application类，不会因为错误操作导致引入更多无法修改的类。

```java
@DefaultLifeCycle(
application = ".SampleApplication",                       //application类名
flags = ShareConstants.TINKER_ENABLE_ALL,                 //tinkerFlags
loaderClass = "com.tencent.tinker.loader.TinkerLoader",   //loaderClassName, 我们这里使用默认即可!
loadVerifyFlag = false)                                   //tinkerLoadVerifyFlag
public class SampleApplicationLike extends DefaultApplicationLike
```

**若采用Annotation生成Application,需要将原来的Application类删掉**。到此为止，Tinker初步的接入已真正的完成，你已经可以愉快的使用Tinker来实现补丁功能了。


## 可选的自定义类
在Tinker中你可以自定义一些类，它们需要在构造Tinker实例时作为参数传递，在[TinkerManager](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/util/TinkerManager.java)的`installTinker`中，你可以根据自己的需要自定义其中的一些类:

```java
//or you can just use DefaultLoadReporter
LoadReporter loadReporter = new SampleLoadReporter(appLike.getApplication());
//or you can just use DefaultPatchReporter
PatchReporter patchReporter = new SamplePatchReporter(appLike.getApplication());
//or you can just use DefaultPatchListener
PatchListener patchListener = new SamplePatchListener(appLike.getApplication());
//you can set your own upgrade patch if you need
AbstractPatch upgradePatchProcessor = new UpgradePatch();
TinkerInstaller.install(appLike,
	loadReporter, patchReporter, patchListener,
    SampleResultService.class, upgradePatchProcessor);
```

你也可以使用`sampleInstallTinker`，即全部使用默认参数。

**各个类具体的功能与使用方法如下：**
   
### 自定义LoadReporter类
[LoadReporter类](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/reporter/LoadReporter.java)定义了Tinker在加载补丁时的一些回调，我们为你提供了默认实现[DefaultLoadReporter.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/reporter/DefaultLoadReporter.java).

一般来说, 你可以继承DefaultLoadReporter实现你自己感兴趣的事件回调，例如[SampleLoadReporter.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/reporter/SampleLoadReporter.java). 

我们在sample中也写了一个默认的回调上报例子，可参考[SampleTinkerReport](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/reporter/SampleTinkerReport.java).

**需要注意的有以下两点**：

1. 回调运行在加载的进程，它有可能是各个不一样的进程。我们可以通过tinker.isMainProcess或者tinker.isPatchProcess知道当前是否是主进程，patch补丁合成进程。
2. 回调发生的时机是我们调用`installTinker`之后，某些进程可能并不需要installTinker。

现对各个回调作简要的说明:

| 函数               | 描述       | 
| ----------------- | ---------  | 
| `onLoadResult`      |这个是无论加载失败或者成功都会回调的接口，它返回了本次加载所用的时间、返回码等信息。默认我们只是简单的输出这个信息，你可以在这里加上监控上报逻辑。     | 
| `onLoadPatchListenerReceiveFail`| 所有的补丁合成请求都需要先通过PatchListener的检查过滤。这次检查不通过的回调，`它运行在发起请求的进程`。默认我们只是打印日志  | 
| `onLoadPatchVersionChanged`      |补丁包版本升级的回调，只会在主进程调用。默认我们会杀掉其他所有的进程(保证所有进程代码的一致性)，并且删掉旧版本的补丁文件。     | 
| `onLoadFileNotFound`    | 在加载过程中，发现部分文件丢失的回调。默认若是dex，dex优化文件或者lib文件丢失，我们将尝试从补丁包去修复这些丢失的文件。若补丁包或者版本文件丢失，将卸载补丁包。 | 
| onLoadFileMd5Mismatch     | 部分文件的md5与meta中定义的不一致。默认我们为了安全考虑，依然会清空补丁。| 
| onLoadPatchInfoCorrupted     | patch.info是用来管理补丁包版本的文件，这是info文件损坏的回调。默认我们会卸载补丁包，因为此时我们已经无法恢复了。|
| onLoadPackageCheckFail     | 加载过程补丁包的检查失败，这里可以通过错误码区分，例如签名校验失败、tinkerId不一致等原因。默认我们将会卸载补丁包|
| `onLoadException`     | 在加载过程捕捉到异常，`十分希望你可以把错误信息反馈给我们`。默认我们会直接卸载补丁包 |
| `onLoadInterpret`     | 系统OTA后，为了加快补丁的执行，我们会采用解释模式来执行补丁。 |

所有的错误码都定义在[ShareConstants.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-loader/src/main/java/com/tencent/tinker/loader/shareutil/ShareConstants.java)，`onLoadPackageCheckFail` 的相关错误码解析如下：

| 错误码              | 数值       | 描述       | 
| ----------------- | ---- | ---------  | 
| ERROR_PACKAGE_CHECK_SIGNATURE_FAIL    	    |-1 | 签名校验失败| 
| ERROR_PACKAGE_CHECK_PACKAGE_META_NOT_FOUND    |-2|找不到"assets/package_meta.txt"文件| 
| ERROR_PACKAGE_CHECK_DEX_META_CORRUPTED | -3|"assets/dex_meta.txt"信息损坏|
| ERROR_PACKAGE_CHECK_LIB_META_CORRUPTED       | -4|"assets/so_meta.txt"信息损坏|
| ERROR_PACKAGE_CHECK_APK_TINKER_ID_NOT_FOUND    |-5|找不到基准apk AndroidManifest中的TINKER_ID | 
| ERROR_PACKAGE_CHECK_PATCH_TINKER_ID_NOT_FOUND | -6|找不到补丁中"assets/package_meta.txt"中的TINKER_ID
| ERROR_PACKAGE_CHECK_TINKER_ID_NOT_EQUAL       | -7|基准版本与补丁定义的TINKER_ID不相等|
| ERROR_PACKAGE_CHECK_RESOURCE_META_CORRUPTED       | -8|"assets/res_meta.txt"信息损坏|
| ERROR_PACKAGE_CHECK_TINKERFLAG_NOT_SUPPORT       | -9|tinkerFlag不支持补丁中的某些类型的更改，例如补丁中存在资源更新，但是使用者指定不支持资源类型更新。|

`onLoadException`的错误码具体如下：

| 错误码              | 数值       | 描述       | 
| ----------------- | ---- | ---------  | 
| ERROR_LOAD_EXCEPTION_UNKNOWN    	                  |-1 | 没有捕获到的java crash | 
| ERROR_LOAD_EXCEPTION_DEX                  |-2|在加载dex过程中捕获到的crash  | 
| ERROR_LOAD_EXCEPTION_RESOURCE | -3|在加载res过程中捕获到的crash|
| ERROR_LOAD_EXCEPTION_UNCAUGHT       | -4|没有捕获到的非java crash,这个是补丁机制的安全模式|

回调中定义的fileType定义如下：

| 文件类型              | 数值       | 描述       | 
| ----------------- | ---- | ---------  | 
| TYPE_PATCH_FILE    	                  |1 | 补丁文件 | 
| TYPE_PATCH_INFO                  |2|"patch.info"补丁版本配置文件 | 
| TYPE_DEX | 3|在Dalvik合成全量的Dex文件|
| TYPE_DEX_OPT       | 4|odex文件|
| TYPE_LIBRARY       | 5|library文件|
| TYPE_RESOURCE       | 6|资源文件|

加载过程的具体的错误类型与错误码可查看[DefaultLoadReporter.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/reporter/DefaultLoadReporter.java)的注释。


对于onLoadPatchVersionChanged与onLoadFileNotFound的复写要较为谨慎，因为版本升级杀掉其他进程与文件丢失发起恢复任务，都是我认为比较重要的操作。

### 自定义PatchReporter类
[PatchReporter类](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/reporter/PatchReporter.java)定义了Tinker在修复或者升级补丁时的一些回调，我们为你提供了默认实现[DefaultPatchReporter.java](http://git.code.oa.com/wechat-android-dev/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/reporter/DefaultPatchReporter.java).

一般来说, 你可以继承DefaultPatchReporter实现你自己感兴趣的事件回调，例如[SamplePatchReporter.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/reporter/SamplePatchReporter.java).

**需要注意的是**:

| 函数               | 描述       | 
| ----------------- | ---------  | 
| `onPatchResult`    |这个是无论补丁合成失败或者成功都会回调的接口，它返回了本次合成的类型，时间以及结果等。默认我们只是简单的输出这个信息，你可以在这里加上监控上报逻辑。     | 
| `onPatchServiceStart`    |这个是Patch进程启动时的回调，我们可以在这里进行一个统计的工作。     | 
| onPatchPackageCheckFail     | 补丁合成过程对输入补丁包的检查失败，这里可以通过错误码区分，例如签名校验失败、tinkerId不一致等原因。默认我们会删除临时文件。|
| onPatchVersionCheckFail     | 对patch.info的校验版本合法性校验。若校验失败，默认我们会删除临时文件。|
| onPatchTypeExtractFail     | 从补丁包与原始安装包中合成某种类型的文件出现错误，默认我们会删除临时文件。| 
| onPatchDexOptFail     | 对合成的dex文件提前进行dexopt时出现异常，默认我们会删除临时文件。|
| onPatchInfoCorrupted     |patch.info是用来管理补丁包版本的文件，这是在更新info文件时发生损坏的回调。默认我们会卸载补丁包，因为此时我们已经无法恢复了。|
| `onPatchException`     | 在补丁合成过程捕捉到异常，`十分希望你可以把错误信息反馈给我们`。默认我们会删除临时文件，并且将tinkerFlag设为不可用。|

PatchReporter中onPatchPackageCheckFail的错误码与LoadReporter的一致。


### 自定义PatchListener类
[PatchListener类](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/listener/PatchListener.java)是用来过滤Tinker收到的补丁包的修复、升级请求，也就是决定我们是不是真的要唤起:patch进程去尝试补丁合成。我们为你提供了默认实现[DefaultPatchListener.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/listener/DefaultPatchListener.java)。

一般来说, 你可以继承DefaultPatchListener并且加上自己的检查逻辑，例如[SamplePatchListener.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/reporter/SamplePatchListener.java)。

若检查成功，我们会调用`TinkerPatchService.runPatchService`唤起:patch进程，去尝试完成补丁合成操作。反之，会回调检验失败的接口。事实上，你只需要复写`patchCheck`函数即可。若检查失败，会在LoadReporter的onLoadPatchListenerReceiveFail中回调。

```java
public int patchCheck(String path)
```

以DefaultPatchListener为例，说明默认我们检查的条件，你可以定义自己的错误码，也可以沿用这里的错误码。

| 错误码              | 数值       | 描述       | 
| ----------------- | ---------  | ---------  | 
| ERROR_PATCH_DISABLE    |-1|当前tinkerFlag为不可用状态。     | 
| ERROR_PATCH_NOTEXIST| -2|输入的临时补丁包文件不存在。  | 
| ERROR_PATCH_RUNNING |-3    | 当前:patch补丁合成进程正在运行。|
| ERROR_PATCH_INSERVICE | -4|不能在:patch补丁合成进程，发起补丁的合成请求。|
| ERROR_PATCH_JIT | -5|补丁不支持 N 之前的 JIT 模式。|
| 其他     | |在SamplePatchListener里面，我们还检查了当前Rom剩余空间，最大内存，是否是GooglePlay渠道等条件。| 

### 自定义AbstractResultService类
[AbstractResultService类](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/service/AbstractResultService.java)是:patch补丁合成进程将合成结果返回给主进程的类。我们为你提供了默认实现[DefaultTinkerResultService.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/service/DefaultTinkerResultService.java)。

一般来说, 你可以继承DefaultTinkerResultService实现自己的回调，例如[SampleResultService.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/service/SampleResultService.java)。**当然，你也需要在AndroidManifest上添加你的Service。**

```xml
<service
    android:name=".service.SampleResultService"
	android:exported="false"
/>
```

默认我们在DefaultTinkerResultService会杀掉:patch进程，假设当前是补丁升级并且成功了，我们会杀掉当前进程，让补丁包更快的生效。若是修复类型的补丁包并且失败了，我们会卸载补丁包。下面对[PatchResult](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/service/PatchResult.java)的定义做详细说明：

| 函数               | 描述       | 
| ----------------- | ---------  | 
| isSuccess| 补丁合成操作是否成功。  | 
| rawPatchFilePath | 原始的补丁包路径。|
| costTime     | 本次补丁合成的耗时。|
| e     | 本次补丁合成是否出现异常，null为没有异常。| 
| patchVersion     | 补丁文件的md5, 有可能为空@Nullable。| 

在[SampleResultService.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/service/SampleResultService.java)中，我们没有立刻杀掉当前进程去应用补丁，而选择在当前应用在退入后台或手机锁屏时这两个时机。你也可以在自杀前，通过发送service或者broadcast inent来尽快重启进程。

### 自定义TinkerLoader类
[TinkerLoader类](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-loader/src/main/java/com/tencent/tinker/loader/TinkerLoader.java)是用来加载补丁的核心类，你可以实现自己的加载逻辑。但是`一般不建议那么做`，如果你一定要你需要保证以下两条规则:

1. 将你的实现的类以及它用到的所有类都加入到dex.loader中;
2. 保证上述的类都在main dex中。

只要简单的将loaderClass参数中的"com.tencent.tinker.loader.TinkerLoader"，换成你的实现类的名称即可，这里只能传递字符串。

```java
@DefaultLifeCycle(
application = ".SampleApplication",                       //application类名
flags = ShareConstants.TINKER_ENABLE_ALL,                 //tinkerFlags
loaderClass = "com.tencent.tinker.loader.TinkerLoader")   //loaderClassName, 我们这里使用默认即可!
public class SampleApplicationLike extends DefaultApplicationLike 
```
### Tinker Notification id设置
为了提高[TinkerPatchService](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/service/TinkerPatchService.java)的进程优先级，我们将它设置为`Foreground`。对于sdk>18的版本，使用innerService方式使通知栏不会显示。

**Warning, 这里占用了id为-1119860829.若你的app存在与它相同的id, 可以使用以下API重新设置**

```java
TinkerPatchService.setTinkerNotificationId(id);
Tinker.with(context).setPatchServiceNotificationId(id);
```

### 自定义UpgradePatch类
[UpgradePatch类](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/patch/UpgradePatch.java)是用来升级当前补丁包的处理类，一般来说你也不需要复写它。

可以看到整个Tinker框架非常灵活，基本所有的逻辑都放在可复写的类或回调中，你可以轻松的完成自身需要的自定义工作。

你可以根据需要自定义以上的一些类，然后我们继续学习[Tinker API概览](https://github.com/Tencent/tinker/wiki/Tinker-API%E6%A6%82%E8%A7%88)。