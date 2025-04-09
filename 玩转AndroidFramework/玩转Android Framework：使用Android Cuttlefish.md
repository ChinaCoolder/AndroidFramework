---
theme: smartblue
---
本文将介绍Android Cuttlefish的安装和使用。

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**

# 什么是Android Cuttlefish?
简单的来说，Android Cuttlefish是一种可以托管在云端的Android模拟器。

这是Android Cuttlefish的官方文档:

>https://source.android.com/docs/setup/create/cuttlefish?hl=zh-cn

# 为什么需要Android Cuttlefish
在实际开发中，我们的代码实际编译过程可能是发生在远程的服务器，那么当编译完成的时候，如果我们想看到我们的编译之后的系统的实际运行效果，可能就需要将编译好的image下载到本地刷进实机或模拟器，才能查看效果。

那么问题来了，我们可不可以把模拟器运行在云端，直接通过浏览器就能看到我们的编译好的系统的运行效果呢？

Android Cuttlefish就是在这个基础上诞生的。

# 安装Android Cuttlefish
下面我们就来介绍怎样安装Android Cuttlefish。

## 第一步 检查虚拟化支持
运行
```bash
grep -c -w "vmx\|svm" /proc/cpuinfo
```
如果返回值大于0，那么进行下一步。

## 第二步 安装依赖

```bash
sudo apt install -y git devscripts config-package-dev debhelper-compat curl
```

## 第三步 下载源码

Cuttlefish的源码在:
> https://github.com/google/android-cuttlefish

在我们的工作目录运行:
```bash
git clone https://github.com/google/android-cuttlefish.git
```
下载完成之后进入cuttlefish目录:
```bash
cd android-cuttlefish
```

## 第四步 确认Go语言版本支持
确定系统Go语言版本:
```bash
go version
```

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c32e51c850884a908ebc8af202722724~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=339&h=96&s=9422&e=png&b=300a25)

如果没有安装Go或者版本小于1.15，那么请安装或升级到最新版的Go:

> https://go.dev/doc/install

## 第五步 修改脚本
由于国内访问不了国外的Go库，所以我们需要设置代理，打开`/android-cuttlefish/frontend/src/goutil`，删除`export GOPROXY="proxy.golang.org|proxy.golang.org|direct"`这一行:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a427a245f3224a7486cfe2f3c487edb9~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=633&h=621&s=59936&e=png&b=ffffff)

增加如下两行代码:
```bash
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.cn,direct
```

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/310777f25fa84bb18828e1daa82450c9~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=752&h=603&s=74310&e=png&b=ffffff)

## 第六步 编译deb包
在`android-cuttlefish`目录运行:
```bash
for dir in base frontend; do
  cd $dir
  debuild -i -us -uc -b -d
  cd ..
done
```
等待一段时间，如果编译成功，那么在根目录应该就可以看到以下的这些deb文件:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0073e963028343288aa99b27d1b163ce~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=584&h=156&s=26493&e=png&b=310a25)

关于这些deb包有什么区别，github上有详细的解释:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0cc2b7f28abd4b89a6453d35f7f00917~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=862&h=222&s=52004&e=png&b=fffefe)

实际上我们只需要`base`和`user`就可以了。

## 第七步 安装deb包
```bash
sudo apt install ./cuttlefish-base_*.deb ./cuttlefish-user_*.deb
```
安装成功以后，就可以开始编译我们的AOSP并运行Android Cuttlefish了。

# 编译AOSP并运行Android Cuttlefish
```bash
. build/envsetup.sh
lunch aosp_cf_x86_64_phone-userdebug
make
```
注意，这里我们的`lunch`目标是`aosp_cf_x86_64_phone-userdebug`。

等待编译完成之后执行:
```bash
launch_cvd
```
如果运行报kvm相关的错误，运行以下指令:
```bash
sudo usermod -aG kvm,cvdnetwork,render 你的用户名
```
之后重启:
```bash
sudo reboot
```
重启完成之后再执行:
```bash
. build/envsetup.sh
lunch aosp_cf_x86_64_phone-userdebug
launch_cvd
```
Android Cuttlefish运行成功之后，浏览器访问`https://localhost:8443/`，无视浏览器不安全提醒，然后就能看到如下画面:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/432b9b121f584e05899ae31ceb8e3e5d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1008&h=615&s=13754&e=png&b=d4d4d4)

然后启用我们的设备，就可以看到我们的模拟器了:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/da9d645a3dcd4d6299efbb80458d0eb9~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=785&h=870&s=96895&e=png&b=dddddd)