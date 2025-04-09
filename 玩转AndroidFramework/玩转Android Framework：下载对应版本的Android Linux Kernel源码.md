---
theme: smartblue
---

本文将介绍如何查找和下载对应版本的Android Linux Kernel代码。

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**

# 什么是Android Linux Kernel
众所周知，Android是基于Linux的，那么Android Linux Kernel，实际就是指优化过的针对Android的一种特殊版本的Linux，这是官方的文档，里面详细介绍了关于Android Linux Kernel的一些知识:

> https://source.android.com/docs/core/architecture/kernel?hl=zh-cn

# 为什么我们需要下载Android Linux Kernel源码
当我们打开AOSP源码的`/kernel`目录下的时候，我们可以看到AOSP并不包含对应使用的Linux源码，只有预先编译好的各个版本的Linux，这意味着如果我们替换AOSP编译时使用的Linux Kernel版本，或者希望修改Linux Kernel代码在设备在启动Android之前做一些工作的话，我们就必须下载对应版本的Linux源码进行修改和编译。

# 确认使用的Android Linux Kernel版本
下面我们就要想办找到我们设备所使用的Android Linux Kernel版本，首先打开模拟器，进入`Setting->About->Android Version`，我们就可以看到我们的Linux Kernel版本了:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/17fa18ad1f374468ba6901d004aba67f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=405&h=785&s=57702&e=png&b=f0eff6)

我们现在可以看到，我们的`Kernel version`是

`6.1.23-android14-4-00257-g7e35917775b8-ab9964412 #1 Mon Apr 17 20:50:58 UTC 2023`。

# 下载源码
这里是Android Linux Kernel的git仓库:

> https://android.googlesource.com/kernel/common/

如果想了解关于Kernel Common中这些分支都代表着什么，可以在这里看到:

> https://source.android.com/docs/core/architecture/kernel/android-common?hl=zh-cn

打开之后我们可以看到如下界面:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07d5900dd15b49229371a49b6eb4ec40~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1047&h=793&s=195630&e=png&b=fdfdfd)

然后我们下载代码:

```bash
git clone https://android.googlesource.com/kernel/common
```
下载完成之后，我们就能看到如下的文件结构:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/576a8ac3c63247dfa431696928b44c89~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=869&h=798&s=66227&e=png&b=fdfcfc)

# 切换到指定commit
现在我们已经有了内核源码，下面我们就要切换指定的提交了。

通过上面的`Kernel version`g开头后面的那一部分，就可以找到我们的短commit id，也就是`7e35917775b8`，直接checkout:

```bash
git checkout 7e35917775b8
```

等待完成之后，目前的代码就是我们设备所真正使用的Android Linux Kernel源码了。

关于Android Linux Kernel的编译以及如何在emulator和cuttlefish上使用指定编译好的Linux Kernel，后面我会找机会再写一篇。