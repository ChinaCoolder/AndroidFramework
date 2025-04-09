---
theme: smartblue
---
本文将介绍有关Android System Server相关的一些知识。

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**

**在阅读本篇建议先阅读:**

> Zygote
> 
> https://juejin.cn/post/7344571269554667583

> Linux `fork`概念
> 
> https://juejin.cn/post/6912612368996368392?searchId=20240311142552A134B99FB0A4E3BF8EDD

本篇依旧会只展示关键代码，所以最好打开AOSP源码一步步的跟着看，才能理解整个的流程。

# 什么是System Server
在Android的应用层开发中，我们会使用到一系列的service，比如`ActivityManagerService`，`PowerManagerService`等，System Server的作用，就是去启动这些Service。

# System Server进程启动分析
我们的System Server是从Zygote进程中`fork`出来的，下面，我就将一步步讲解Zygote启动System Server的整个流程。

首先，我们打开`/frameworks/base/core/java/com/android/internal/os/ZygoteInit`，找到它的`main`方法，然后可以找到如下代码:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cf18b1d64734983aa45ef627026c12c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=800&h=239&s=19845&e=png&b=fffefe)

这里，就是我们System Server启动的关键部分，从代码中我们可以得知，只有`startSystemServer`这个变量为`true`的时候，执行我们启动System Server的逻辑，在往上，我们可以看到:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/794f8c6cb49f4df8babef642dfc50e2d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=655&h=407&s=55962&e=png&b=ffffff)

这个变量的值，是由外部参数决定的，这个参数其实在我们的Zygote启动的时候，就已经传进来了，可以打开`/system/core/rootdir/init.zygote64.rc`，可以看到对应的参数:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80f31a6e9e5541d3b2e2af15540b055e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=978&h=189&s=37581&e=png&b=ffffff)

那么`forkSystemServer`这个方法究竟是做什么的，其实从名字上就可以判断出来，这个方法就是从Zygote去`fork`出我们的System Server的进程的。

首先进入这个方法，其它的先不管，找到这段代码:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4af6348d83774a6882da8343cf0ef1ff~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=413&h=187&s=16851&e=png&b=fffefe)

再进入`Zygote`类的`forkSystemServer`方法:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/15eb5385465c4902a28f83e6312a8ddb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=725&h=329&s=37368&e=png&b=fffefe)

就可以看到实际上我们是通过Native层去`fork`我们的进程的，打开`/frameworks/base/core/jni/com_android_internal_os_Zygote.cpp`，找到`com_android_internal_os_Zygote_nativeForkSystemServer`，这就是我们的Native层的实际调用。

然后在这个方法里面，我们可以看到它调用了`zygote`的`ForkCommon`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cba2e0a8b7084478af3e169747da8fff~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=670&h=522&s=62607&e=png&b=fffefe)

再进入这个方法，在中间我们就可以找到`fork`的调用了:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f4903ae52d44cbb9ba4393460e0f629~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=711&h=277&s=32696&e=png&b=fefdfd)

到这一步，其实我们的System Server进程就已经`fork`出来了，但问题是目前这个进程还没有我们System Server相关的内容，那么接下来，我们就要开始真正启动我们的System Server了。

# System Server! 启动!
打开`/frameworks/base/core/java/com/android/internal/os/ZygoteInit`，继续看它的`forkSystemServer`方法，首先我们看这一部分:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dc6a3dd6df8544069c68eebc38868e50~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=813&h=286&s=30039&e=png&b=a5d2ff)

从注释里面我们也能看出来，这是System Server的启动参数，比如`--nice-name=system_server`，这个其实就是进程名，`com.android.server.SystemServer`，其实就是我们的启动类了。

再往下看，可以看到这部分代码:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c55f22aa5340450294d2cc09c1a9f4b2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=537&h=462&s=49077&e=png&b=fffefe)

从刚才我们就已经知道，`Zygote.forkSystemServer`，是帮我们`fork`进程的，那么下面这段代码，只有子进程，也就是我们刚刚`fork`出来的System Server才会执行，如果不明白这部分逻辑的，可以看下开头的`fork`相关概念。

那我们看下这段代码，首先，System Server进程做了一个判断，如果有32位的Secondary Zygote，那么就等待，等待Secondary Zygote启动之后再进行下一步，接着，就关闭了本进程的`zygoteServer`，因为被`fork`出来的System Server子进程不需要通过`LocalSocket`来`fork`应用进程，再然后，会调用`handleSystemServerProcess`这个方法：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2b814d8ebf5443b816f1971d30f2a37~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=777&h=410&s=65426&e=png&b=fffefe)

然后我们直接看这段代码，然后再依次进入`ZygoteInit.zygoteInit`，`RuntimeInit.applicationInit`，`findStaticMain`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/251286454f5049fa9e39e0d379c639fe~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=617&h=251&s=29182&e=png&b=fffefe)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/483a50744b0349e690b70d8280f96f4a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=715&h=311&s=46001&e=png&b=fffefe)


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/111189dfdd574f098d6655e4de02eda5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=779&h=455&s=68390&e=png&b=fffefe)


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c5e6ba73eeb4f1b8126321229169a6a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=646&h=298&s=30974&e=png&b=ffffff)

最后，在`findStaticMain`中，找到这部分代码:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29ebe2d3f1f144e0a8964105c754f3fc~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=589&h=168&s=18409&e=png&b=fffefe)

可以看到，这就是我们最终返回的`Runnable`了，那么这个类主要是做什么的呢? 其实就是执行我们传给他的`startClass`的`main`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be534d27f7b849b7a55af6a55f7482e3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=601&h=709&s=66874&e=png&b=ffffff)

最后这个`Runnable`的执行，可以在`/frameworks/base/core/java/com/android/internal/os/ZygoteInit`的`main`方法里面，`forkSystemServer`调用的后面找到:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/800b787898be404e9312264bd2bead89~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=787&h=231&s=19650&e=png&b=fffefe)

在注释里我们也可以看到，这部分代码是由`system_server`，也就是Zygote`fork`出来的System Server进程，去执行的。

那么接下来，我们就要看看我们的`com.android.server.SystemServer`的`main`方法，看看它到底做了什么。

# System Server启动其它服务
打开`/frameworks/base/services/java/com/android/server/SystemServer.java`，找到它的`main`方法，可以看到他就是调用了自己这个类的`run`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5194a46174fd44a5bf203d5177a2b386~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=399&h=82&s=6852&e=png&b=ffffff)

我们再进入`run`方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/62124c5f13a84bc896dd9bee7f5719cb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=601&h=263&s=41179&e=png&b=fffefe)

找到如下的代码块:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce151b7e42414a6f8f5ac52d2dcf5e27~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=748&h=385&s=43908&e=png&b=fffefe)

这里，就可以看到我们启动服务的地方了，可以看到，服务被分为了4类：`BootstrapServices`，`CoreServices`，`OtherServices`和`ApexServices`，比如我们熟悉的`ActivityManagerService`，他就被放在了`startBootstrapServices`中启动。

然后我们回到刚才`run`方法的最后，可以看到在这里，System Server进入了loop状态，而所有的这些Service就都会运行在`system_server`这个进程里面了。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e799ce74dd5f4f91be6066cf35975654~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=662&h=103&s=9147&e=png&b=ffffff)

打开adb shell，输入`ps -A | grep system_server`，就可以看到我们的System Server进程了:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/753e7eb044524fe4b5293d02f2851277~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=678&h=42&s=7235&e=png&b=fffefe)

到这里，我们就完成了整个System Server的启动分析，和它启动其它Service的分析，那么接下来，我们就要去看看，我们的Launcher是如何启动的了。