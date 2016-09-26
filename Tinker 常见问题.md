Tinker 常见问题
====================================
## Tinker编译相关问题？
编译过程相关的issue请先查看是否是以下情况：

1. `无法打开sample工程`： 请使用单独的IDE窗口打开tinker-sample-android工程；
2. `tinkerId is not set`: 这是因为没有正确的配置IDE的git路径, 这里你也可以使用其他字符作为tinkerId;
3. 对于编译与补丁时发生的异常，请到[Tinker 自定义扩展](https://github.com/Tencent/tinker/wiki/Tinker-%E8%87%AA%E5%AE%9A%E4%B9%89%E6%89%A9%E5%B1%95)中查看具体错误码的原因。并通过“Tinker.”过滤Tinker相关的日志提交到issue中。

在提交issue之前，我们应该先查询是否已经有相关的issue。提交issue时，我们需要写明issue的原因，以及编译或运行过程的日志(加载进程以及Patch进程)。

## Tinker库中有什么类是不能修改的？
Tinker库中不能修改的类一共有25个，即com.tencent.tinker.loader.*类。加上你的Appliction类，只有25个类是无法通过Tinker来修改的。即使类似Tinker.java等管理类，也是可以通过Tinker本身来修改。

## 什么类需要放在主dex中？
Tinker并不干涉你分包与多dex的加载逻辑，但是你需要确保以下几点：

1. com.tencent.tinker.loader.*类，你的Application类需要在主dex，并且已经在dex.loader中配置;
2. 若你自定义了TinkerLoader类，你需要将TinkerLoader的自定义类，以及它用的到类也放在主dex，并且已经在dex.loader中配置;
3. ApplicationLike的继承类也需要放在主dex中，但是它无须在dex.loader中配置，因为它是可以使用Tinker修改的类。最后`如果你需要在加载其他dex之前加载Tinker的管理类，你也可以将com.tencent.tinker.*都加入到主dex`。
4. 你的ApplicationLike实现类的直接引用类以及在调用Multidex install之前加载的类也都需要放到主dex中。
   
## 如何对Library文件作补丁？
当前我们并没有直接将补丁的lib路径添加到`DexPathList`中，理论上这样可以做到程序完全没有感知的对Library文件作补丁。这里主要是因为在多abi的情况下，某些机器获取的并不准确。**当前对Library文件作补丁可参考[Tinker API概览](https://github.com/Tencent/tinker/wiki/Tinker-API%E6%A6%82%E8%A7%88)，这里以后需要考虑优化。**

另外一方面，对于第三方库文件的加载我们无法干预，但是只要在我们的代码提前加载第三方的库文件即可。不过这里确保我们使用的是同一个classloader来加载。

无论是对Library还是Application，我们都是采用尽量少去反射的策略，这也是为了提高Tinker框架的兼容性。上线前，我们应当严格测试补丁是否正确加载了修改后的So库。**不使用反射的另外一个好处是我们可以做的工作更多，例如加载前验证它的MD5。**

## 如何对资源文件作补丁？
Tinker采用全量合成方式实现资源替换，这里有以下几点是使用者需要明确的：

1. remoteView是无法修改，例如transition动画，notification icon以及桌面图标;
2. Tinker只会将满足res pattern的资源放在最后的合成补丁资源包中。一般为了减少合成资源大小，我们不建议输入classes.dex或lib文件的pattern;
3. 若一个文件:assets/test.dex, 它既满足dex pattern, 又满足res pattern。Tinker只会处理dex pattern, 然后在合成资源包会忽略assets/test.dex的变更。library也是如此。

最后我们应该查看编译过程中生成的`resources_out.zip`是否满足我们的要求。 

## 每次编译我应该保留哪些文件？
正如sample中[app/build.gradle](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/build.gradle)，每个可能用到Tinker发布补丁的版本，需要在编译后保存以下几个文件：

1. 编译后生成的apk文件，即用来编译补丁的基础版本；
2. 若使用proguard混淆，需要保持mapping.txt文件；
3. 需要保留编译时的R.txt文件；
4. 若你同时使用了资源混淆组件[AndResGuard](https://github.com/shwenzhang/AndResGuard), 你也需要将混淆资源的mapping保留下来。

微信通过将补丁编译与Jenkins很好的结合起来，只需要点击一个按钮，即可方便的生成补丁包。

## 我应该使用哪个作为补丁包下发？
`patch_signed_7zip.apk`是已签名并且经过7z压缩的补丁包，但是你最好重命名一下，不要让它以`.apk`结尾，这是因为有些运营商会挟持以`.apk`结尾的资源。

另外一点，我们在发起补丁请求时，**需要先将补丁包先拷贝到dataDir中**。因为在sdcard中，补丁包是极其容易被清理软件删除。这里可以参考[UpgradePatchRetry.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/util/UpgradePatchRetry.java)的实现。

## tinkerId应该如何选择？
tinkerId是用了区分基准安装包的，我们需要严格保证一个基准包的唯一性。在设计的初期，我们使用的是基准包的CentralDirectory的CRC，但某些APP为了生成渠道包会对安装包重新打包，导致不同的渠道包的CentralDirectory并不一致。

**我们需要保证tinkerId一定是要唯一性的，这里推荐使用git rev或者svn rev. 如果我们升级了客户端版本，但tinkerId与旧版本相同，会导致可能会加载旧版本的补丁.**

## Tinker中的dex配置'raw'与'jar'模式应该如何选择？
它们应该说各有优劣势，大概应该有以下几条原则：

1. 如果你的minSdkVersion小于14, 那你务必要选择'jar'模式；
2. 以一个`10M`的dex为例，它压缩成jar大约为`4M`，即'jar'模式能节省`6M`的ROM空间。
3. 对于'jar'模式，我们需要验证压缩包流中dex的md5,这会更耗时，在`小米2S`上数据大约为'raw'模式`126ms`, 'jar'模式为`246ms`。

因为在合成过程中我们已经校验了各个文件的Md5，并将它们存放在/data/data/..目录中。`默认每次加载时我们并不会去校验tinker文件的Md5`,但是你也可通过开启loadVerifyFlag强制每次加载时校验，但是这会带来一定的时间损耗。

**简单来说，'jar'模式更省空间，但是运行时校验的耗时大约为'raw'模式的两倍。如果你没有打开运行时校验，推荐使用'jar'模式。**

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

1. 使用5.1版本点proguard;
2. 将内联函数的优化关掉；
3. 自己对mapping文件去除内联函数的行信息。 

## Tinker的最佳实践？
为了使补丁的成功率更高，我们在Sample中还做了以下工作：

1. 由于合成进程可能被各种原因杀死，使用[UpgradePatchRetry.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/util/UpgradePatchRetry.java)来做重试功能，提高成功率；
2. 防止补丁后程序无法启动，使用[SampleUncaughtExceptionHandler.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/crash/SampleUncaughtExceptionHandler.java)做crash启动保护。`这里更推荐的是进入安全模式`，使用配置的方式强制清理或者升级补丁；
3. 为了防止BuildConfig的改变导致大量类的变更，使用[BuildInfo.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/app/BuildInfo.java)非final的变量来中转。
4. 为了加快补丁应用同时保持用户体验，在[SampleResultService.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/service/SampleResultService.java)在应用退入后台或手机灭屏时，才杀掉进程。你也可以在杀掉进程前，直接通过发送broadcast或service intent的方式尽快的重启进程。
5. 把jumboMode打开，防止由于字符串增多导致force-jumbol，导致更多的变更。
6. 使用zip comment方式生成渠道包。

**更多的使用范例，大家请仔细阅读Sample。Tinker框架支持高度自定义，若使用过程中有任何问题或建议，欢迎联系我们!**