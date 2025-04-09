---
theme: smartblue
---

本文将介绍Android Init Language 相关的一些知识。

# 什么是Android Init Language
Android Init Language 是Android用来进行服务初始化的一种脚本语言，这些脚本的后缀名均为.rc，这种脚本主要的作用是用来顺序启动Android的一系列服务

这是Android Init Language的官方文档:

> https://android.googlesource.com/platform/system/core/+/master/init/README.md

# 语法结构
Android Init Language 语法结构十分简单，大致如下:
```
import <path>

service <name> <pathname> [ <argument> ]*
   <option>
   <option>
   ...

on <trigger> [&& <trigger>]*
   <command>
   <command>
   <command>
```
一个rc文件，由5个部分组成，Actions，Commands,，Services，Options，和 Imports。

# 阅读RC文件
接下来我们将讲解如何阅读一个rc文件，首先在`/system/core/rootdir`下面，可以看到多个rc文件:


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/45ffd1d106634bbcbc2761fbf080014f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=822&h=456&s=29913&e=png&b=f1f0ef)

打开`init.zygote64.rc`，我们可以看到以下的文件内容:
```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    # NOTE: If the wakelock name here is changed, then also
    # update it in SystemSuspend.cpp
    onrestart write /sys/power/wake_lock zygote_kwl
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart media.tuner
    onrestart restart netd
    onrestart restart wificond
    task_profiles ProcessCapacityHigh MaxPerformance
    critical window=${zygote.critical_window.minute:-off} target=zygote-fatal
```
这个时候我们可以打开Android Init Language 的Readme文件，来一项项查找这个文件的每条指令代表什么。

## `service zygote  /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote`

从文档中我们可以看出，这代表着注册一个名字为zygote的service，并且指定此服务的执行文件路径，以及执行此文件时所带的参数:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fee9aa78aec74a98bcfa173a63812f0d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=744&h=216&s=13840&e=png&b=fcfcfc)

## `class main`
这是服务的类名，相当于这个服务的分组，默认为`default`，这些分组的服务可以整组的启动或停止:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3956d0a0f84a44ada222a8f9a53f1e18~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1012&h=171&s=39734&e=png&b=fef7ed)

## `priority -20`
服务的优先级，取值范围为-20到19，默认为0:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b66f3d12c94a47d8b237afa0a90f71a5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=886&h=103&s=10409&e=png&b=fff9f5)

## `user root`
执行此服务的之前切换的用户，默认为`root`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac2dfa338a1349aa8df6cebdeeb6eb7d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=990&h=213&s=53896&e=png&b=fff9ec)

## `group root readproc reserved_disk`
执行此服务前更改组，从第二个参数开始，都是此进程的补充组:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/acda90d7f6e64f95940a4d6abfb210fc~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=980&h=113&s=20701&e=png&b=fef7f1)

## `socket zygote stream 660 root system`
创建一个domain socket，第一个参数是此socket的名字，第二个是类型dgram, stream 或者seqpacket其中一个，660为权限设置，后面的为用户以及组:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c11499c99ee42fe974100e7dc0dac25~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=986&h=171&s=42868&e=png&b=fef8ec)

## `onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse`
服务重启时要执行的命令:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/14e6966fa7a44882a53c68a302d05c2d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=987&h=86&s=6332&e=png&b=fff9f3)

## `task_profiles ProcessCapacityHigh MaxPerformance`
设置任务配置文件:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ad250fbb265493ca387e984403847ac~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=986&h=119&s=18250&e=png&b=fef7f1)

## `critical window=${zygote.critical_window.minute:-off} target=zygote-fatal`
指定此服务为关键服务:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af3b801170ad426a9e3375f16029b84a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=996&h=145&s=34013&e=png&b=fdf6f0)

至此，我们就对照文档了解了整个zygote rc文件的指令，后续如果遇到其它的rc文件，也可以照此办法进行解读。

# Android Init Language 源码
Andorid解析rc文件的源码，主要在`/system/core/init`下面:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a409d74e174d4b2683e67439c158dfb1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=819&h=694&s=76258&e=png&b=f0efee)

比如如果你想了解Android是如何通过解析rc文件创建并启动service的，可以查阅`service.cpp`，`service_list.cpp`，`service_parser.cpp`，`service_utils.cpp`，这些相关文件。