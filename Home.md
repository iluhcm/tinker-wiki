Tinker -- 微信Android热补丁方案          
====================================
[![license](http://img.shields.io/badge/license-BSD3-brightgreen.svg?style=flat)](http://git.code.oa.com/tinker/tinker/blob/master/LICENSE)

## Tinker是什么
Tinker是微信官方的Android热补丁解决方案，它支持动态下发代码、So库以及资源，让应用能够在不需要重新安装的情况下实现更新。当然，你也可以使用Tinker来更新你的插件。

它主要包括以下几个部分：

1. gradle编译插件: `tinker-patch-gradle-plugin` 
2. 核心sdk库: `tinker-android-lib`
3. 非gradle编译用户的命名行版本: `tinker-patch-cli.jar`

## 为什么使用Tinker
市面的热补丁方案有很多，其中比较出名的有淘宝的Dexposed、支付宝的AndFix以及QZone的超级补丁方案。但它们都存在无法解决的问题，这也是正是我们推出Tinker的原因。

|                  | Tinker     | QZone   | AndFix    | Dexposed   | 
| ---------------- | -------------- | ----------| ----------  |  ---------- |
| 类替换            | yes            | yes       | no          |  no         |
| So替换            | yes            | no        | no          | no          |
| 资源替换           | yes            | yes       | no          | no          |
| 全平台支持         | yes            | yes       | yes         | no         |
| 即时生效           | no             | no        | yes         | yes        |
| 性能损耗           | 较小              | 较大       | 较小         |  较小    |
| 补丁包大小          | 较小              | 较大         | 一般         |  一般    |
| 开发透明           | yes            | yes       | no         | no         |
| 复杂度             | 较低              | 较低       | 复杂          | 复杂     |
| gradle支持         | yes            | no        | no         | no         |
| 接口文档           | 丰富              | 较少        | 一般         | 较少      |
| Rom体积           | Dalvik较大             | 较小        | 较小         | 较小      |
| 成功率           | 较高             | 最高        | 一般         | 一般      |

**总的来说:**

1. Dexposed无法支持全平台，并不适合应用到商业产品中。
2. AndFix作为native解决方案，首先面临的是稳定性与兼容性问题，更重要的是它无法实现类替换，它是需要大量额外的开发成本的。
3. QZone方案主要问题是插桩带来Dalvik的性能问题，以及为了解决Art下内存地址问题而导致补丁包急速增大的。

Tinker热补丁方案不仅支持类、So以及资源的替换，它还是2.X－7.X的全平台支持。它无需插桩，补丁大小也远远小于其他方案。Tinker已运行在微信的数亿Android设备上，那么为什么你不使用Tinker呢？

## 如何使用Tinker
1. 如何快速接入请参考[Tinker 接入指南](https://github.com/Tencent/tinker/wiki/Tinker-%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97)。
2. 如何自定义类请参考[Tinker 自定义扩展](https://github.com/Tencent/tinker/wiki/Tinker-%E8%87%AA%E5%AE%9A%E4%B9%89%E6%89%A9%E5%B1%95)。
3. Tinker的API预览请参考[Tinker API预览](https://github.com/Tencent/tinker/wiki/Tinker-API%E6%A6%82%E8%A7%88)。
4. 还要其他问题，请参考[常见问题](https://github.com/Tencent/tinker/wiki/Tinker-%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98)。

如有其他疑问，欢迎联系我们!

## Tinker的TODO
Tinker经过几次全量上线，也发现了一些热补丁的问题。有以下的一些优化工作尚未完成：

1. Keep dex apply插件:即保证编译补丁包时使用与基础包一样的分包方案，减少由于dex移动导致的变化；
2. 资源全量Test，直接使用基础包的resources.arsc生成public.xml，无须applyResourceId；
3. 假设利用补丁发布功能，即使分平台合成的dex也可能较大，这个时候可以在厂商OTA后增加启动过渡页；

## 更多文章
关于Tinker与其他热补丁方案的具体对比或Tinker的实现原理，可参考以下文章：

1. [微信Android热补丁实践演进之路](https://github.com/WeMobileDev/article/blob/master/%E5%BE%AE%E4%BF%A1Android%E7%83%AD%E8%A1%A5%E4%B8%81%E5%AE%9E%E8%B7%B5%E6%BC%94%E8%BF%9B%E4%B9%8B%E8%B7%AF.md)  

2. [Android N混合编译与对热补丁影响解析](https://github.com/WeMobileDev/article/blob/master/Android_N%E6%B7%B7%E5%90%88%E7%BC%96%E8%AF%91%E4%B8%8E%E5%AF%B9%E7%83%AD%E8%A1%A5%E4%B8%81%E5%BD%B1%E5%93%8D%E8%A7%A3%E6%9E%90.md)   

3. [Dev Club 微信热补丁Tinker分享](http://dev.qq.com/topic/57ad7a70eaed47bb2699e68e)   