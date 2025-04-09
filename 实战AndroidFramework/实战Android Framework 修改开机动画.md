---
theme: smartblue
---

本文将介绍如何修改Android的开机动画，基于AOSP分支android-14.0.0_r28。

# 源码分析
Android绘制开机动画相关的代码，主要集中在`frameworks/base/cmds/bootanimation/`中，打开我们可以看到如下的目录结构:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d04cd63acc144d4d8d9220d646952c31~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=655&h=397&s=20855&e=png&b=f1f0ef)

至于开机动画的实际绘制，具体在`BootAnimation.cpp`中，其它的代码无需关心，重点要看从815行开始的代码:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bd627977a11c4266ad4215ea55baaa41~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=453&h=484&s=64870&e=png&b=fffefe)

从这里我们可以看到，这里面做了一个判断，如果没有对应的zip file，那么会执行`android()`，如果没有，那么会执行`movie()`。

**实际上，当我们查看两个方法的时候可以看到无论我们使用哪种方法，最后实际上都是使用opengl来绘制的，只是movie会读取对应zip文件中的图片来进行绘制。**

那么这里我们可以得出一个结论，我们有两种方法来修改Android的开机动画，**第一种，就是我们提供给Android所需的zip文件，第二种，就是我们直接使用opengl来手绘我们的开机动画**。

# BootAnimation Zip 文件方式修改开机动画
首先介绍最常用的一种方法，提供开机动画Zip文件。
## Where
从上面的源码分析，我们现在第一步要做的，就是要找到这个`mZipFileName`是怎么赋值的。

实际上Android关于这的处理也比较简单，主要是这两个方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07e1ddfa541c436d9dac0e19198f4019~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=882&h=635&s=114329&e=png&b=fffefe)

从代码逻辑上我们可以看到，如果是`sys.init.userspace_reboot.in_progress`这个值为true，那么就会使用`userspaceRebootFiles`，如果当前是关机的话，那么会使用`shutdownFiles`，最后才会使用`bootFiles`。

从名字上我们可以得知，第一种是在Android软启动的时候，才会使用的zipfile，这是关于Android软启动的介绍:

> https://source.android.com/docs/core/runtime/soft-restart?hl=zh-cn

第二种则是Android关机的时候，才会使用的zipFile，第三种则是平常开机时候，使用的zipFile。

软启动时候，所寻找的zip文件位置如下:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c18f16a334084a67848458d8bebcd0e3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=895&h=75&s=20007&e=png&b=fffefe)

关机的时候，所寻找的zip文件位置如下:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/66b77cc27b2349088d5082ab3543f9c2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=762&h=88&s=18242&e=png&b=fefcfc)

平常开机的时候，所寻找的zip文件位置如下:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b5aea6d3a5ee4627b5b956ee1d210ca7~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=848&h=134&s=28117&e=png&b=fefcfc)

**注意，从代码逻辑上可以看出，这些文件是有优先级的，比如如果同时存在`/system/media/bootanimation.zip`和`/product/media/bootanimation.zip`，那么会优先使用`/product/media/bootanimation.zip`。**

**关于上面的这个`playDarkAnim`，我找了半天也没找到`ro.boot.theme`这个值是在哪设置的，看了下git hisotry，这个代码19年就在了，有知道的可以留言讨论一下。**

## How

现在已经知道我们该把文件输出到哪里了，接下来我们就要找到，这个文件是个什么格式。

关于Bootanimation Zip文件的格式，可以查看`/frameworks/base/cmds/bootanimation/FORMAT.md`这个文件，里面详细介绍了该zip文件的格式以及我们要准备的内容。

打开文件，我们可以看到下面的内容:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edf04315856843b4bfeff870cfcf4264~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=563&h=262&s=18183&e=png&b=f9f9f9)

那么从这里面我们可以知道，zip文件中的第一层结构，就是一个`desc.txt`和一堆以`part0`开头的文件夹，里面要放我们的动画文件，也就是png格式的图片。

首先，我们来看一下这个`desc.txt`的详细描述:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/04c5f149200047a6b7fbfb06deba6d92~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=710&h=887&s=161282&e=png&b=fcfcfc)

文件很长，而且有有些配置是可选的，我们先写一个最简单的:

```
320 480 24
p 1 2 android
p 0 0 part1
```

**注意，这里我得目录名没有以part0开头，因为实际上在解析zip文件的时候，是先解析desc.txt，然后再去寻找对应的目录。**

然后这是我的zip第一层目录结构，和里面的png frame:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/559761fd8fe1459794fc663828fa1e8c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=881&h=271&s=27417&e=png&b=fefefe)
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec3d8fe317c64a1dbe7792922fff00e2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=873&h=370&s=74607&e=png&b=fffefe)

## Do It
现在，我们准备好了我们的zip文件，就要想办法编译到我们的android系统里面了。

首先，将`bootanimation.zip`复制到`/device/generic/goldfish/data/media`下面，接着，我们修改`/device/generic/goldfish/x86_64-vendor.mk`文件，从28行开始修改成如下指令:
```make
PRODUCT_COPY_FILES += \
    device/generic/goldfish/data/media/test/swirl_136x144_mpeg4.mp4:data/media/0/test/CtsMediaTestCases-1.4/swirl_136x144_mpeg4.mp4 \
    device/generic/goldfish/data/media/test/swirl_132x130_mpeg4.mp4:data/media/0/test/CtsMediaTestCases-1.4/swirl_132x130_mpeg4.mp4 \
    device/generic/goldfish/data/media/test/swirl_130x132_mpeg4.mp4:data/media/0/test/CtsMediaTestCases-1.4/swirl_130x132_mpeg4.mp4 \
    device/generic/goldfish/data/media/test/swirl_144x136_mpeg4.mp4:data/media/0/test/CtsMediaTestCases-1.4/swirl_144x136_mpeg4.mp4 \
    device/generic/goldfish/data/media/test/swirl_128x128_mpeg4.mp4:data/media/0/test/CtsMediaTestCases-1.4/swirl_128x128_mpeg4.mp4\
    device/generic/goldfish/data/media/bootanimation.zip:product/media/bootanimation.zip
```
可以看到，最后一行会在编译时，将我们的zip文件复制到`product/media`下面，从上面的代码分析我们可以知道，在这里提供`bootanimation.zip`也是可行的。

这样就完成了，接下来，我们重新编译项目:
```bash
. build/envsetup.sh
lunch sdk_phone_x86_64
make
```
编译成功之后，如果在`/out/target/product/emulator_x86_64/product/media`下面看到我们的`bootanimation.zip`，那么就说明我们成功了:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/892d4fd1bd8d4ccbbdcf43cecbff2a05~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=431&h=251&s=12135&e=png&b=f2f1ef)

之后运行模拟器:
```bash
emulator
```

最终我们就会看到开机动画已经被修改了:

![屏幕截图_1.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9afad83ba1884517b872acfab8aa2648~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=416&h=868&s=25692&e=png&b=010101)

# Opengl方式，修改开机动画
进入上面说过的`android()`方法，我们可以看到android使用opengl库进行绘制:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/300e0f9368644e9488ea54944f04f4d1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=801&h=866&s=96451&e=png&b=ffffff)

如果需要通过OpenGL方式修改开机动画，可以不提供zip文件，直接在这里进行修改。

**注意，可以在源代码里面看到，android()方法绘制开机动画时，android官方特意将其限制为12fps，这是为了不要绘制的太快，防止影响CPU处理其它重要工作**:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7383b9b40aad4c3ba74df92f78be8e7a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=546&h=172&s=15401&e=png&b=fffefe)

# 其它
## bootanimation.zip增加音效
在描述zip文件的`FORMAT.md`，提到我们可以为每个动画的part提供音效文件，这些音效会在开机时播放。

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77cfc860a54e4442afb6cbefc70cf55a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=647&h=128&s=14987&e=png&b=fdfdfd)
## 压缩PNG帧
官方建议我们将每一帧png图片尽量压缩，并且提到了可以使用如下工具:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69595d6048c34ac0bbf8baae3d4eca2e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=803&h=360&s=38237&e=png&b=f9f9f9)