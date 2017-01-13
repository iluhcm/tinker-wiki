Tinker API概览          
====================================
我们需要使用的API大约几种在以下几个类中：

| 函数               | 描述       | 
| ----------------- | ---------  |
| [TinkerInstaller.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/tinker/TinkerInstaller.java)      |TinkerInstaller.java封装了一些常用的函数，例如Tinker对象的构建，发起补丁请求以及lib库的加载。 |
| [Tinker.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/tinker/Tinker.java)      | Tinker.java是Tinker库的Manager类，tinker所有的状态、信息都存放在这里。 | 
| [TinkerLoadResult.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/tinker/TinkerLoadResult.java)      |TinkerLoadResult.java是用来存放加载补丁包时的相关结果，它本身也是Tinker.java的一个成员变量。 |
| [TinkerApplicationHelper.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/tinker/TinkerApplicationHelper.java)      |TinkerApplicationHelper.java封装了一些无需构建Tinker都可调用的函数，一般我们更推荐使用上面的三个类。|
| [TinkerLoadLibrary.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/library/TinkerLoadLibrary.java)      |TinkerLoadLibrary.java封装了一些反射或者加载补丁Library的方法。|

## TinkerInstaller相关接口
TinkerInstaller封装了一些比较重要的函数，现作简单的说明：

### Tinker实例的构建
Tinker类是整个框架的核心，我们需要构建它的单例。其中intentResult存放是我们加载补丁时的数据，它在install之后才将数据赋值给Tinker与TinkerLoadResult类。

全部使用默认定义类的构造方法：

```java
public static void install(ApplicationLike applicationLike) {
	 Tinker.with(tinkerApplication).install(applicationLike.getTinkerResultIntent());
}
```

若使用了自定义类，可选择多参数的install方法，具体用法可参考[SampleApplicationLike](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/app/SampleApplicationLike.java)。

**你一定要先install Tinker之后，才能使用Tinker相关的API。不然你只能使用TinkerApplicationHelper.java中的API。**

### 发起补丁修复请求
正如之前所说的，所有的补丁升级请求都将会分发到PatchListener去处理。

发起升级补丁请求，即收到一个新的补丁包，多次补丁也是调用下面这个接口：

```java
public static void onReceiveUpgradePatch(Context context, String patchLocation) {
    Tinker.with(context).getPatchListener().onPatchReceived(patchLocation);
}
```
### Library库的加载
####不使用Hack的方式
更新的Library库文件我们帮你保存在tinker下面的子目录下，但是我们并没有为你区分abi(部分手机判断不准确)。所以若想加载最新的库，你有两种方法，第一个是直接尝试去Tinker更新的库文件中加载，第二个参数是库文件相对安装包的路径。

```java
TinkerLoadLibrary.loadLibraryFromTinker(getApplicationContext(), "assets/x86", "libstlport_shared");
```

但是我们更推荐的是，使用TinkerInstaller.loadLibrary接管你所有的库加载，它会自动先尝试去Tinker中的库文件中加载，但是需要注意的是`当前这种方法只支持lib/armeabi目录下的库文件`！

```java
//load lib/armeabi library
TinkerLoadLibrary.loadArmLibrary(getApplicationContext(), "libstlport_shared");
//load lib/armeabi-v7a library
TinkerLoadLibrary.loadArmV7Library(getApplicationContext(), "libstlport_shared");
```

若存在Tinker还没install之前调用加载补丁中的Library库，可使用[TinkerApplicationHelper.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/tinker/TinkerApplicationHelper.java)的接口

```java
//load lib/armeabi library
TinkerApplicationHelper.loadArmLibrary(tinkerApplicationLike, "libstlport_shared");
//load lib/armeabi-v7a library
TinkerApplicationHelper.loadArmV7Library(tinkerApplicationLike, "libstlport_shared");
```

若想对第三方代码的库文件更新，可先使用TinkerLoadLibrary.load\*Library对第三方库做提前的加载！更多使用方法可参考[MainActivity.java](https://github.com/Tencent/tinker/blob/master/tinker-sample-android/app/src/main/java/tinker/sample/android/app/MainActivity.java)。

####使用Hack的方式
以上使用方式似乎并不能做到开发者透明，这是因为我们想尽量少的去hook系统框架减少兼容性的问题。Tinker也提供了一键反射Library Path的方式供大家选择：

```java
// 将tinker library中的armeabi注册到系统的library path中。
TinkerLoadLibrary.installNavitveLibraryABI(context, "armeabi");
```

当然，当前手机系统的abi 需要大家自行判断传入即可，这样我们就无需再对library的加载做任何的介入。


### 设置LogIml实现
你可以设置自己的Log输出实现：

```java
public static void setLogIml(TinkerLog.LogImp imp) {
	TinkerLog.setLogImp(imp);
}
```

## Tinker类相关接口
Tinker类是整个库的核心，基本所有的补丁相关的信息我们都可以在这里获取到。TinkerApplication也有一些获取tinkerFlag的相关API，它们可以在Tinker未被初始化前使用。

### 获取单例的方法
获取单例的方法非常简单：

```java
Tinker manager = Tinker.with(context);
```   
### 获取加载状态
`loaded`成员变量是标记是否有补丁加载成功的标记，只有它为true时，才能保证TinkerLoadResult的各个变量非空。

```java
boolean isLoaded = Tinker.with(context).isTinkerLoaded();
boolean isInstalled = Tinker.with(context).isTinkerInstalled();

```

获得加载的结果，也就是TinkerLoadResult的实例。它是有可能为空的，使用它请先确保`loaded`为true：

```java
TinkerLoadResult loadResult = Tinker.with(context).getTinkerLoadResultIfPresent();
```
   
### 清除补丁
当补丁出现异常或者某些情况，我们可能希望清空全部补丁，调用方法为：

```java
Tinker.with(context).cleanPatch();
```   

当然我们也可以选择卸载某个版本的补丁文件：

```java
Tinker.with(context).cleanPatchByVersion();
```  

**在升级版本时我们也无须手动去清除补丁，框架已经为我们做了这件事情。需要注意的是，在补丁已经加载的前提下清除补丁，可能会引起crash。这个时候更好重启一下所有的进程。** 

其他API这里不再一一概述，请大家自行翻阅[Tinker.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/tinker/Tinker.java)。

## TinkerLoadResult相关接口
因为加载补丁是在Tinker的Install之前的，我们将加载的结果保存在intent中，然后在Tinker的install方法中恢复这些结果。这里包括补丁的所有信息，例如加载的dex，library，我们定义的package config以及各个文件的目录等。但是需要注意的是，这里面的变量大多数是nullable。若Tinker的loaded为true，除了`dexes与libs`(要看补丁包里面是否真的有)，其他变量可以确保非空。

获取packageConfig：

```java
public String getPackageConfigByName(String name) {
    if (packageConfig != null) {
        return packageConfig.get(name);
    }
    return null;
}
```  

**这里需要注意的是，检查dex的Md5值需要使用[SharePatchFileUtil.verifyDexFileMd5](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-loader/src/main/java/com/tencent/tinker/loader/shareutil/SharePatchFileUtil.java)方法,这是由于dex有可能是被我们重新打包成jar模式。**

```java
//获得基准包的tinkerId
String oldTinkerId = Tinker.with(context).getTinkerLoadResultIfPresent().getTinkerID();
//获得补丁包的tinkerId
String newTinkerId = Tinker.with(context).getTinkerLoadResultIfPresent().getNewTinkerID();
```  

更多接口请参考[TinkerLoadResult.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/tinker/TinkerLoadResult.java)。

## TinkeApplicationHelp相关接口
在有些时候，你可能想在更后的时机才去`installTinker`，甚至在某些进程永远也不会去做这个动作。那样你可以通过[TinkerApplicationHelper.java](https://github.com/Tencent/tinker/blob/master/tinker-android/tinker-android-lib/src/main/java/com/tencent/tinker/lib/tinker/TinkerApplicationHelper.java)来获得一些当前加载的信息。

基本所有的信息都可以在这里获得，但是由于这里是直接读取结果intent

**还有其他问题？请参考[Tinker 常见问题](https://github.com/Tencent/tinker/wiki/Tinker-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)!**