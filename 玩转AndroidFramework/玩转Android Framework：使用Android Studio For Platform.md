---
theme: smartblue
---
本文将介绍如何安装并使用Android Studio For Platform

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**

# 什么是Android Studio For Platform
Android Studio For Platform，是一种用来方便我们进行AOSP开发的特殊版本Android Studio，他与普通的Android Studio其实没有太大差别，只是增加了对AOSP开发相关的一些支持。

# 为什么要使用Android Studio For Platform
在原来，我们调试自己开发的源码的主要手段，可能是通过生成ipr文件来导入到Android Studio中，然后再进行系统源码的调试，但是这样做很麻烦，所以Google为AOSP开发者提供了专门的开发工具来帮助我们的开发工作。

# 下载并安装Android Studio For Platform
这是Android Studio For Platform的官网:

> https://developer.android.com/studio/platform?hl=en

下载完成之后，安装对应的Deb文件:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08ef3b1d4d0643a39a786cf3fd03b56b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=281&h=385&s=8615&e=png&b=fafafa)

安装完成之后，运行:
```bash
cd /opt/android-studio-for-platform/bin
./studio.sh
```
然后就可以看到:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6cc6b03929d4691997433087c15800b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=829&h=650&s=39756&e=png&b=222326)

# 导入AOSP
首先我们点击:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6669c4e10df74180a2e7c5abc371033a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=156&h=160&s=2801&e=png&b=202124)

进入如下界面:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4c744b435f56434fbf3fb0f5decb7539~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=900&h=681&s=30876&e=png&b=2b2d30)

`Repo checkout`，选择你的AOSP源码根目录。

`Lunch target`，选择你的lunch目标，比如模拟器的话就填写`sdk_phone_x86_64`。

`Module Path`，选择你要调试的模块，比如调试frameworks模块的话，就添加选择frameworks。

`Project name`，填写你的项目名。

`Location`，这是你的Android Studio For Platform存储项目信息的路径。

最后如下：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b1b820d8dbe4e5b920664d96a198013~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=905&h=681&s=41823&e=png&b=2b2d30)

点击确定，进入Android Studio For Platform。

**注意，首次进入时需要做建立索引等一些工作，所以会比较慢，耐心等待即可。**


# 调试AOSP
下面我们将介绍如何调试我们的AOSP代码，在调试之前，我们要先将我们的模拟器运行起来:
```bash
. build/envsetup.sh
lunch skd_phone_x86_64
make
emulator
```
运行起来之后，就可以开始下面的工作。

## Attach Debugger
点击右上角的Attach Debugger To Android Process:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b1a037025db44a086416a4005c8d0f1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=42&h=46&s=800&e=png&b=27282e)

勾选`Show all processes`，然后选择`Java Only`，然后选择我们要attach的进程，比如我们的Setting:


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f205a726cb984e808933259537b16b26~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=473&h=497&s=30918&e=png&b=f5f7fa)

当出现如下字样，说明我们的debugger已经attach完成了:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07b73ecfab7449948db8385599953edb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=665&h=234&s=13516&e=png&b=ffffff)

## 设置断点
找到我们需要打断点的类，比如我们最熟悉的`/frameworks/base/core/java/android/app/Activity.java`，找到我们的`onResume方法`，设置断点:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5bbf7f22dbd946ec959528e74a882252~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=714&h=288&s=37033&e=png&b=fffdfd)

接下来打开我们模拟器的Setting应用，就可以看到我们已经进入我们的断点了:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb85decc6f4d4e2c8a68440cc34869b1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=744&h=303&s=37767&e=png&b=ffffff)
