---
theme: smartblue
---
本文将介绍有关Android Zygote相关的一些知识。

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**

**在阅读本篇建议先阅读:**


> **Android Init Language**
>
> https://juejin.cn/post/7341668771910811675

> **Init进程:**
> 
> https://juejin.cn/post/7343889290432839691


相信如果是做了比较长时间App开发的话，对这个词应该是不陌生的，可能面试得时候也会被经常问到。

我看了很多Zygote相关的博客，要么是大段大段的抽象概念，感觉像是在应付面试的，要么就是大段大段的贴代码，各种跳转看的人头晕眼胀，所以我想尽量以详尽的叙述和少量关键代码的形式来讲讲，什么是Zygote。

# 什么是Zygote
Zygote翻译过来就是受精卵的意思，这个名字其实就已经说明了它的作用，是为了诞生"生命"用的。

那么对于Android来说，什么是最重要的"生命"? 那当然是我们的一个又一个App了。

打开模拟器，输入`adb shell`，然后列出所有进程`ps -A`，找到我们的`zygote64`的`pid`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d4164752a494228aa4774699ea3b9e0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=755&h=55&s=9001&e=png&b=300a25)

然后输出`ps -A | grep 284`，就可以看到好多`ppid`为284，也就是说由我们的Zygote孵化出来的进程了:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/149e0f2527494006bafa776c124d704b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=973&h=961&s=207951&e=png&b=310a25)

我们可以看到，这么多的进程，都是由Zygote孵化的，如果说Init是Android世界的起始点，那么Zygote可以说是Android 应用层的起始点了。

# 为什么我们需要Zygote
看这篇文章的人，应该都基本清楚字节码，虚拟机的这些概念，也能理解，我们的App里面的dex文件，最终是需要一个Dalvik虚拟机来运行的，还有每个App和他们的虚拟机，默认情况下应该就在单独一个进程里。

那么问题来了，如果我们每次启动一个应用，然后每次都从头开始一个进程，然后这个进程去做虚拟机的初始化工作并启动虚拟机，那手机就慢死了，因为每个应用开启前都要进行大量的重复工作。

所以，最好能有一个进程，让他在系统启动的时候去完成这些繁杂的重复工作，把运行应用程序之前所需的一些系统资源加载到内存里，然后常驻在Android系统中，如果有新的应用要启动，那么直接从这个进程`fork`出来就好了，而且所有的这些预加载资源，会变成应用间共享的。

Zygote就是起这样一个作用的。

那么现在，我们知道了Zygote的作用和行为，接下来就要看看，Zygote是怎么完成这一系列工作得了。

# 谁启动了Zygote
Zygote是由Init进程启动的，打开`/system/core/rootdir/init.rc`，可以看到如下的代码，在`late-init`中，trigger了`zygote-start`:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51613cda86d54623a204e8af0e4386ea~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=711&h=613&s=94171&e=png&b=ffffff)

然后在`zygote-start`中，启动了`zygote`和`zygote_secondary`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc5916aa515f4e9ba92fd34b0233fddd~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=498&h=158&s=18007&e=png&b=ffffff)

这里就会有一个疑问了，怎么出来了2个zygote，还有上面的进程图里面，也出现了`zygote`和`zygote64`，这其实是因为我们两个版本的zygote一个是对32位的，一个是对64位的，我们可以打开
`/system/core/rootdir/init.zygote64_32.rc`，就可以看到`zygote_secondary`，就是32位版本的zygote:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bab93caf68db45748c9e54ff14d27362~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1168&h=249&s=45306&e=png&b=ffffff)

根据最上面我们的进程图可以看到，基本上目前的进程主要是由`zygote64`进程`fork`来的，这个zygote，也就是我们的primary zygote，即主要的zygote，32版本的zygote，即是secondary zygote，即次要的zygote。

那么接下来我们就要找到`late-init`是在哪里被trigger的，其实打开`/system/core/init/init.cpp`的`SecondStageMain`方法，就可以找到了:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/52fb4a76c0884e7d9554c04f43c4554f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=664&h=208&s=29978&e=png&b=fffefe)

# Zygote Native起始工作流程
首先打开`/frameworks/base/cmds/app_process/app_main.cpp`，这个类便是Zygote Native的起始点，找到`main`方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d14c92b16a1c4d30b29df5a4f9cc0026~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=761&h=336&s=38228&e=png&b=fffefe)

然后直接看`main`方法的最后一行，可以看到这三行代码:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b7cd745ab0246eabe9eed632e30c77a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=772&h=208&s=32416&e=png&b=fffefe)

可以看到，这里进行了一个判断，我们先看自己最关心的，第一行里面，好像是和Zygote最相关的，如果希望运行到这里，需要`zygote`为`true`，那么这个变量，到底什么情况下会为`true`?

再往上翻，就可以看到，`zygote`是在什么情况下设置为`true`了:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1c83584111d24ae983da60bee03738be~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=527&h=423&s=42031&e=png&b=ffffff)

再上面的代码逻辑，我就不贴出来了，主要是处理一些传进来的参数，我们从这里面的代码逻辑，就能看到，当我们的`main`函数在有`--zygote`这个参数的时候，`zygote`就会被设置为`true`，那么这个参数什么时候会被传进来?

其实在Init进程启动Zygote的时候，就会传进来了，打开`/system/core/rootdir/init.zygote64.rc`，我们可以看到被传递进来的参数里，就有`--zygote`：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3a5d6fbd6b4f40dbbe386aaa1def5be3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=963&h=160&s=30156&e=png&b=ffffff)

那么我们就可以得出一个结论，在刚开始的时候
```c++
runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
```

这段代码，就会被执行。

接下来的问题就是，这段代码代表着什么?

首先，我们来看`runtime`的定义，在`main`函数的开始部分，我们就可以看到:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42794097be414d598f5e9535e2babe32~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=740&h=85&s=12506&e=png&b=fefcfc)

然后我们再看下同个文件的`AppRuntime`这个类:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3388f32644e841dcb211e7af70786bc4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=743&h=364&s=33492&e=png&b=ffffff)

会发现这个类总共就80多行，看起来也没什么，下面我们再看看他的父类`AndroidRuntime`，具体在`/frameworks/base/core/jni/AndroidRuntime.cpp`，这个类里面，我们就可以看到大量虚拟机相关的代码了，比如`startVm`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64703459fec34e27a67163c2d14f1aee~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=826&h=250&s=62086&e=png&b=fffefe)

好了，我们再回到刚才的代码，进入`start`方法，进入之后可以看到最后跳转的是`AndroidRuntime`的`start`方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac1c5f96ad3a4742bc61747e7ece0462~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=813&h=244&s=32551&e=png&b=fffefe)

其它代码暂时不用关心，往下翻，看到这里：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5e7280d0ea6472f96570575dac7061d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=675&h=215&s=20603&e=png&b=a6d2ff)

从这里就可以看到，我们进入了上面提到过的`startVm`方法，那么这个方法，其实就是我们启动虚拟机的地方了，进入这个方法里面，可以看到Android硬编码了一大堆虚拟机参数:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b57bc11d059f455496056fe68590d4e1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=796&h=632&s=185726&e=png&b=fffefe)

这个方法很长，中间我们都不需要关心，我们只需要知道，这个方法是用来启动虚拟机的就可以了，那么当虚拟机启动好了以后，我们还要做后续的工作。

我们再看最开始的这段代码:
```c++
runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
```
可以看到他传了一个参数，这好像是个Java类，而且从名称上判断，这应该和Zygote初始化有关系。

那么我们的虚拟机是什么时候处理这个参数的?

在`AndroidRuntime`的`start`方法中，在`startVm`再往下翻，就可以看到这样一段代码:


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c49a3f18a7d94e90973df0c172cee523~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=825&h=490&s=73193&e=png&b=fffefe)

这个`className`，从`start`开头的地方可以知道，就是我们传进来的`com.android.internal.os.ZygoteInit`，再往下翻，我们就可以看到处理的逻辑了:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3f81e7d100841329762789f6e1f6c2c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=755&h=408&s=55053&e=png&b=fffefe)

这段代码的意思，就是执行我们传进来的那个Java类的main方法了:
```c++
jmethodID startMeth = env->GetStaticMethodID(startClass, "main", "([Ljava/lang/String;)V");
env->CallStaticVoidMethod(startClass, startMeth, strArray);
```

接下来，我们要进入Java的世界了。

# ZygoteInit预加载资源
打开`/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`，并找到它的`main`方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1f83c9246034878aae672eaf573c4eb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=649&h=339&s=35354&e=png&b=fffefe)

找到这部分代码:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3c9a9b62e6540d895ae8a6a1403c882~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=799&h=271&s=47519&e=png&b=fffefe)

可以看到这个判断是，如果不启用lazy preload，那么就会开始预加载资源，启用了则跳过，然后再打开`/system/core/rootdir/init.zygote64.rc`，可以看到我们的primary zygote是没有启用lazy preload的，但是打开`/system/core/rootdir/init.zygote64_32.rc`我们就会看到我们的secondary zygote，启用了lazy preload，所以我们的primary zygote，也就是64位版本的zygote，是一定会执行`preload`的:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/56c46a7fac134e89b60effab0fea5a58~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=971&h=124&s=19253&e=png&b=fefefe)


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1dc03b44cac2433f9da6f8832b70236a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1142&h=95&s=12654&e=png&b=ffffff)

进入`preload`，可以看到Android预先加载了各种资源：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b96522647718452e9c415113e23ecee1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=753&h=686&s=118063&e=png&b=fffefe)

首先我们进入`preloadClasses`，可以看到`PRELOAD_CLASSES`这个常量:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36ffddddfd1d406b92f841860c3a1341~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=602&h=234&s=24381&e=png&b=ffffff)

他的值为：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/313a5e3b0cb9455db14fd041a205dfd5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=702&h=52&s=7673&e=png&b=fffefe)

从代码中，我们可以得知，会一行行的读取这个`preloadClasses`文件，并且将对应的类加载进来:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79a72780652344fb94bbcfdf48bbdd3f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=636&h=410&s=48875&e=png&b=fefcfc)

那么这个文件在哪呢? 这个文件其实在`/frameworks/base/config/preloaded-classes`，打开这个文件，就可以看到如下内容:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a32ae9276cd84df1bcee5b374655506b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=851&h=729&s=151507&e=png&b=fffefe)

这个文件有17000多行，这些就是我们要预先加载的类了。

再回到`preload`，进入`preloadResources`方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7ccfd86c20ec4401a83606beb5ab4ec3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=630&h=291&s=40764&e=png&b=fffefe)

可以看到这部分就是根据`R.array.preloaded_drawable`，去加载一些Drawable文件，这个array的定义是在`/frameworks/base/core/res/res/values/arrays.xml`下面：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/37e54574eacc4e85a0abac73d02c0ed0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=828&h=437&s=111865&e=png&b=fffefe)

可以看到注释里也提到了这是给zygote预加载资源用的。

至于`preload`里面的`nativePreloadAppProcessHALs`，`maybePreloadGraphicsDriver`，`preloadSharedLibraries`，`preloadTextResources`这些就不展开说了，有兴趣的可以查看源码。

至此，Zygote就完成资源的预加载。

# ZygoteInit启动Zygote Server并开始监听
从上面我们讲过，Zygote除了预加载这些虚拟机和资源文件，还需要做好准备，随时等待被`fork`成一个应用进程，那么Zygote是怎么做到的?

其实Zygote是通过LocalSocket来完成这一点的。

再次打开`/frameworks/base/core/java/com/android/internal/os/ZygoteInit.java`，并找到它的`main`方法，可以看到在方法的开头定义的`zygoteServer`:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/320810bf2f7b4cff9174554b49b885c4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=634&h=166&s=18038&e=png&b=fffefe)

然后下面可以看到`zygoteServer`的初始化:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b4c3c138597145ad9a9cb8a9ffeed659~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=613&h=70&s=5150&e=png&b=ffffff)

进入构造方法，我们就可以看到使用`LocalSocket`相关的代码了:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/400d1a4045884a498681fa067e39475f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=841&h=409&s=58888&e=png&b=ffffff)

打开我们模拟器的adb shell，输入:
```bash
netstat -nlp | grep LISTEN | grep zygote
```
就可以看到端口监听是对得上的:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2436a9f83498420dba24be416e6b64ff~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=920&h=86&s=22121&e=png&b=fffefe)

我们可以看到，怎么还有`usap`的监听，这是干什么的? 

其实这是Android新提供的一种新的应用启动方式，是为了加速我们的应用启动的，相关的介绍可以看这里:

> https://juejin.cn/post/6922704248195153927

在`main`方法后面的位置，我们可以看到`zygoteServer`的启动:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd38b74792f64c579d4e1fef8023b7b2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=732&h=92&s=10989&e=png&b=a6d2ff)

从注释我们也可以知道，到这，Zygote就已经进入等待的状态了，剩下的工作就交给`ZygoteServer`去处理了，至于`ZygoteServer`和`ZygoteConnection`是如何处理请求并`fork`处新的应用进程的，我后续会找机会再写一篇。

# 结尾
至此，我们已经梳理了整个Zygote初始化的过程，由于我只贴了关键代码，所以还是建议跟着AOSP源码去看这篇文章，才能理解整个的处理逻辑。

在Zygote的Init过程中，其实可以看到很多`SystemServer`相关的代码，这是因为Zygote最重要的需要`fork`进程之一，就是`SystemServer`了，下一篇我就会写`SystemServer`的启动流程分析。