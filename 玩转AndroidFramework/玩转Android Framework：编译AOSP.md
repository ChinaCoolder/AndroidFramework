---
theme: smartblue
---
本系列将基于Android 14详细讲解Android Framework的系列知识。

首先如果想要了解AOSP的编译，可以参照Google AOSP官网的介绍:
> https://source.android.com/docs/setup/start/requirements?hl=zh-cn

# 硬件要求

官方文档上说我们至少需要16gb内存来编译源码以及250g的硬盘空间来检出源码，**这里我实际编译Android 14的时候16gb内存是无法编译通过的，所以建议系统内存升级到32gb**。

这是我的硬件，实际编译过程大约在2到3个小时:  
硬盘: 2T  
CPU: Intel(R) Core(TM) i7-7700HQ CPU @ 2.80GHz  
内存: 32gb

# 系统要求

如果想要编译Android 源码，那么我们就需要使用Ubuntu系统，具体的系统要求及配置步骤可以参考这里:   
> https://source.android.com/docs/setup/start/initializing?hl=zh-cn

**为了少踩坑，建议准备一台安装Ubuntu系统的电脑或者安装Windows Ubuntu双系统，尽量不使用虚拟机或者WSL来进行编译。**

Ubuntu双系统安装，可以参考这里:
>  https://blog.csdn.net/Flag_ing/article/details/121908340
 
我安装的ubuntu版本是 18.04 lts，也可以选择安装22.04 lts或其它高于18.04的 lts版本。

**在安装ubuntu系统过程中，swap建议设置为实际内存的1倍，比如如果是32gb内存那么swap也设置为32gb。**

# 编译环境准备

当系统安装成功之后，首先要做的就是下载编译Android所需要的软件包:

```bash
sudo apt-get install git-core gnupg flex bison build-essential zip curl zlib1g-dev libc6-dev-i386 libncurses5 lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig
```

# 下载源码

安装成功之后，就要开始下载源码，AOSP使用repo进行版本管理，这实际上是一个基于git的python脚本，关于repo的介绍可以看这里:
> https://source.android.com/docs/setup/create/repo?hl=zh-cn

首先我们需要下载repo，这里推荐使用国内的清华镜像进行repo以及源码的下载：
> https://mirrors.tuna.tsinghua.edu.cn/help/AOSP/

##  第一步 下载repo文件:
```bash
mkdir ~/bin
PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

## 第二步 初始化工作目录:
```bash
mkdir Android14
cd Android14
```

## 第三步 初始化源码下载
**这里我们需要指定镜像地址和分支**，AOSP的分支可以在这里找到
  
> https://android.googlesource.com/platform/manifest.git/+refs 

由于我们是基于Android 14，所以在里面查找android 14的最新分支 android-14.0.0_r28:
```bash
repo init -u https://mirrors.tuna.tsinghua.edu.cn/git/AOSP/platform/manifest -b android-14.0.0_r28
```
## 第四步 下载源码
```bash
repo sync
```

Android源码的下载需要大约3到4个小时，这个时候只要耐心等待就好了，最后下载完成的目录大概是这个样子:


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9068566dd364b51a498ff17b3ab899a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=746&h=493&s=35561&e=png&b=f1f0ef)

# 编译源码

当我们完成Android 源码的下载，下一步就可以开始进行编译了

## 第一步 编译环境初始化:
```bash
. build/envsetup.sh
```

## 第二步 选择编译目标

```bash
lunch
```
然后控制台会打印出现有的编译目标:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0d278f299094585905a75fa4c290621~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1229&h=675&s=113053&e=png&b=2d0922)

接下来输入` sdk_phone_x86_64`，可以看到输出:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f2d8464d170142e0996aeeccc9d3c12f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=610&h=281&s=38965&e=png&b=2e0923)

## 第三步 进行编译

```bash
make
```

然后等待就可以了，编译时间一般在2到3个小时，等待编译成功之后，会出现如下字样:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f49e38858744ab8bb92294947d7dc54~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=855&h=376&s=54373&e=png&b=2e0923)

## 第四步 运行模拟器

然后输入以下命令，就可以直接看到我们编译过后的Android 14了:
```bash
emulator
```
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7406746fd484cda93e725ea51832526~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=486&h=894&s=355704&e=png&b=030016)

# 编译问题汇总

## 运行repo命令提示python格式错误

这个时候需要检查一下系统默认的python文件是否为python3，如果不是，需要将系统默认的python修改为python3
> https://blog.csdn.net/White_Idiot/article/details/78240298

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47535241a65e4903b032639faaa2761f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=741&h=61&s=9152&e=png&b=2d0922)

## repo init 或者repo sync报错
这个时候遇到的绝大多数都是网络错误，可以检查网络状态重新repo init 或者repo sync

## make报错 build failed
这个时候需要检查一下依赖包是否都安装上了，还有就是硬件要求是否达标，是否设置了系统swap

## emulator报错 This user doesn't have permissions to use KVM (/dev/kvm)
这个时候运行一下以下命令就可以了:
```bash
sudo chown 你的用户名 -R /dev/kvm
```