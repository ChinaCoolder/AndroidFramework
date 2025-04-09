---
theme: smartblue
---
本文将介绍有关Android Launcher启动相关的一些知识。

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**

**在阅读本篇建议先阅读:**

> System Server
> 
>https://juejin.cn/post/7345082426995556363

# 什么是Launcher
Launcher的概念，应该做应用层开发的都不会陌生，它可以说对用户来讲最重要的一个东西了，就好像Windows的桌面一样，也是我们其它应用主要的入口。

Launcher本质上来讲，也只是我们的一个Android应用，既然是应用，就肯定要被启动，那么问题来了，谁启动了Launcher?

# 谁启动了Launcher
我想这个问题肯定立刻就有人抢答了，要么说是`System Server`，要么说是`ActivityManagerService`，那么真的是这样吗? 就让我们接下来一探究竟。

# 启动源码分析
打开`/frameworks/base/services/java/com/android/server/SystemServer.java`，如果了解了`SystemServer.java`这个类的话，就应该知道它的主要工作就是启动一系列的重要Service，找到`startOtherServices`方法，拉到最后，可以找到如下代码:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9f8eff7e0d64bcba2078de5fbc4206a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=828&h=383&s=58529&e=png&b=fffefe)

可以看到，在这里`SystemServer`调用了`ActivityManagerService`的`systemReady`方法，进入此方法，其它的先不管，找到这段代码:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d324a033439d42749e105c90e965cb38~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=791&h=228&s=35272&e=png&b=fffefe)

可以看到这里调用了`mAtmInternal.startHomeOnAllDisplays`，这个`mAtmInternal`其实是一个`ActvityTaskManagerService`实例，我们进入`ActvityTaskManagerService`的`startHomeOnAllDisplays`这个方法，然后可以看到它依次调用了:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/384ae0a73ca442b192db258ac4c0c1b8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=670&h=174&s=19117&e=png&b=fffefe)


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b4c9bb0c53db472cad54eeafc415ab6a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=649&h=211&s=26762&e=png&b=fffefe)


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/04af6935d68a452794c54729a45a980a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=793&h=149&s=19073&e=png&b=fffefe)


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6380fc320a449c59b6753a43328bf72~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=806&h=335&s=50200&e=png&b=fef7f6)

最终，我们进入`RootWindowContainer`的`startHomeOnTaskDisplayArea`方法:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73c155aace424d74ad356b35e347e762~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=807&h=377&s=55240&e=png&b=fffefe)

可以看到，这个方法的主要内容就是构造了一个`homeIntent`，最后调用`ActivityStartController`的`startHomeActivity`方法，将`Intent`发送了出去:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/284d601d40a642fe837e2d6a6ede8bbf~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=802&h=852&s=117889&e=png&b=fffefe)

我们再看里面的`getHomeIntent`方法，可以看到确实加了Home的Category:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51dcb411e0754f19ad41a79a93ebb965~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=771&h=246&s=30480&e=png&b=ffffff)

接下来，应该就是启动Luancher了吧。

# Log分析
好，我们接下来验证一下。

首先打开我们的模拟器，等待它进入到Launcher之后，接着进入`adb shell`，输入`logcat | grep "ActivityTaskManager: START"`，然后可以看到如下的日志：

```log
03-13 17:51:41.529   509   509 I ActivityTaskManager: START u0 {act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000100 cmp=com.android.settings/.FallbackHome (has extras)} with LAUNCH_MULTIPLE from uid 0 (BAL_ALLOW_ALLOWLISTED_UID) result code=0
03-13 17:51:35.896   509   937 I ActivityTaskManager: START u0 {act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000100 cmp=com.android.launcher3/.uioverrides.QuickstepLauncher (has extras)} with LAUNCH_SINGLE_TASK from uid 0 (BAL_ALLOW_ALLOWLISTED_UID) result code=0

```
可以看到，我们启动的第一个Activity，并不是Luancher而是`com.android.settings.FallbackHome`，那么问题就来了，`FallbackHome`是什么?

# FallbackHome是什么
我们打开`/packages/apps/Settings/src/com/android/settings/FallbackHome.java`，先看看它的布局文件:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c2e34c42e32246f5b0a848653de2962f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=671&h=731&s=85688&e=png&b=ffffff)

好像也没什么特别内容，再看看它的代码，尤其注意以下三块：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e34a987ad2744f3892ec0c2c0340f41c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=764&h=87&s=6990&e=png&b=ffffff)


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a789f4ddccf44b68cbe3ab164cd7b49~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=581&h=193&s=16739&e=png&b=fffefe)


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7986708b6da741cea938b820b046e94d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=452&h=180&s=14906&e=png&b=fffefe)

可以看到多处在尝试调用这个`maybeFinish`，那么我们再看下这个方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b8046bf869a479ea2ed2edabcbe7561~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=806&h=367&s=52467&e=png&b=fffefe)

这里的核心逻辑，其实就是调用`isUserUnlocked`这个方法，然后只要返回`true`，那么就再发送一个Home Intent，从上面的Activity启动log我们可以得知，这里的Home Intent，就会被Launcher拦截了。

那么`isUserUnlocked`到底代表什么? 其实这代表着设备的解密，这也是为什么我们需要`FallbackHome`的原因。

# 为什么需要FallbackHome
如果想要知道这个，首先我们就要了解什么是`文件级加密`和`直接启动`，这里是官方文档:

> 文件级加密
>
> https://source.android.com/docs/security/features/encryption/file-based?hl=zh-cn 

> 直接启动
> 
> https://developer.android.com/privacy-and-security/direct-boot?hl=zh-cn

简单来说，在Android 7以后，Android开始支持文件级加密，Android 10以后，更是开始强制使用文件级加密，文件级加密将存储分成了两部分，凭据加密部分和设备加密部分，以下是两者区别:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a859b2076676446e8a6ba00faf946a75~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=660&h=130&s=31161&e=png&b=fffefe)

对于我们的设备来讲，刚开始开机的时候，系统属于未解密状态，因为Launcher可能与其它的很多应用会产生数据的交互，需要读取凭据加密存储空间，所以默认情况下无法支持直接启动，那么我们就需要一个loading页面作为中间页，让他来显示等待页面，等待解密完成后，由它来启动Launcher。

我们可以打开`/packages/apps/Settings/AndroidManifest.xml`，可以看到整个`FallBackHome`所属于的整个`Setting`应用都增加了对直接启动模式的支持:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f7f46b0296449438093526a6ba8f17e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=658&h=349&s=46670&e=png&b=fffefe)

至此，我们就完成了对Launcher启动的整个源码分析。