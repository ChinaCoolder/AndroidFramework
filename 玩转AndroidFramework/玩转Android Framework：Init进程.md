---
theme: smartblue
---
本文将介绍有关Android Init进程相关的一些知识。

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**

# 什么是Init进程
当我们在adb shel中输入`ps -A`列出所有进程时，我们可以看到有一个名字为`init`，pid为1的进程，

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/78b7f63da83b4014a94858707f1e24c0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=940&h=606&s=82102&e=png&b=300a25)

再往下看，可以看到很多ppid为1的进程，这意味着这些进程都是从`init`进程中`fork`出来的，包括我们最熟悉的zygote进程:


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4e8c39abf5a04bc8ba0e4fbe20260abf~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=816&h=267&s=53053&e=png&b=300a25)

可以说，`init`进程是我们整个Android世界的起始点，那么了解它的作用，就变得十分重要了。

# 谁启动了Init进程
在Android设备启动时，第一个步骤便是`bootloader`，这部分Android官网有相关的介绍:

> https://source.android.com/docs/core/architecture/bootloader?hl=zh-cn

之后，就会开始启动我们的Linux Kernel，当Linux Kernel启动完成，就会启动我们的Init进程了，关于这部分，可以在Android Linux Kernel源码中，查看`/init/main.c`的`rest_init`方法，这里即是Linux Kernel启动我们的Init进程的地方:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/302c6412dd264f158df0645145d0e609~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=697&h=719&s=129756&e=png&b=fffefe)

**如果不知道如何查看Android Linux Kernel源码的，可以看这篇文章：**

> https://juejin.cn/post/7343509431714742313

# Init进程做了什么
Init相关的代码主要集中在`/system/core/init`，入口方法是这个目录下的`main.cpp`的`main`方法，打开之后，我们可以看到如下代码，这就是init进程的初始化逻辑的主干部分:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21864834b938495cbb2dee64ddaf18e8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=714&h=616&s=99608&e=png&b=fffefe)

# Init进程执行顺序
从上面的代码我们可以看到，我们的Init貌似有好几个步骤，根据参数的不同，要执行不同的逻辑，那么这些步骤都是干什么的?

其实我们可以打开`/system/core/init/README.md`，可以看到Google已经为我们做了大致的解释:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/87973fbf820649f68bf73cc194f62f18~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1308&h=608&s=138958&e=png&b=fefefe)

从这里我们可以知道，这个`main`方法会被传入不同的参数反复执行，执行的顺序大概是first stage, selinux stage和second stage。

# 解析RC文件并启动服务

first stage从文档中可以看到，工作和逻辑较为繁杂，对这部分有兴趣的可以查看`/system/core/init/first_stage_init.cpp`，这里不做过多赘述:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f9f3c63dab648d4b749e7aa184fc1d2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=847&h=526&s=111006&e=png&b=fffefe)

selinux stage，则是为Android加载selinux，关于Android对SELinux的支持，可以看这里:

> https://source.android.com/docs/security/features/selinux?hl=zh-cn

最后，就是second stage了，文档里面也提到了，在second stage，我们就会开始解析rc文件并根据rc文件启动一系列进程，打开`/system/core/init/init.cpp`，就可以看到我们的second stage的主要处理方法了:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef69d0c73b464c99b81cce9cf68c540c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=933&h=687&s=140504&e=png&b=fffefe)

在这个方法中查找，我们可以看到`LoadBootScripts`方法被调用，进入此方法，我们就可以看到，在这里，Android完成了对rc文件的读取以及解析工作:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d020142642444379bf80fbb4c553be9f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=741&h=317&s=68868&e=png&b=fffefe)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eb414fbc94ba44cdb949ebb7f63a1983~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=774&h=482&s=96249&e=png&b=fffefe)

最后，就是开始启动rc文件中的`early-init`和`init`了：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/466d4d221892443791ebf917fa591dc3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=860&h=462&s=108651&e=png&b=fefefe)

下面是`/system/core/rootdir/init.rc`文件中`early-init`和`init`两个trigger的定义：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c92002b507614c7f9530be2c3708c869~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=751&h=461&s=84367&e=png&b=fffefe)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b396efcfc564a1193ffcbaf5c9cc146~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=619&h=535&s=93574&e=png&b=fffefe)

**如果不知道如何阅读rc文件，可以看这里:**

> https://juejin.cn/post/7341668771910811675

至此，可以说Android已经开始基于rc文件要启动起来了。