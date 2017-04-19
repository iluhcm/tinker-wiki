Tinker 常见问题
====================================
## Issue/提问须知
**在提交issue之前，我们应该先查询是否已经有相关的issue。提交issue时，我们需要写明issue的原因，以及编译或运行过程的日志(加载进程以及Patch进程)。issue需要以下面的格式：**

```
异常类型：app运行时异常/编译异常

手机型号：如:Nexus 5(如是编译异常，则可以不填)

手机系统版本：如:Android 5.0 (如是编译异常，则可以不填)

tinker版本：如:1.7.7

gradle版本：如:2.10

是否使用热更新SDK： 如 TinkerPatch SDK 或者 Bugly SDK

系统：如:Mac

堆栈/日志：
1. 如是编译异常，请在执行gradle命令时，加上--stacktrace;
2. 日志我们需要过滤"Tinker."关键字;
3. 对于合成失败的情况，请给出:patch进程的日志,这里需要将Android Moniter右上角设为No Filter。
```

提问题时若使用`不能用/没效果/有问题/报错`此类模糊表达，但又没给出任何代码截图报错的，将绝对不会有任何反馈。这种issue也是一律直接关闭的,大家可以参阅[提问的智慧](https://github.com/tvvocold/How-To-Ask-Questions-The-Smart-Way)。

Tinker是一个开源项目，希望大家遇到问题时要学会先思考，看看sample与Tinker的源码，更鼓励大家给我们提pr.

## Tinker编译相关问题？
编译过程相关的issue请先查看是否是以下情况：

1. `无法打开sample工程`： 请使用单独的IDE窗口打开tinker-sample-android工程；
2. `tinkerId is not set`: 这是因为没有正确的配置IDE的git路径, **若不是通过clone方式下载tinker，需要本地手动commit一次**。这里你也可以使用其他字符作为tinkerId;
3. 对于编译与补丁时发生的异常，请到[Tinker 自定义扩展](https://github.com/Tencent/tinker/wiki/Tinker-%E8%87%AA%E5%AE%9A%E4%B9%89%E6%89%A9%E5%B1%95)中查看具体错误码的原因。并通过“Tinker.”过滤Tinker相关的日志提交到issue中;
4. 若自定义TinkerResultService，请务必将新的Service添加到Manifest中;
5. 权限问题；请务必已经将读取sdk权限添加到AndroidManifest.xml中，并且已允许权限运行； 
6. 若使用`DefaultLifeCycle`注解生成Application，需要将原来Application的实现移动到ApplicationLike中，并将原来的Application类删掉;
7. 关于Application的改造这一块大家比较疑惑，这块请认真阅读[自定义Application类](https://github.com/Tencent/tinker/wiki/Tinker-%E8%87%AA%E5%AE%9A%E4%B9%89%E6%89%A9%E5%B1%95#%E8%87%AA%E5%AE%9A%E4%B9%89application%E7%B1%BB)，大部分的app应该都能在半小时内完成改造。
8. 如果出现`Class ref in pre-verified class resolved to unexpected implementation`异常, 请确认以下几点：Application中传入ApplicationLike的参数时是否采用字符串而不是Class.getName方式；新的Application是否已经加入到dex loader pattern中; 额外添加到dex loader pattern中类的引用类也需要加载到loader pattern中。


## Tinker库中有什么类是不能修改的？
Tinker库中不能修改的类一共有26个，即com.tencent.tinker.loader.*类。加上你的Appliction类，只有26个类是无法通过Tinker来修改的。即使类似Tinker.java等管理类，也是可以通过Tinker本身来修改。

**注意，在1.7.6版本之前，我们需要手动将不能修改的类添加到tinkerPatch.dex.loader pattern中。对于1.7.6以后的版本会自动生成。**

## 什么类需要放在主dex中？
Tinker并不干涉你分包与多dex的加载逻辑，但是你需要确保以下几点：

1. com.tencent.tinker.loader.*类，你的Application类需要在主dex，并且已经在dex.loader中配置;
2. 若你自定义了TinkerLoader类，你需要将TinkerLoader的自定义类，以及它用的到类也放在主dex，并且已经在dex.loader中配置;
3. ApplicationLike的继承类也需要放在主dex中，但是它无须在dex.loader中配置，因为它是可以使用Tinker修改的类。最后`如果你需要在加载其他dex之前加载Tinker的管理类，你也可以将com.tencent.tinker.*都加入到主dex`。
4. 你的ApplicationLike实现类的直接引用类以及在调用Multidex install之前加载的类也都需要放到主dex中。

**注意：Tinker会自动生成需要放在主dex的keep规则。在1.7.6版本之前，你需要手动将生成规则拷贝到自己的multiDexKeepProguard文件中。例如Sample中的`multiDexKeepProguard file("keep_in_main_dex.txt")`。在1.7.6版本之后，这里会通过脚本自动处理，无须手动填写。**

**另外，如果minsdkverion >=21, multiDexEnabled会被忽略。我们可以在build/intermediates/multi-dex查找最终的keep规则以及结果。**

## 我应该使用哪个作为补丁包下发，如何做多次修复？
`patch_signed_7zip.apk`是已签名并且经过7z压缩的补丁包，但是你最好重命名一下，不要让它以`.apk`结尾，这是因为有些运营商会挟持以`.apk`结尾的资源。

另外一点，我们在发起补丁请求时，**需要先将补丁包先拷贝到dataDir中**。因为在sdcard中，补丁包是极其容易被清理软件删除。这里可以参考[UpgradePatchRetry.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/util/UpgradePatchRetry.java)的实现。

对于补丁包的版本问题，我们可以在packageConfig中增加，例如sample中的

```xml
packageConfig {
	/**
     * patch version via packageConfig
     */
     configField("patchVersion", "1.0")
}
```

**Tinker支持对同一基准版本做多次补丁修复，在生成补丁时，oldApk依然是已经发布出去的那个版本。即补丁版本二的oldApk不能是补丁版本一，它应该依然是用户手机上已经安装的基准版本。**
   
## 如何对Library文件作补丁？
当前我们并没有直接将补丁的lib路径添加到`DexPathList`中，理论上这样可以做到程序完全没有感知的对Library文件作补丁。这里主要是因为在多abi的情况下，某些机器获取的并不准确。**当前对Library文件作补丁可参考[Tinker API概览](https://github.com/Tencent/tinker/wiki/Tinker-API%E6%A6%82%E8%A7%88)，tinker 1.7.7版本我们也提供了一键反射的方案给大家选择。**

大家可以根据自己的项目需要选择合适的方案，事实上，无论是对Library还是Application，我们都是采用尽量少去反射的策略，这也是为了提高Tinker框架的兼容性。上线前，我们应当严格测试补丁是否正确加载了修改后的So库。

## 如何对资源文件作补丁,为什么有时候会提示大量没有改变的图片发生变更？
Tinker采用全量合成方式实现资源替换，这里有以下几点是使用者需要明确的：

1. remoteView是无法修改，例如transition动画，notification icon以及桌面图标;
2. 对于资源文件的更新(尤其是assets)，需要注意代码中是否采用直接读取sourceApk路径方式读取，这样方式是无法更新的;
3. Tinker只会将满足res pattern的资源放在最后的合成补丁资源包中。一般为了减少合成资源大小，我们不建议输入classes.dex或lib文件的pattern;
4. 若一个文件:assets/classes.dex, 它既满足dex pattern, 又满足res pattern。Tinker只会处理dex pattern, 然后在合成资源包会忽略assets/classes.dex的变更。library也是如此。
5. 只要资源发生变成的前提下我们才会合成新的资源包，这一定程度会增加占Rom体积，请在考虑后使用。

**Waringing:若出现资源变更，我们需要使用applyResourceMapping方式编译，这样不仅可以减少补丁包大小，同时防止remote view id变更造成的异常情况。**最后我们应该查看编译过程中生成的`resources_out.zip`是否满足我们的要求。 

有时候会发现大量明明没有改变的png发现变更，解压发现的确两次编译这些png的md5不一致。经分析，aapt在其中一次编译将png优化成8-bit，另外一次却没有，从而导致png改变了。如果你们app出现了这种情况，我们建议关闭aapt对png的优化：

```xml
aaptOptions{
	cruncherEnabled false
}
```

若你对安装包大小非常care，可以提前使用命令行工具将所有图片手动优化一次。我们也可以选择一些有损压缩工具，获得更大的压缩效果。

如果你确认png并没有修改，你可以在tinker的配置使用ignoreChange来忽略所有png文件的修改。

```xml
res {
	ignoreChange = ["*.png"]
｝
```

## Tinker中的dex配置'raw'与'jar'模式应该如何选择？
它们应该说各有优劣势，大概应该有以下几条原则：

1. 如果你的minSdkVersion小于14, 那你务必要选择'jar'模式；
2. 以一个`10M`的dex为例，它压缩成jar大约为`4M`，即'jar'模式能节省`6M`的ROM空间。
3. 对于'jar'模式，我们需要验证压缩包流中dex的md5,这会更耗时，在`小米2S`上数据大约为'raw'模式`126ms`, 'jar'模式为`246ms`。

因为在合成过程中我们已经校验了各个文件的Md5，并将它们存放在/data/data/..目录中。`默认每次加载时我们并不会去校验tinker文件的Md5`,但是你也可通过开启loadVerifyFlag强制每次加载时校验，但是这会带来一定的时间损耗。

**简单来说，'jar'模式更省空间，但是运行时校验的耗时大约为'raw'模式的两倍。如果你没有打开运行时校验，推荐使用'jar'模式。**


## 如何兼容多渠道包？
关于渠道包的问题，若使用flavor编译渠道包，会导致不同的渠道包由于BuildConfig变化导致classes.dex差异。这里建议的方式有：   

1. 将渠道信息写在AndroidManifest.xml或文件中，例如channel.ini；  
2. 将渠道信息写在apk文件的zip comment中，这种是建议方式，例如可以使用项目[packer-ng-plugin](https://github.com/mcxiaoke/packer-ng-plugin)或者可使用V2 Scheme的[walle](https://github.com/Meituan-Dianping/walle)；  
3. 若不同渠道存在功能上的差异，建议将差异部分放于单独的dex或采用相同代码不同配置方式实现；

事实上，tinker也支持多flavor直接编译多个补丁包，具体可参考[多Flavor打包](https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97#%E5%A4%9Aflavor%E6%89%93%E5%8C%85)。

## tinker是否兼容加固？
由于各个厂商的加固实现并不一致，在1.7.6以及之后的版本，tinker不再支持加固的动态更新。

## Google Play版本是否可以有Tinker相关代码？
由于Google play的使用者协议，对于GP渠道我们不能使用Tinker动态更新代码，这里会存在应用被下架的风险。**但是在Google play版本，我们依然可以存在Tinker的相关代码，但是我们需要屏蔽补丁的网络请求与合成相关操作。**

## tinker与instant run的兼容问题？
事实上，若编译时都使用assemble*, tinker与instant run是可以兼容的。但是不少用户基础包与补丁包混用两种模式导致补丁过大，所以tinker编译时禁用instant run，我们可以在设置中禁用instant run或使用assemble方式编译。

大家日常debug时若想开启instant run功能，可以将tinker暂时关闭：

```xml
ext {
    //for some reason, you may want to ignore tinkerBuild, such as instant run debug build?
    tinkerEnabled = false
}
```

## 每次编译我应该保留哪些文件，如何兼容AndResGuard？
正如sample中[app/build.gradle](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/build.gradle)，每个可能用到Tinker发布补丁的版本，需要在编译后保存以下几个文件：

1. 编译后生成的apk文件，即用来编译补丁的基础版本；
2. 若使用proguard混淆，需要保持mapping.txt文件；
3. 需要保留编译时的R.txt文件；
4. 若你同时使用了资源混淆组件[AndResGuard](https://github.com/shwenzhang/AndResGuard), 你也需要将混淆资源的resource_mapping.txt保留下来，同时将`r/*`也添加到res pattern中。具体我们可以参考[build.gradle](https://github.com/dodola/tinker/blob/add5a7dc9f066cf8f1fd476c9ae1f44d210cb2aa/tinker-sample-android/app/build.gradle)。

微信通过将补丁编译与Jenkins很好的结合起来，只需要点击一个按钮，即可方便的生成补丁包。
	
## tinkerId应该如何选择？
tinkerId是用了区分基准安装包的，我们需要严格保证一个基准包的唯一性。在设计的初期，我们使用的是基准包的CentralDirectory的CRC，但某些APP为了生成渠道包会对安装包重新打包，导致不同的渠道包的CentralDirectory并不一致。

编译补丁包时，我们会自动读取基准包AndroidManifest的tinkerId作为package_meta.txt中的TINKER_ID。将本次编译传入的tinkerId, 作为package_meta.txt中的NEW_TINKER_ID。当前NEW_TINKER_ID并没有被使用到，只是保留作为配置项。如果我们使用git rev作为tinkerid, 这时只要使用`git diff TINKER_ID NEW_TINKER_ID`即可获得所有的代码差异。

**我们需要保证tinkerId一定是要唯一性的，这里推荐使用git rev或者svn rev. 如果我们升级了客户端版本，但tinkerId与旧版本相同，会导致可能会加载旧版本的补丁。这里我们一定要注意，升级可客户端版本，需要更新tinkerId!**

## 如何使生成的补丁包更小？
对于代码来说，我们最好记住以下几条规则：

1. 编译补丁包时，proguard使用applymapping模式；
2. 对于多dex的情况，`保持原本的分包规则`，尽量减少由于分包变化而带来的变更。在生成补丁包过程中，对于class分包的变化将会输出`Warning:Class Moved`日志, 我们应该尽量减少这种变化；
3. 大量静态常量的改变与资源R文件的变更,这里我们推荐使用applyResouceMapping方式保持资源ID。大量类分包的改变对补丁包的影响不大，但是对于`合成的时间消耗`与`占ROM的体积`影响更大。我们每次生成补丁后，都应该查看`TinkerPatch`输出文件夹的日志；
4. 其他的例如使用force jumbo模式以及使用7zip压缩补丁包。

## 关于使用的ClassLoader问题？
Tinker没有使用parent classloader方案，而是使用Multidex插入dexPathList方式，这里主要考虑到分平台内部类可能存在校验classloader的问题。

1. 若SDK>=24, 即Android N版本，当补丁存在时，我们将PathClassloader替换为AndroidNClassLoader, 但是它依然继承与PathClassLoader。我们依然可以像以往那样对它进行类似makeDexElements的操作。；
2. 若SDK<14, 我们没有对classloader做处理，这里需要注意补丁的Dex是插入在dexElement的前方。

## 什么时候调用installTinker？
首先我们推荐在最开始的时候就是执行installTinker操作，但是即使你不去installTinker，也不会影响Tinker对代码、So与资源的加载。installTinker只是做了以下几件事件：

1. 回调LoadReporter，返回加载结果；
2. 初始化各个自定义类与Tinker实例，可以调用Tinker相关API，发起升级补丁以及处理相关的回调。

事实上，微信只在主进程与:patch进程执行installTinker操作。其他进程只要不处理回调结果，不发起补丁请求即可。在[SampleUncaughtExceptionHandler](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/crash/SampleUncaughtExceptionHandler.java)中，为了防止Crash时并没有执行`installTinker`，全部使用的是[TinkerApplicationHelper](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/tinker/TinkerApplicationHelper.java)中的API，详细可以查看[Tinker API概览](https://github.com/Tencent/tinker/wiki/Tinker-API%E6%A6%82%E8%A7%88)。

## Proguard 5.2.1 applymapping出现Warning？
这是因为5.2.1增加了内联函数的行输出信息导致，你可以使用以下几种方法解决：

1. 使用5.1版本proguard;
2. 将内联函数的优化关掉；
3. 自己对mapping文件去除内联函数的行信息。 

**如果使用 4.X 版本的 Proguard 强烈建议升级到 5.1 版本。可以先下载 5.1的 Proguard， 然后通过以下方式指定：**

```
 classpath files('proguard-5.1.jar')
```

若使用gradle编译，与multiDexKeepProguard不同，我们无需将生成的tinker_proguard.pro拷贝到自己的配置中。另外一个方面，若applymapping过程出现冲突，我们可以采取以下几个方法：

1. 添加ignoreWarning；需要注意的是如果某些类的确需要采用新的mapping，这样补丁后App会出问题，一般我们并不建议采用这种方式；
2. 修改基准包的mapping文件；我们需要根据新的mapping文件，修正基准包的mapping文件。例如将warning项删掉或者将新mapping中keep的项复写到基准的mapping中。可以参考脚本[proguard_warning.py](https://github.com/Tencent/tinker/blob/master/tinker-build/tinker-patch-cli/tool_output/proguard_warning.py)与[merge_mapping.py](https://github.com/Tencent/tinker/blob/dev/tinker-build/tinker-patch-cli/tool_output/merge_mapping.py)。

**注意，如果想通过直接删除旧mapping文件的冲突项，需要注意删除类的内部类是否存在混淆冲突**

## TinkerPatch补丁管理后台与Tinker的关系？
[TinkerPatch平台](http://www.tinkerpatch.com) 是第三方开发基于CDN分发的补丁管理后台。它提供了补丁后台托管，版本管理，一键傻瓜式接入等功能，让我们可以无需修改任何代码即可轻松接入Tinker。

我们可以根据自己的需要选择接入，它是独立于Tinker项目之外。对于`Tencent/tinker`, 我们依然会以它的稳定性与性能作为第一要务。

## Tinker的最佳实践？
为了使补丁的成功率更高，我们在Sample中还做了以下工作：

1. 由于合成进程可能被各种原因杀死，使用[UpgradePatchRetry.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/util/UpgradePatchRetry.java)来做重试功能，提高成功率；
2. 防止补丁后程序无法启动，使用[SampleUncaughtExceptionHandler.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/crash/SampleUncaughtExceptionHandler.java)做crash启动保护。`这里更推荐的是进入安全模式`，使用配置的方式强制清理或者升级补丁；
3. 为了防止BuildConfig的改变导致大量类的变更，使用[BuildInfo.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/app/BuildInfo.java)非final的变量来中转。
4. 为了加快补丁应用同时保持用户体验，在[SampleResultService.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/service/SampleResultService.java)在应用退入后台或手机灭屏时，才杀掉进程。你也可以在杀掉进程前，直接通过发送broadcast或service intent的方式尽快的重启进程。
5. 把jumboMode打开，防止由于字符串增多导致force-jumbol，导致更多的变更。
6. 使用zip comment方式生成渠道包。

**更多的使用范例，大家请仔细阅读Sample。Tinker框架支持高度自定义，若使用过程中有任何问题或建议，欢迎联系我们!**