---
theme: smartblue
---

本文将介绍什么是Soong构建系统以及如何阅读及编写BP文件

# Soong构建系统介绍

对于Android App层开发来说，使用的是Gradle进行项目的构建，AOSP在Android 7.0之前，使用的是Make来进行AOSP的构建。

但是Android 7.0开始，Google开始使用Soong来进行AOSP的构建。
相比make文件，Soong使用的bp文件语法更加简洁，这是官方文档，里面有对Soong的介绍以及与make的对比:
>  https://source.android.com/docs/setup/build?hl=zh-cn

**Soong并不是终点，Google最终的目的是会将构建系统全面迁移到Bazel，这里是关于迁移到Bazel的计划:**
> https://source.android.com/docs/setup/build/bazel/introduction?hl=zh-cn

# Soong源码及构建流程
Soong是使用Go进行编写的，源代码在`/build/soong`，目录结构大致如下:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/22486d6164ce4c32ac855ff8b9f3e467~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=838&h=787&s=78329&e=png&b=f1efee)

**可以看到这里有一个README.md，里面详细介绍了关于Soong构建系统的一些知识，如果有兴趣可以看看。**

关于Soong的启动分析，可以参考这篇文章:
> https://zhuanlan.zhihu.com/p/342817768

# BP文件介绍
首先这是BP文件的大致结构:
```
gzip_srcs = ["src/test/minigzip.c"],
cc_binary {
    name: "gzip",
    srcs: ["src/test/minigzip.c"],
    shared_libs: ["libz"],
    stl: "none",
}
```
**BP文件不包含控制流语句和条件语句，只是单纯的配置，复杂逻辑全部由Go完成，但是可以声明并使用变量。**

首先以模块类型开头，比如上面的`cc_binary`，就是一个模块类型，其次必须指定一个`name`，而且按照官方文档的说法，这个`name`必须是唯一的，仅有两个例外情况是命名空间和预构建模块中的 Android.bp 属性值，其它的均为对模块类型的属性配置。

所有的模块类型和可配置属性可以参考这里:
> https://ci.android.com/builds/submitted/11519141/linux/latest/view/soong_build.html

比如`cc_binary`这个模块类型，可以在里面找到关于这个模块的详细解释和可配置属性都有哪些，以及对对应属性的解释:


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c3376d27c2746bb94cf191e30204017~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=765&h=756&s=191039&e=png&b=fffefe)

>[ cc_binary doc](https://storage.googleapis.com/android-build/builds/aosp-build-tools-linux-linux/11519141/a97bf94102ad7981c68b6ae6b6f96d4009eb8810dd848429634131f5ee6901b2/cc.html?GoogleAccessId=gcs-sign@android-builds-project.google.com.iam.gserviceaccount.com&Expires=1709388375&Signature=bEB7pvlfQQACyB4RzLoXExaM0kbRtEYn072kXmfEulzP0RrN37SCMkQoE5G47OcWts85M2tYA9o7fQzJzsVVXHIVw0WlZTdlSuD2/U%2BIt3%2BVgbHQyFFdnEt3ALqFeBEQoaPrLvebIL6YShSkMm/3Zfcru487sZpfYbo9TvVPePvhUvDYo9qaQpdRUknUZRgXhl8Wj0eGnQQuEuxwaoXNg6r2iCCPhljr/GlGrlpBtYcZEwwuuhz7RYaMtdM%2Bd9zYEbOaurAc3OlP4ZkXErBWJc7lKuAZvocFEYwNvnbIv6HgF%2BYpiBA4xYkTiO73fYy%2B%2B4z%2BS9dzLPwcnpkh2KWyww%3D%3D#cc_binary)

# Hello world
下面我们就要自己手动编写一个hello world c程序以及对应的bp文件，然后对其进行编译并运行。

## 第一步 创建工作目录
```bash
mkdir hello_world_test
cd hello_world_test
```
## 第二步 创建C文件
```bash
touch hello_world.c
```
下面是具体的代码:
```c
#include <utils/Log.h> //Android Log 库，用来打印输出到logcat，具体的实现在 /system/logging/liblog下面

int main(void) {
    int i;
    for(i = 0; i < 10; i ++) {
        ALOGD("JYC Hello World");
    }
}
```
## 第三步 创建BP文件
```bash
touch Android.bp
```
下面是BP文件的内容:
```
cc_binary {
    name: "jyc_hello_world", //定义模块名
    srcs: ["hello_world.c"], //指定源文件
    shared_libs: ["liblog"], //引用android log模块，具体的模块定义在 /system/logging/liblog/Android.bp 里面
}
```
## 第四步 编译hello world模块
记得要先进行编译环境的准备:
```bash
. build/envsetup.sh
lunch sdk_phone_x86_64
```
然后编译我们的hello world模块
```bash
make jyc_hello_world
```
最后可以看到编译成功:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aff92f8dbe734047accc1fba731ab89b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=720&h=117&s=21735&e=png&b=2e0923)

从图片中我们可以看出，最终生成的可执行文件在`out/target/product/emulator_x86_64/system/bin/`这个目录下，到这个目录我们可以看到我们的hello world已经编译成功了:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/34dd6dfbdf71489ca883ba8f72c08eb5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=524&h=600&s=40022&e=png&b=efeeed)

## 第五步 运行程序
首先我们打开模拟器:
```bash
emulator
```
然后新开个Terminal，将文件push到模拟器上面:
```bash
cd /out/target/product/emulator_x86_64/system/bin
adb push jyc_hello_world /data/local
```
然后进入adb shell，并且到指定目录下执行:
```bash
adb shell
cd /data/local
logcat -c
./jyc_hello_world
logcat | grep "jyc"
```
然后我们就可以看到运行结果了:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cf62792e46fa46e5a0299edc1f031f12~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=600&h=215&s=12989&e=png&b=2f0924)

# 其它

## 将Android make 文件转换成 bp文件
Soong提供了相关的工具，可以用来来将make文件转换成bp文件:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d70eae4191c4252b993a96dd10ab6e5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=651&h=91&s=17982&e=png&b=2e0923)
```bash
. build/envsetup.sh
lunch sdk_phone_x86_64
androidmk Android.mk > Android.bp
```
如果提示androidmk命令没有找到，那么可以先编译一下:
```bash
. build/envsetup.sh
lunch sdk_phone_x86_64
make androidmk
```
## 格式化Android bp文件
Soong还提供了一个工具来格式化bp文件:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd044607b60e4f8a9e975c8ac2971257~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=569&h=146&s=18304&e=png&b=2e0923)

```bash
. build/envsetup.sh
lunch sdk_phone_x86_64
bpfmt -w Android.bp
```