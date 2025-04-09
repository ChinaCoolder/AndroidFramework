---
theme: smartblue
---

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**

相信做过Android开发的，一定对Binder这个词不陌生，我看过很多关于Binder的面试题和面试答案，总感觉是在隔靴搔痒，死记硬背一些概念，所以本文打算系统的讲解一下，到底什么是Binder。

# 什么是Binder
Binder，其实本质上来讲就是Android为我们提供的一种基于Binder驱动的跨进程通讯方式，如果说要问Android区别于Linux最大的特色是什么，那我可能会说Binder。

那么问题来了，为什么我们需要Binder?

# 为什么需要Binder
Linux本身有很多跨进程通讯手段，比如Socket，比如管道，为什么Android要多提供Binder这样一种新的跨进程通讯方式呢? 这个问题其实由`Brian Swetland`回答过了，他是2004年到2012年期间Android系统及内核的技术负责人，而他在邮件中是这么说的:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea31c81fb5c24904a5332e359cce92d6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=872&h=714&s=146058&e=png&b=cdd8d8)

Android使用Binder的核心理由，其实就两条:
```
- avoiding copies by having the kernel copy from the writer into a
ring buffer in the reader's address space directly (allocating space
if necessary)
- managing the lifespan of proxied remoted userspace objects that can
be shared and passed between processes (upon which the userspace
binder library builds its remote reference counting model)
```

第一条，其实是说的Binder在驱动层使用了Linux提供的mmap内存映射机制，第二条，应该是指的Binder驱动层对"对象"的生命周期的管理，总之，Android的技术负责人认为这种新的IPC更高效和便捷。

好了，现在我们知道了Android为什么要使用Binder了，那我们应该看看，Binder到底是怎么工作的。

# Binder驱动
首先，我们来看看Binder驱动，Binder驱动层的代码，并不在AOSP中，而是在Android Linux Kernel中，如何下载Android Linux Kernel源码，可以看这里:

> https://juejin.cn/post/7343509431714742313

下载好之后，查看`drivers/android/`，在此目录下可以看到如下文件:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9678dbf46577453ebcc8c82efefa5ed2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=590&h=230&s=13840&e=png&b=fdfdfd)

打开`binder.c`，这里，就是驱动层的核心代码所在了:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0e1594d152174f48a81f707639dc6637~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=771&h=968&s=142288&e=png&b=fffefe)

下面，我将详细介绍整个Binder的工作流程。

# Binder的工作流程
## 第一步 注册Binder设备
首先，我们已经知道了我们的Binder通信是基于Binder驱动来进行的，那么最开始的工作，就是Binder驱动层的初始化，打开`drivers/android/binder.c`，找到`binder_init`方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d422f58fd9764feb816ee8fd83995dc2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=617&h=608&s=79620&e=png&b=fffefe)

这里，便是我们的Binder驱动初始化方法，其它的我们暂时不关心，主要看调用的`init_binder_device`方法，这里面最核心的，就是调用了`misc_register`去注册了我们的Binder驱动:


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/163f9581fe5646b392de9574bd58a8ee~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=647&h=516&s=67729&e=png&b=ffffff)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/306e0a5d175a4f98a2f9b20e69b215a0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=613&h=547&s=76045&e=png&b=fffefe)

下面是我们的`binder_device`结构体:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dda4720551814fad89631f356444f486~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=699&h=324&s=46412&e=png&b=fffefe)

`hlist`: 这个变量主要是用来存储`binder_device`的全局链表，目前`binder_device`，会有三种，分别是`/dev/binder`，`/dev/hwbinder`和`/dev/vndbinder`，他们的区别可以看这里
> https://www.jianshu.com/p/a79ccbf84f06

`miscdevice`: 这是杂项设备信息，如果不了解什么是杂项设备，可以看这里: 

> https://developer.aliyun.com/article/842351

`context`: 这个变量存储的即是一个特殊的`binder_node`，即`ServiceManager`，后面会讲到。

这是我们的Binder驱动设备文件操作方法定义：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9c63537c60ef425ba8b1ff1778d042c8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=399&h=201&s=20768&e=png&b=fefdfd)

如果不了解`misc_register`是怎么注册设备文件方法的，可以看这里：

> https://www.cnblogs.com/snowdrop/articles/8678389.html

简单来说，当我们注册好设备文件方法，当Native层调用`mmap`的时候，最终会调用到`binder_mmap`这个方法中来。

## 第二步 启动ServiceManager
什么是`ServiceManager`呢? `ServiceManager`你可以理解为它是一个Binder驱动层的DNS服务，当一个客户端想要与服务端进行通讯的时候，只需要通过一个"网址"，就可以通过`ServiceManager`找到真正的服务端，它可以说是理解Binder通信的核心概念之一，下面我们来看看，它是怎么被启动的。

首先打开`AOSP/system/core/rootdir/init.rc`，找到如下三行:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1339a6039914dd99585e003f45766ba~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=407&h=108&s=11813&e=png&b=ffffff)

我们可以看到，它也是被Init进程通过解析RC文件被启动的，我们再打开`AOSP/frameworks/native/cmds/servicemanager/main.cpp`，找到`main`方法，这里，便是`ServiceManager`的实际启动入口:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2607742ceb9d4b25a49cbfbfee673779~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=780&h=802&s=92171&e=png)

我们可以看到，整个方法并不复杂，但是我首先要简单介绍两个类的概念，以方便我们对Native层Binder通信的理解。

### ProcessState
打开`AOSP/frameworks/native/libs/binder/ProcessState.cpp`，我们就可以看见`ProcessState`的具体实现。

这个类的主要作用是控制Binder驱动，比如打开，关闭。

每个**进程**，都会只有一个`ProcessState`实例，如果有兴趣的可以仔细看看它的`init`方法，看看它是怎么保持每个进程只有单个实例的。

### IPCThreadState
打开`AOSP/frameworks/native/libs/binder/IPCThreadState.cpp`，我们就可以看到`IPCThreadState`的具体实现。

这个类的主要作用，就是实际的通过Binder驱动发送和接收数据了。

每个**线程**，都会只有一个`IPCThreadState`实例。

好，接下来我们重新回到`ServiceManager`的初始化部分，首先我们来看这段代码:
```C++
sp<ProcessState> ps = ProcessState::initWithDriver(driver);
```
最先开始，是进行了`ProcessState`的初始化，进入此方法,可以发现调用的是`init`方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0c7b801bb4f845c9a8e83f442f5e15cb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=649&h=106&s=12343&e=png&b=fffefe)

进入`init`方法，可以看到最终调用了`ProcessState`的构造方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ab04121db7c42a18406ae14ed78c18e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=774&h=865&s=114565&e=png&b=fffefe)

接下来再进入构造方法，我们可以看到如下内容:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/280bc22943d94f9d82d716534642142c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=884&h=869&s=105439&e=png&b=fffefe)

这里，我们就可以看到，最开始，调用了`open_driver`方法，这个方法，就是用来打开Binder驱动的，我们进入此方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a63c5c0d6dc8497f825565127da42664~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=763&h=724&s=99185&e=png&b=fffefe)

前面知道，我们已经注册了Binder驱动层设备文件方法了，所以最开始的open，会进入Binder驱动的`binder_open`方法，我们先看看驱动层在这个方法做了什么:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3d20d244de0419d834d7947f1561de4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=609&h=732&s=116650&e=png&b=fefdfd)

其实`binder_open`方法，最主要的作用就是在Binder驱动层创建并初始化了一个`binder_proc`，接下来我再介绍下，什么是`binder_proc`。

### binder_proc
在`AOSPLinuxKernel/drivers/android/binder_internal.h`中，可以找到这样一个结构体:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ea52032a76949769c999ddb038f9f44~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=484&h=680&s=67005&e=png&b=fffefe)'

这个结构体的主要作用，就是存储我们的进程信息，当Native层每次打开Binder驱动的时候，Binder驱动层都会创建一个`binder_proc`，可以说每一个Native层的`ProcessState`，在Binder进程中，都会有一个对应的`binder_proc`。

好，接下来我们再回到Native层`ProcessState`的`open_driver`，可以看到调用`open`之后，获取了Binder驱动的fd，下面就是执行这样一条:
```C++
status_t result = ioctl(fd, BINDER_VERSION, &vers);
```
这里，又会再次回到Binder驱动层的`binder_ioctl`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ba3db6fecad47ffb47ed8e3af998fec~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=728&h=408&s=54213&e=png&b=ffffff)

这个方法，主要是执行我们的一些控制命令，从上面我们可以知道，我们传进去的命令是`BINDER_VERSION`，那么就会执行到下面这个部分：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ee8998613f2408ead482ad090166391~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=487&h=269&s=25118&e=png&b=ffffff)

这里主要是获取内核Binder版本，然后与用户层的Binder版本进行对比，防止错误。

接下来我们再回到`open_driver`，可以看到接下来执行到了这一步:
```C++
result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
```
同样的，我们再回到驱动层，可以看到执行的是这里:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d055147e1e4e405b90c878b664c786ef~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=395&h=249&s=23017&e=png&b=fffefe)

他的主要作用，是设置了驱动层当前进程的对应`binder_proc`的线程上限。

再回到`open_driver`，接下来到了这一步：
```C++
result = ioctl(fd, BINDER_ENABLE_ONEWAY_SPAM_DETECTION, &enable);
```
然后驱动层就会执行这里：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/85ccc691d0f8444ea1096d468f92b1ea~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=539&h=226&s=25579&e=png&b=ffffff)

这里就是设置是否开启OneWay Spam的检测，那么什么是OneWay Spam呢?众所周知，我们的方法是可以设置为one way的，也就是客户端不等待服务端的返回，直接继续执行自己的逻辑，那么这个时候，客户端如果疯狂的调用oneway方法，就会挤爆Binder的缓冲区，这个行为就叫OneWay Spam，当开启此项，如果进程有OneWay Spam的嫌疑，那么驱动层就会返回一个`BR_ONEWAY_SPAM_SUSPECT`，在`IPCThreadSate.cpp`里，也可以看到对此返回的处理:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bec7b360e0b346359b83238006ebc3a8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=801&h=118&s=18765&e=png&b=fffefe)

到这里，`open_driver`方法已经执行完毕了，在最后，返回了驱动的`fd`，接下来我们再回到`ProcessState`的构造方法。

接下来，会对`open_driver`的返回进行一个判断，如果打开正常，则会执行如下:
```C++
mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE,
                opened.value(), 0);
```
首先，我们来看`BINDER_VM_SIZE`的定义:
```C++
#define BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)
```
这就是我们Binder缓冲区的大小，默认情况下是1024kb - 8kb，也就是**1016kb**。

这里的`mmap`，最终会调用驱动层的`binder_mmap`:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4679b8fc851e467ca9f77804e85632cc~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=697&h=509&s=75043&e=png&b=fffefe)

可以看到，最后实际执行的`binder_alloc_mmap_handler`去进行内存的分配，这里就不展开讲了，下面说明一下，Binder驱动层所谓的一次拷贝到底是怎么实现的。

### Binder单次拷贝
Binder的单次拷贝，其原理就是Binder驱动会申请一块实际的物理内存，并且通过`mmap`内存映射机制，将服务端进程的用户空间的虚拟内存地址和内核空间的虚拟内存地址都同时映射在这一块物理内存上，这样当客户端进程向Binder驱动层传输数据的时候，Binder驱动层会把数据写入服务端进程内核空间的对应虚拟内存地址，那么对于服务端的用户空间来说，就可以直接读取到客户端传输过来的数据，因为虚拟内存地址所指向的是同一块物理内存。

到这一步，其实Binder驱动层只是分配了内核的连续内存和实际的物理内存，当驱动层调用`binder_transaction`并且在里面调用`binder_alloc_new_buf`，才会进行实际的内存映射。

至此`ProcessState`的构造方法便执行完了，接下来，我们再回到`AOSP/frameworks/native/cmds/servicemanager/main.cpp`，可以看到`ProcessState`初始化完成以后，会调用:
```C++
ps->setThreadPoolMaxThreadCount(0);
```

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/76b3d36eaff3484e93677cdbdbf70119~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=689&h=269&s=37410&e=png&b=fffefe)


这里，最终调用的也是刚才驱动层的方法，这个方法主要是设置的Native层`ProcessState`的最大线程上限为0，并且让驱动层对应的`binder_proc`的最大线程上限也为0。

好，我们再回到刚才，接着往下看这段代码:
```C++
sp<ServiceManager> manager = sp<ServiceManager>::make(std::make_unique<Access>());
if (!manager->addService("manager", manager, false /*allowIsolated*/, IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT).isOk()) {
    LOG(ERROR) << "Could not self register servicemanager";
}
```
可以看到，这里初始化了`ServiceManager`，并且调用了`addService`，并且把自己的实例传了进去，这里就是开始注册服务端了，而最先开始被注册成服务端的，第一个服务，就是我们的`ServiceManager`，也可以理解为Binder通讯的"DNS服务"。

我们进入此方法，直接找到这一段:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1bb945ecd7e44005b82e72abb50ea950~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=837&h=719&s=83682&e=png&b=fffefe)

我们可以看到，其实`ServiceManager`是通过`mNameToService`来保存我们的Binder服务端实例的，我们再看下它的类型:
```
struct Service {
    sp<IBinder> binder; // not null
    bool allowIsolated;
    int32_t dumpPriority;
    bool hasClients = false; // notifications sent on true -> false.
    bool guaranteeClient = false; // forces the client check to true
    Access::CallingContext ctx;   // process that originally registers this

    // the number of clients of the service, including servicemanager itself
    ssize_t getNodeStrongRefCount();

    ~Service();
};
using ServiceMap = std::map<std::string, Service>;
ServiceMap mNameToService;
```
可以看到它就是一个HashMap，key为服务的名字，而value则是`Service`实例，而在`Service`中，则保存了`binder`实例，但是这并不是一个实际上其它进程的服务实例，而是一个基于其句柄构造`BPBinder`实例。

我们再回到前面的`main`方法，看这两条语句:
```
IPCThreadState::self()->setTheContextObject(manager);
ps->becomeContextManager();
```
第一条，是获取或创建当前线程的`IPCThreadState`实例，并且调用它的`setTheContextObject`方法并且将ServiceManager实例赋值给它，可以看到里面的实现很简单:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f8b694e8bff3408d93ffcc9d9b3a884c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=592&h=158&s=12615&e=png&b=fffefe)

第二条，则是调用了刚才创建的`ProcessState`的`becomeContextManager`，我们进入此方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a190894c1ed44cdf84c9e122da67d8dc~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=739&h=531&s=49765&e=png&b=ffffff)

可以看到这里调用了驱动层，并且命令为`BINDER_SET_CONTEXT_MGR_EXT`，我们进入驱动层的此方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c977c654c3194e308fa3d85cb461ee7a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=450&h=231&s=22283&e=png&b=fefefe)

可以看到最终调用了驱动层的`binder_ioctl_set_ctx_mgr`，这里，其实就是根据当前`binder_proc`创建一个特殊的`binder_node`，并且将它作为特殊的`binder_node`保存下来，这里再说下`binder_node`。

### binder_node
`binder_node`可以理解为一个实体的节点，这个节点就是指一个Binder服务端，也就是说，每一个Binder服务端，在驱动层都会有对应的`binder_node`，下面是`binder_node`的结构体:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce431059f3c7490aafcb3691c53612a2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=381&h=752&s=58145&e=png&b=ffffff)

总之`becomeContextManager`这个方法的作用，你可以理解为`ServiceManager`作为一个特殊的Binder服务端，不光是在Native层特殊，在驱动层，Binder也为`ServiceManager`留下了一个特殊的位置，即一个特殊的`binder_node`，可以让驱动层随时能联系上它。

好，我们再回到`main`方法，可以看到接下来，创建了一个Native层的`Looper`，并且进入等待状态:
```C++
sp<Looper> looper = Looper::prepare(false /*allowNonCallbacks*/);

BinderCallback::setupTo(looper);
ClientCallbackCallback::setupTo(looper, manager);

#ifndef VENDORSERVICEMANAGER
if (!SetProperty("servicemanager.ready", "true")) {
    LOG(ERROR) << "Failed to set servicemanager ready property";
}
#endif

while(true) {
    looper->pollAll(-1);
}

// should not be reached
return EXIT_FAILURE;
```
至此，`ServiceManager`，也就是我们的"DNS服务"已经准备完成了。
## 第三步 注册成为服务端
那么现在，我们的Binder驱动也准备好了，"DNS服务"也注册好了，现在要做的是什么呢? 就是开启一个又一个的Binder服务了。

我们来找个Binder服务注册的例子，打开`AOSP/frameworks/av/media/audioserver/main_audioserver.cpp`，找到`main`方法，划到这部分代码:
```C++
sp<IServiceManager> sm = defaultServiceManager();
sm->addService(String16(IAudioFlinger::DEFAULT_SERVICE_NAME), afAdapter,
        false /* allowIsolated */, IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT);
```
可以看到，首先调用`defaultServiceManager`，获取到了`IServiceManager`，那我们先来看看这个方法的实现：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25fefd39e8bf4c39a8608f272b90fed7~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=954&h=596&s=76920&e=png&b=ffffff)

我们可以看到这里实际调用了`ProcessState`的`getContextObject`，再进入此方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70161bcd66744f7fb9d6e662c8538daf~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=766&h=307&s=36774&e=png&b=fffefe)

可以看到这里调用了`getStrongProxyForHandle`，并且传入了`0`，下面我不再展开，而是大致说一下这里的实现逻辑。

首先，`handle`是一个驱动层使用的概念，你可以理解为它是一个句柄，驱动层通过这个句柄，可以找到对应的`binder_node`和`binder_proc`，而我们的`ServiceManager`因为其特殊的身份，所以它的句柄是**0**。

因为我们目前与`ServiceManager`不在同一个进程，所以我们的`defaultServiceManager`获取的并不是`ServiceManager`实例，而是一个可以与`ServiceManager`进行Binder通信的`BpServiceManager`，而这个`BpServiceManager`，是由`AOSP/frameworks/native/libs/binder/aidl/android/os/IServiceManager.aidl`生成出来的，而当我们调用`IServiceManager`的方法的时候，你可以理解为最终是通过`BpServiceManager`通过Binder通讯的方式，最终调用到了我们在单独进程的那个`ServiceManager`，这中间的逻辑根我们在Java层使用aidl文件的逻辑，其实没有什么差别，具体中间是怎样实现的，有兴趣的可以详细看一下相关类的具体实现，这里不再赘述。

接下来，我们再回到刚才的`main`方法，这里的`addService`，最终会调用我们`ServiceManager`的`addService`，这里的逻辑其实和`ServiceManager`初始化后调用自己的`addService`一样，就是把当前的服务名和Binder对象添加进了`mNameToService`里。

到这里，服务可以说是已经注册完成了，接下来就是开始启动我们的Binder服务。

## 第四步 启动Binder服务
启动Binder服务端，只需要调用这两条语句就可以:
```C++
ProcessState::self()->startThreadPool();
IPCThreadState::self()->joinThreadPool();
```
首先，我们先分析`startThreadPool`，`ProcessState`的初始化，上面已经讲过，这里不再赘述:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4bdebd91c0084922847ae166fb2ce704~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=711&h=270&s=25718&e=png&b=ffffff)

可以看到，这里调用了`spawnPooledThread`，再看看这个方法:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aac7cf520ac049e98f3bb98bebedc207~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=598&h=278&s=35675&e=png&b=fffefe)

可以看到，这里面最核心的其实是创建了一个`PoolThread`，并且运行它的`run`方法，那我们再看看`PoolThread`的定义:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/390a25d991af44eb86a527e47743c705~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=630&h=395&s=24126&e=png&b=ffffff)

可以看到，里面很简单，就是调用了`IPCThreadState::self()->joinThreadPool()`。

那么问题来了，可以看到上面还会执行一次`IPCThreadState::self()->joinThreadPool()`，为什么同样的方法要执行两次?

这个问题，只要进入`joinThreadPool`就可以知道了:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a6849abcfb647a0aed8b4f1d5065dec~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=893&h=809&s=109009&e=png&b=fffefe)

这个方法，其实就是我们最终与Binder驱动层交互的主要逻辑了，可以看到这里的主要执行逻辑，就是不断地等待客户端的请求，并且处理命令，主要`result`不出问题，那么就会一直循环执行。

我们接下来再进入`getAndExecuteCommand`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9852069568af4f788d79e05504550ebe~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=723&h=508&s=61087&e=png&b=fffefe)

可以看到先调用了`talkWithDriver`，进入此方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae3559383b76404b8809888b6fa3ac5c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=685&h=621&s=59096&e=png&b=fffefe)

可以看到前面的代码，主要是为了构造一个`binder_write_read`，在Native层，它的结构体如下:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5339953dc7c945f68b640406b89c2c46~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=279&h=181&s=10586&e=png&b=fffefe)

然后我们再进入驱动层，也可以找到一个`binder_write_read`，结构如下:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a3b912c110af4eb0bd6a5d5ff6826a24~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=634&h=256&s=41822&e=png&b=fefdfd)

可以看到Native层和驱动层的结构基本是一致的，这个`binder_write_read`，代表的就是驱动层的单次读写。

我们再回到`talkWithDriver`，找到这部分代码:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4206e9da56c34d299b1d664bda733dfd~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=670&h=108&s=11542&e=png&b=fffefe)

这里就是将我们构造好的`binder_write_read`传输给驱动层了，我们再进入驱动层的处理:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d4f86af1d7a74e3ea7c9875398f49152~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=536&h=106&s=11272&e=png&b=fefbfb)

再进入`binder_ioctl_write_read`:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29ce0f33d6c048fcb8bd04b1e8245276~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=573&h=887&s=114971&e=png&b=fffefe)

可以看到里面调用了`binder_thread_write`和`binder_thread_read`去处理写和读:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af267fe5ecf9436896ddc967cf42486d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=582&h=866&s=113945&e=png&b=fffefe)

因为我们给`mOut`复制了值，这个值可能是`BC_ENTER_LOOPER`或`BC_REGISTER_LOOPER`，可以看到对应的处理，主要是给`thread`的`looper`进行赋值:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e1cd8a5e6704c7eb9339dc0d37fc426~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=786&h=603&s=114521&e=png&b=fffefe)

**要注意，当调用`ProcessState::self()->startThreadPool()`的时候，会创建一个新的线程，并设置那个新的线程为主线程，当调用`IPCThreadState::self()->joinThreadPool()`的时候，指的是当前线程，所以`ProcessState::self()->startThreadPool()`最终创建出来的线程同样会去调用`IPCThreadState::self()->joinThreadPool()`，但是那是主线程的`joinThreadPool()`，你可以理解当执行完`ProcessState::self()->startThreadPool();
IPCThreadState::self()->joinThreadPool();`的时候，目前已经有两条，即一条主线程，一条子线程可以与Binder通信的线程了**

回到刚才的`binder_ioctl_write_read`，我们执行完`binder_thread_write`之后，就要执行`binder_thread_read`了，因为在`IPCThreadState`的`talkWithDriver`中，我们的`doReceive`和`needRead`都为`true`，所以`read_size`的值是大于0的:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59e0f394fb104a368e9ae1c4f174762b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=623&h=615&s=58522&e=png&b=fffefe)

我们进入`binder_thread_read`，看到这里调用了`binder_wait_for_work`：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19628c9a42fe4c73b28cdc2d204a4b07~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=632&h=155&s=15786&e=png&b=ffffff)

可以看到从这个方法开始，线程进入等待状态，并且将当前的`thread`加入对应`binder_proc`的`waiting_threads`里面:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e13035d12e654b43b88d13e302332e57~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=701&h=593&s=78200&e=png&b=fefefe)

到这里，我们的Binder服务就启动起来了，线程会进入等待状态，当有客户端想要进行Binder通讯时，对应的线程才会被唤醒，接下来，我们就来看看当有客户端进行Binder通信时，整个的处理流程是怎么样的。

## 第五步 Binder客户端通信
比如当我们看到如下代码的时候:
```C++
sp<IBinder> binder =
    defaultServiceManager()->getService(String16("media.player"));
sp<IMediaPlayerService> service = interface_cast<IMediaPlayerService>(binder);
CHECK(service.get() != NULL);

service->addBatteryData(params);
```
前三句，我们都已经知道具体流程了，那么当我们调用`service->addBatteryData(params)`，实际调用的是什么呢? 这里实际的调用，是在`AOSP/frameworks/av/media/libmedia/IMediaPlayerService.cpp`中，可以看到`BpMediaPlayerService`的实际实现：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dfb042a697304fad99e5d8edf7f001e5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=739&h=162&s=23222&e=png&b=fffefe)

可以看到，这里面实际的传输，是通过`remote()->transact()`来进行传输的，`remote()`返回的，其实就是一个`BpBinder`实例，`transact`方法的具体实现，可以在`AOSP/frameworks/native/libs/binder/BpBinder.cpp`看到:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89eecbf75206458ebf680f0355d3b7a1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=887&h=861&s=113748&e=png&b=fffefe)

可以看到，最终实际的传输还是调用`IPCThreadState::self()->transact`来进行的，我们再进入此方法，可以看到，这里调用了`writeTransactionData`，并且传入的`cmd`参数为`BC_TRANSACTION`，再进入此方法:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8986e67fd3ff468c8b438cdf3ef29526~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=968&h=489&s=63296&e=png&b=fffefe)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a93dc6e9a8441a5b63c92ab2ae6f231~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=714&h=786&s=84880&e=png&b=ffffff)

可以看到，这个方法主要是组装了一个`binder_transaction_data`，并将其写入`mOut`中。

我们再次回到`IPCThreadState::self()->transact`中，可以看到下面这段代码:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6607e6a020a5493c89026686e41c01eb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=896&h=837&s=93402&e=png&b=ffffff)

这段代码的主要含义，就是判断当前的调用是否是`ONE_WAY`，如果不是的话，就向`waitForResponse`传入`reply`，如果是的话，就传入`nullptr`。

我们再进入`waitForResponse`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bac77a5087a1425db2c0efbcd0854639~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=804&h=527&s=61337&e=png&b=ffffff)

可以看到，从这里开始，我们又进入了`talkWithDriver`方法了，也就意味着，我们开始要往驱动层写入数据了，我们直接进入驱动层:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/db40dee367b74ab78217fdb3c4e1e2d5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=403&h=220&s=21883&e=png&b=fefcfc)

可以看到，无论是`BC_TRANSACTION`还是`BC_REPLY`，驱动层的逻辑都是相同的，就是先把传入的`binder_transaction_data`从用户空间拷贝到内核空间，然后调用`binder_transaction`方法，我们再进入此方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/886301b45f2f48568dd5836f815f963e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=627&h=104&s=21141&e=png&b=fefdfd)

可以看到，这里面的方法很长，一共900多行，所以我大致讲一下这里面做的事情，首先通过传入的`binder_transaction_data`的`handle`找到对应的服务端`binder_node`和`binder_proc`，然后将传入的数据写入到服务端进程的缓冲区内，然后从目标进程的`waiting_thread`中找一条线程来进行唤醒。

至于里面是如何通过`binder_thread`的todo列表，`binder_transaction`和`binder_work`来实现的，有兴趣可以自己看看。

接下来，就是服务端进程的工作了，我们再回到第三步所提到过的`binder_thread_read`，直接看这部分:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6290eaf6a49c405b959ee27f4d6fcd55~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=586&h=427&s=48983&e=png&b=ffffff)

这里，就是服务端的线程开始被唤醒，并且开始处理客户端的请求了，接下来主要是一系列的数据的处理和赋值，我们先不关心，主需要知道，从这里开始，服务端线程会把回传给Native层的数据的`cmd`设置为`BR_TRANSACTION`，就可以了，然后也需要知道执行完这里，我们的服务端就可以获取到客户端传过来的数据了，以上面的代码举例也就是`params`这个参数。

接下来我们再看服务端Native层的处理:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1c5165efc814286b8f5b495231e7ce8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=751&h=586&s=70915&e=png&b=ffffff)

这个时候`talkWithDriver`已经不会再阻塞，服务端下面要执行的就是`executeCommand`，进入此方法，找到这部分代码:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8626f67f756143079af2baf1c2b138f0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=721&h=336&s=44713&e=png&b=fffefe)

可以看到，这里调用的就是根据`tr.cookie`转换成的`BBinder`指针，通过`MediaPlayerService`的初始化方法我们可以知道，这其实指向的就是一个`MediaPlayerService`实例:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc3f81148e904f2c8f7426bf27fbd569~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=668&h=107&s=13547&e=png&b=fefdfd)

我们再进入`BBinder`的`transact`方法，可以看到最终调用的是自身的`onTransact`，那么我们看下`MediaPlayerService`的头文件，可以看到他继承了`BnMediaPlayerService`，然后再看`BnMediaPlayerService`，可以看到它重写了`onTransact`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5ffcd069be68471f92e22b40ee211247~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=565&h=237&s=18060&e=png&b=fffefe)

可以看到它的实现，最终是调用了`MediaPlayerService`的`addBatteryData`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1f026e11542649cdaa4649c4e0ceda41~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=522&h=139&s=15599&e=png&b=fffefe)


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b66be8e525c0429c85527d8d330b7fe8~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=509&h=84&s=9473&e=png&b=fffefe)

然后我们再回到`executeCommand`的`BR_TRANSACTION`部分的处理，可以看到接下来下做了判断，如果不是`OneWay`调用的话，就会调用`sendReply`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9a9c67a9c4543ff8accf6e7385e9229~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=724&h=729&s=97830&e=png&b=fffefe)

进入此方法，可以看到又调用了`writeTransactionData`和`waitForResponse`，将服务端的返回写入到驱动层:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ccbc47b47c3b4326979fa3904896c06d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=807&h=222&s=26967&e=png&b=ffffff)

这里就不再赘述了，这里其实还是一样的逻辑，将回复的数据写入客户端的内存缓冲区中，然后将`cmd`设置成`BR_REPLY`，然后再让客户端对应的请求去处理服务端的返回。

客户端的Native层，会将驱动层的数据先读取到`binder_transaction_data`中，然后再将数据赋值给`reply`：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af34d50cddac4cfeb24625c208fb6a19~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=850&h=659&s=97151&e=png&b=fffefe)

最后，上层再从`reply`中读取到服务端的返回值，至此，就完成了一次Binder通信。


