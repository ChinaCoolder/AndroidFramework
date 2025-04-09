---
theme: smartblue
---

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**

# Android的触摸事件是如何获取的

平常我们在应用层开发的时候，当我们想去处理`view`的触摸事件，可能会写以下的代码:

```Kotlin
mView.setOnTouchListener(object : OnTouchListener {
    override fun onTouch(v: View?, event: MotionEvent?): Boolean {
       
    }
})
```

然后根据获取的`MotionEvent`去处理具体的触摸事件，达成我们想要的效果，那么问题来了，从手指去触摸手机屏幕开始，事件是怎么传递到自定义View的`onTouch`这里的?

这其实就是`Inputflinger`的工作了。

首先打开`adb shell`， 然后输入`getevent -lrt`，可以看到如下界面:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d616c8eb38c4ea1b3fcd51c82c328e0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=628&h=369&s=42244&e=png&b=300a25)

这个时候我们使用手指滑动屏幕，就可以看到如下的打印:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/731690e435a045e8b382d27b5e1f7bb4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=944&h=568&s=135343&e=png&b=310a25)

这里，其实就是驱动层输出的数据了，我们的`Inputflinger`在触摸这里的处理，就是要将我们从驱动层获取的这些数据，最终发送到我们触摸的View中去，然后去处理我们的上层逻辑，比如滑动，点击等。

# Android驱动层协议
Android触摸相关的官方文档，可以看这里:
> https://source.android.com/docs/core/interaction/input/touch-devices?hl=zh-cn

从文档中我们可以得知，Android的多点触控基于的是Linux的`multi-touch-protocol`，这里可以看到具体的协议文档：
> https://www.kernel.org/doc/Documentation/input/multi-touch-protocol.txt

在官方文档中，我们可以看到驱动层传输的这些数据都代表着什么:


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2a419241019442f7831e13b191d8f244~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=839&h=786&s=219269&e=png&b=fffefe)

现在，我们清楚了这些之后，就可以看看，Android是怎么初步帮我们去解析并分发这些数据的。

# 启动InputManagerService

首先，我们先打开`/frameworks/base/services/java/com/android/server/SystemServer.java`，可以看到`InputManagerService`的初始化和启动相关的代码:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67418b763d05438d87c4769e8f1eaa07~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=589&h=69&s=12077&e=png&b=fffefe)


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d52e5eb0466744e6a71f414db27e7074~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=742&h=93&s=15129&e=png&b=fffefe)

我们先来看`InputManagerService`的初始化方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/31d729443b654a17a4be6a34408e2da3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=788&h=771&s=129419&e=png&b=fffefe)

可以看到里面有个`mNative`，从上面的代码，我们可以得知这个`mNative`是`NativeInputManagerService`类型，我们再看`mNative`的初始化:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/48bfaf9517da43adbdeca6335ead9cce~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=370&h=42&s=7108&e=png&b=a3d1fe)
![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8957379551e34f4e9e77aa0399d8be4f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=672&h=111&s=14321&e=png&b=fffefe)
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0623b9121ac847509b93e3d5a5ab487c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=710&h=395&s=38743&e=png&b=fffefe)

可以看到，最终的实现都是Native方法，然后在初始化的时候就调用了Native方法`init`，我们再打开`/frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp`，就可以看到Native`init`的具体实现:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab055a2a783d4e41b9c7308a2a5c4473~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=805&h=403&s=59710&e=png&b=fffefe)

可以看到里面初始化了`NativeInputManager`，我们再去它的初始化方法看一看:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/acdce4c7b84b422fbfb6f7ab85260e4f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=732&h=234&s=34661&e=png&b=fffefe)

可以看到这里创建了一个Native层的`InputManager`，并且将它添加为了一个Binder服务，我们再看`InputManager`的初始化方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e610c4a7f6914256abeee6d38f2ae501~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=678&h=261&s=43141&e=png&b=fefcfc)

可以看到这里初始化了四个对象`mReader`，`mBlocker`，`mProcessor`和`mDispatcher`，这里面的`Blocker`和`Processor`，其实都是过滤事件的，我们重点要关注的其实是`mReader`和`mDispatcher`，因为他们负责的就是事件的读取与分发，并且可以看到，这里使用的是责任链的模式去一层层的传递事件数据。

这里的初始化我们先略过，接下来我们回到`/frameworks/base/services/java/com/android/server/SystemServer.java`，可以看到接下来就是启动我们的`InputManagerService`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/423feba429cb407db425c59afaf73a87~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=565&h=84&s=14292&e=png&b=fffefe)

我们再进入`start`方法，可以看到最终调用的还是Native得`start`方法:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d5474cce73b4087b90e83fd918f1ef6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=422&h=150&s=16418&e=png&b=fffefe)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64365ff199e9446bb937648b18b866cf~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=690&h=193&s=24223&e=png&b=fffefe)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/658bdbd0345b4ea39d5153bb7a96c14c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=671&h=380&s=30416&e=png&b=ffffff)

可以看到，这里先调用的是`mDispatcher`的`start`，然后才是`mReader`的`start`，我们先看`mReader`的`start`:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/28ad280d2e2143449e5313b0de153b19~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=958&h=193&s=21458&e=png&b=ffffff)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64a0b7c0b8fd4126b1cf034aaf5257b1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=819&h=131&s=20905&e=png&b=fffefe)

可以看到这里就是开启了一个线程，并且最终不断调用自身的`loopOnce`，那么这个`loopOnce`里面，其实就是我们最终`InputReader`去读取驱动事件的主要逻辑了。

# 加载并读取驱动数据
首先，我们进入`loopOnce`方法，可以看到这里调用了`mEventHub`的`getEvents`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a54cee0d236d46fb9fc9b4d9896155d1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=804&h=569&s=73176&e=png&b=fffefe)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/316a216e3da9482dab99d9f89f8a94d6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=590&h=161&s=18331&e=png&b=fffefe)

在此方法中找到如下代码，可以看到这里调用了`scanDevicesLocked`：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4983a7911ed54ffbb61c909e9affeba9~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=458&h=135&s=9149&e=png&b=a5d2ff)

我们再进入此方法:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2f6f7ed8601c4c6ab43a53a767f25cd1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=793&h=569&s=62982&e=png&b=ffffff)

这个方法的主要作用，就是扫描我们的输入驱动设备了，可以看到目录扫描的`DEVICE_INPUT_PATH`的值为`/dev/input` 如果对Linux系统有了解的话就会知道这个目录就是Linux的输入子系统，不光是触摸屏，键盘，手柄等其它输入设备也会挂载到这个目录下面。

先进入`scanDirLocked`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4eb04a5b641c40d4bfd043769544d130~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=817&h=141&s=19847&e=png&b=fffefe)

然后再进入`openDeviceLocked`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9a00a3ba3134f558eb23cd9c79e807d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=831&h=344&s=49146&e=png&b=fefdfd)

可以看到这个方法，就是用`open`和`ioctl`等系统调用去打开我们的驱动设备了，并且最终，会转换成一个`Device`对象，并存储到`mOpeningDevices`中。

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e3af286a93ae458790b3c0eeefbb7eea~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=576&h=635&s=75180&e=png&b=fffefe)
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4c22e6d09f8445da65226f413efba14~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=760&h=272&s=32233&e=png&b=fffefe)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/51750b3321bc477f88d6a3cd0427184f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=651&h=110&s=15488&e=png&b=fffefe)

打开设备之后，会进入第一段循环，这里会取出所有`mOpeningDevices`，往要返回的事件中添加打开设备事件，并且把`Device`实例添加到`mDevice`里面:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c3c1cdb4360e465e9783e8f567d00ec2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=897&h=688&s=85311&e=png&b=fffefe)

然后再下面是这段:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aefc6d44f6c84e45bc2d408b70276f86~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=389&h=222&s=15274&e=png&b=fffefe)

这段，是往要返回的事件中添加扫描设备结束事件。

再然后，是第二段循环:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/acefbf1cc65c4ba7a7d5bc96c445bacf~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=694&h=233&s=26778&e=png&b=fffefe)

注意，这里当我们第一次进入的时候，是不会进来的，因为这个时候我们的`mPendingEventCount`是0，所以继续往下看:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d59d40762fff4572914b1556568e4c6f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=660&h=98&s=8758&e=png&b=fefdfd)

到这里，我们就会直接返回设备扫描和添加的相关事件，这里我们先跳过，因为我们知道`InputThread`会不断的运行`loopOnce`，那么当我们第二次进入这里的时候，因为我们的设备已经扫描过了，所以会跳过这一段，接着往下运行:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2cd61af9a2634742b5bf100cf22fbcee~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1025&h=534&s=52589&e=png&b=ffffff)

这里，线程就调用了`epoll_wait`进入等待状态了，然后当我们读取到触摸事件的时候，这个时候就会继续往下运行并且将`mPendingEventCount`置为非`0`，并且再次循环，并且会进入到刚才提到的第二段循环中，进行`epoll`事件的处理，进入此循环，直接看这段代码：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/efbbf7b830a84cba92b6727307c6f14f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=616&h=448&s=48438&e=png&b=fffefe)

可以看到这里就会把获取到的触摸事件添加到`events`里，然后再通过后面相同的逻辑把事件返回回去:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4da9be5d3a6b4b56ad1d16e50534e1fe~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=640&h=108&s=8727&e=png&b=fffefe)

# InputReader加工数据
现在，我们已经从驱动层获取到了触摸数据，但是现在的数据是`RawEvent`，是非常原始的数据，所以下一步，我们就要对获取的数据进行加工:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84e76eaf18a549daa84c0af181c0dca3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=821&h=245&s=23752&e=png&b=fffefe)

回到`InputReader`的`loopOnce`方法，可以看到接下来就调用了`processEventsLocked`，我们进入此方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72671f9d5766415ebcb1a0944a94027d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=851&h=864&s=111707&e=png&b=fffefe)

可以看到，首先，我们处理数据之后返回的对象是是一个`NotifyArgs`的`list`，`NotifyArgs`具体的类型可能会是这些:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d7816108c7e548f99780cc2e012926ac~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=825&h=105&s=13750&e=png&b=fefcfc)

然后第一步，会先取出`RawEvent`的`type`，并进行判断`type < EventHubInterface::FIRST_SYNTHETIC_EVENT`，首先这个判断条件大概率是`true`的，因为我们触摸的时候返回的`type`基本上都是`EV_ABS`，`EV_KEY`之类的，在`/bionic/libc/kernel/uapi/linux/input-event-codes.h`可以看到这些`type`的值都很低:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd568d17c80b452b83290c68d9a23e11~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=470&h=230&s=13011&e=png&b=ffffff)

而`FIRST_SYNTHETIC_EVENT`的值却很高:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aeac1b964c4c4b0da042f79f8c003a70~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=421&h=199&s=18862&e=png&b=fffefe)

所以当我们触摸的时候，这里的判断基本都是`true`。

然后再看里面的处理，是调用了`processEventsForDeviceLocked`：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cdaaddc4a31c4a18ad9993aa189b9506~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=764&h=381&s=46160&e=png&b=fffefe)

然后再下面，就是`InputDevice`的`process`方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1aecd8dc0ca4f89b964ee929383b568~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=944&h=822&s=119635&e=png&b=fffefe)

可以看到，这里最终调用的就是`mapper`的`process`，那么这是什么意思呢?

## Mapper的工作方式
打开`/frameworks/native/services/inputflinger/reader/mapper/`，我们可以看到很多的`Mapper`文件:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d5b2275c1464d13a655b45c6f95d4da~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=800&h=494&s=38354&e=png&b=fcfbfb)

这些`Mapper`文件，最主要的作用其实就是转换我们的`RawEvent`的，他们会根据不同的输入协议，采用不同的逻辑，最后将我们的`RawEvent`转换成不同的`NotifyArgs`，比如我们想关心的是触摸屏的输入处理，那我们就打开`MultiTouchInputMapper.cpp`看下它的处理：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2a0a7cbf0482417291ed1bf7d22d8f8c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=673&h=150&s=19558&e=png&b=fffefe)

具体的代码十分复杂，有兴趣可以自己看看，里面实现的各个方法其实就是对驱动层输入的数据的各种校准，储存，和计算，最后生成数据供更上层处理，建议结合协议文档去看，比如里面`Slot`的概念，其实就是来自于协议中的概念:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1aecc60005fe466da6449e5888356b68~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=592&h=78&s=13034&e=png&b=fffdfd)

# 传递事件给InputDispacher

我们再回到`InputReader`的`loopOnce`，可以看到当我们的`notifyArgs`准备好的时候，就会调用`notifyAll`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7885fb6739e480b9a30a24d0f9cbb99~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=534&h=129&s=15072&e=png&b=fffefe)

这个`mQueuedListener`，其实就是`InputManager`初始化的时候传进来的`mBlocker`，因为这里使用了责任链模式，事件最终经过过滤会传递给`InputManager`的`mDispacher`，我们再看下`notify`方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e281f66fc99d403eb0b7a6c9aaa12228~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=893&h=330&s=75340&e=png&b=fefdfd)

可以看到，不同的`NotifyArgs`会调用不同的`notify`方法，因为我们传递进来的是触摸事件，也就是`NotifyMotionArgs`，所以最后调用的其实是`mDispacher`的`notifyMotion`方法。

那么，接下来我们再进入`/frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp`的`notifyMotion`，略过上面的一些打印log和检查的逻辑，直接进入方法的后面:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f1860f5607b4a5ba6d60a7f82c5cb6d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=783&h=435&s=61264&e=png&b=fffefe)

可以看到，这部分的逻辑主要就是用传入进来的`NotifyMotionArgs`转换成一个`MotionEntry`，之后调用`enqueueInboundEventLocked`方法，然后进入此方法，可以看到这里主要就是将此`MotionEntry`push给`mInboundQueue`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/70e861e1c50f493285b6c5e547f80c2b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=715&h=131&s=22089&e=png&b=fffefe)

**注意，`MotionEntry`是`EventEntry`的子类，`mInboundQueue`后续处理的时候，都是使用`EventEntry`来进行处理的**

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/246e65ec85fd486a8679ade2c1a7cf57~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=300&h=163&s=10422&e=png&b=fffefe)

那么接下来，`InputDispatcher`是怎么处理的呢?

# InputDispatcher的分发处理
接下来，我们回到Native端的`InputManager`，找到它的`start`方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47e01b045ed940a6b82189f81f1e3e15~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=665&h=373&s=30275&e=png&b=ffffff)

可以看到，在调用`InputReader`的`start`之前，`InputManager`先调用了`InputDispatcher`的`start`方法，我们进入此方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2172e8eda1274c268ce5e067a9db4c1b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=659&h=196&s=20101&e=png&b=fffefe)
可以看到这里所做的，就是新开始一个线程，然后不停调用`dispatchOnce`方法，我们进入此方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6f9ab38b2c845c2bdadad572825edf0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=843&h=784&s=96531&e=png&b=fffefe)

可以看到里面调用了`dispatchOnceInnerLocked`，我们再进入此方法，然后跳转到以下这段:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/296873248cdb4341a9f58ff6cc3117d1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=658&h=844&s=87936&e=png&b=fffefe)

可以看到这里的逻辑，就是从`mInboundQueue`取出一个`EventEntry`，然后将它赋值给`mPendingEvent`，接下来，就会对`mPendingEvent`的`type`进行判断:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/35435f6d6791480b81c8eba88f1e506a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=343&h=30&s=3582&e=png&b=fffefe)

我们直接进入`Motion`部分:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6cbb3200008c4ebf95a769c21789977e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=788&h=332&s=53369&e=png&b=fefdfd)

可以看到，这里调用了`dispatchMotionLocked`，我们进入此方法，找到如下代码段:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef371f25e6774a4a9571784006316895~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=986&h=692&s=107600&e=png&b=fffefe)

这部分的逻辑，就是找到我们的目标窗口，也就是我们事件分发的最终目的地，最后再执行真正的分发方法`dispatchEventLocked`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7f60f860fe3497da4c538155501e61d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=487&h=41&s=5624&e=png&b=fffefe)

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3b51e78e119e470ca90bb9cd5aa40f64~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=842&h=624&s=81136&e=png&b=fffefe)

可以看到，这里的逻辑，就是根据我们的`InputTarget`中的`inputChannel`的`ConnectionToken`，找到我们对应的`Connection`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c20e1a3164b043959f08f45aacee0790~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=809&h=296&s=27581&e=png&b=ffffff)

然后再调用`prepareDispatchCycleLocked`，这里的逻辑，我先按下不表，目前只需要知道`Connection`这里是都能取到值就行，因为`InputChannel`涉及到`WMS`。

我们再进入`prepareDispatchCycleLocked`，可以看到最后调用的是`enqueueDispatchEntriesLocked`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1f88966f7ae64d37ba28285c79705f16~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=819&h=781&s=105702&e=png&b=fffefe)


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4932a9f7f914999b23720457a8b56e3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=816&h=760&s=127447&e=png&b=fffefe)

我们可以看到，这里先调用了6次`enqueueDispatchEntryLocked`，最后调用`startDispatchCycleLocked`，我们先进入`enqueueDispatchEntryLocked`，可以看到它的主要逻辑就是根据我们的`EventEntry`，创建了一个`DispatchEntry`，然后将此`DispatchEntry`添加给对应`Connection`的`outboundQueue`中:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0cef031927624c64a884d7336e7f1818~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=702&h=120&s=15100&e=png&b=fefdfd)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7a98f9693174262ae77ae219ffb8bc4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=563&h=85&s=11892&e=png&b=fefcfc)

然后我们再进入`startDispatchCycleLocked`，可以看到里面调用了`publishMotionEvent`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/55d298d3e3c94b8db518da4759dfc816~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=667&h=260&s=26715&e=png&b=fffefe)

然后可以看到，最终里面调用的就是对应`Connection`的`inputPublisher`的`publishMotionEvent`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/af6ab33bd9b44eacbb5563ca74a3f60f~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=782&h=318&s=51066&e=png&b=fffefe)

我们再进入此方法，可以看到这里又将事件封装成了一个`InputMessage`，最后调用了`InputChannel`的`sendMessage`:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1ab0f3d4f2574eebbfbf3f1156975de5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=658&h=879&s=135280&e=png&b=ffffff)

再进入此方法，可以看到事件最终通过`Socket`的方式发送出去了:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b99d0bc0aa984c42bcadeb6316bf3311~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=857&h=737&s=89252&e=png&b=ffffff)

到这一步，我们的问题就来了，最终这个事件到底会被谁接收? 还有上面提到的那些`InputChannel`，`Connection`到底是什么？ 那么我们接下来就要找到这些问题的答案。

# WMS注册InputChannel
首先，我们要知道我们的Activity的根View实际上是一个`DecorView`，当我们想要去显示我们的界面的时候，需要去通过`WindowManager`去执行`addView`方法，这个方法最终执行的，是`WindowManagerGlobal`的`addView`方法，而这个方法里面会创建一个`ViewRootImpl`，最后会调用它的`setView`，这个方法中，我们可以找到如下代码:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/45a15ea7481d4dd599297fba300fc3cf~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=625&h=114&s=14192&e=png&b=fffefe)


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b22894ce8314e97bd13ebce3b8e256e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=642&h=94&s=18170&e=png&b=fffefe)

我们可以看到，在这个方法里，初始化了一个`InputChannel`，并且将这个`InputChannel`传递给了这个方法`addToDisplayAsUser`，我们再跳转到`/frameworks/base/services/core/java/com/android/server/wm/Session.java`，找到这个方法，可以看到这里调用的是`WMS`的`addWindow`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86836f4dddf144b1a6baa80697384371~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=779&h=261&s=39290&e=png&b=fffefe)

我们进入此方法，可以看到这里又调用了`openInputChannel`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9c0b8e8c0ff4df083b4160aaa96a1fc~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=600&h=127&s=14636&e=png&b=fffefe)

在`openInputChannel`中，调用了`InputManagerService`的`createInputChannel`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dbea0458984145a5acbdebd94e8d47d2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=653&h=277&s=38878&e=png&b=fffefe)

在其中，又调用了`NativeInputManagerService`的`createInputChannel`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0db42bd3376044ad8e1ff79420425026~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=433&h=69&s=7808&e=png&b=fffefe)

接下来我们再回到Native层:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d1c3b3875c3427f9017e83c1f155ef5~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=709&h=133&s=15707&e=png&b=fffefe)

可以看到，这里最终又调用回到了`InputDispatcher`的`createInputChannel`：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d768a0bd25f4f638a52edc410fc0b60~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=854&h=829&s=105544&e=png&b=fffefe)

我们先来看`openInputChannelPair`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e18d9f53dc94b6ba7ffb1951db242a3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=964&h=677&s=116322&e=png&b=fefdfd)

可以看到，这个方法主要就是创建了一个`socketPair`和一个`BBinder`，并且将`fd`和创建的`binder`传给新创建的两个`InputChannel`，分别代表服务端和客户端。

我们再回到`createInputChannel`，可以看到接下来就是根据服务端的`InputChannel`创建一个`Connection`，并且将它保存到`InputDispatcher`的`mConnectionsByToken`，最后再将客户端的`InputChannel`传递回去。

最后，这个客户端的`InputChannel`会被保存在应用层，至此，我们就完成了应用层的`Window`和`InputDispatcher`的联系，数据最终会通过`Socket`的方式发送给不同进程中的应用。

# InputReceiver的创建与初始化
接下来，我们再回到`ViewRootImpl`的`setView`，找到如下代码块：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d58e4cdd7567457fa9f75f2b74b388bc~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=678&h=380&s=51125&e=png&b=fffefe)

这个时候，我们已经知道了，这块的`inputChannel`是`SocketPair`通信中代表客户端的`InputChannel`，在这段代码中，我们可以看到`mInputEventReceiver`被初始化为了一个`WindowInputEventReceiver`，进入此类，我们可以看到它继承了`InputEventReceiver`，进入`InputEventReceiver`的构造方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e558064947884441b63ad72a77208806~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=701&h=356&s=43498&e=png&b=fffefe)

可以看到这里调用`nativeInit`，并且将`InputChannel`和`MessageQueue`都传了进去，接下来我们再打开`/frameworks/base/core/jni/android_view_InputEventReceiver.cpp`，找到`nativeInit`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/aa0e379649e147ff9e905ffa707b07d6~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=859&h=661&s=83809&e=png&b=fffefe)

可以看到这里创建了一个`NativeInputEventReceiver`，并且调用了它的`initialize`，我们进入此方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/262a913813e0460496fdcad464ff980a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=425&h=91&s=9491&e=png&b=fffefe)

可以看到这里调用了`setFdEvents`，我们再进入此方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0f40a1cb610482489b5ab9277a692bf~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=789&h=253&s=27946&e=png&b=ffffff)

可以看到它是将`InputChannel`的`fd`传给了`Looper`，并且设置了`callBack`为当前的`NativeInputEventReceiver`，也就是说当通过`Socket`接收到数据时，最终会调用当前`NativeInputEventReceiver`的`handleEvent`，因为我们的`NativeInputEventReceiver`是一个`LooperCallback`的子类:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f572b7ce992e4850ac7d499dc50a1a23~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=590&h=265&s=39415&e=png&b=fffefe)


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b404743a9981429898943964f40d0fac~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=661&h=367&s=39196&e=png&b=fefdfd)

也就是说，当我们的触摸事件通过`Socket`的方式传输出去的时候，最终调用的，就是`NativeInputEventReceiver`的`handleEvent`。

# InputReceiver处理事件
我们进入`handleEvent`，可以看到里面调用了`consumeEvents`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/31c689a32f5e47c28af526ddb0043942~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=835&h=809&s=102306&e=png&b=fffefe)

进入此方法，找到如下代码块:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9df56d4edb144b78b3fc9137644438ab~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=628&h=189&s=25113&e=png&b=fefdfd)

可以看到它先调用了`mInputConsumer`的`consume`，`mInputConsumer`是在`NativeInputReceiver`初始化的时候根据`InputChannel`创建的，而它的`consume`方法主要就是为了用`InputChannel`通过`Socket`的方式读取到传输过来的触摸数据，并且将其写入到`InputEvent`里面:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6bb619543cc24658b306097ddad6846b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=816&h=436&s=55347&e=png&b=fffefe)

接下来我们回到`consumeEvents`往下看，可以看到在这创建了一个`inputEventObj`，并且将上面获取的`InputEvent`转换为`MotionEvent`，并且将其赋值给`inputEventObj`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b415a60cb2c4bb7b92dd1b9b80aa506~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=739&h=505&s=62722&e=png&b=fffefe)

最后，我们就可以看到这个`inputEventObj`会被传递给Java层并回调其`dispatchInputEvent`方法:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/feb7606836b24fadb6b96f5111d77acb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=739&h=240&s=34781&e=png&b=fffefe)

我们再回到Java层的`InputEventReceiver`，找到对应的`dispatchInputEvent`方法，可以看见这里调用的是自身的`onInputEvent`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de375a8de4ae4304bcca7fccd85320e1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=690&h=197&s=27622&e=png&b=fffefe)

我们再回到`ViewRootImpl`，可以看到里面的`WindowInputEventReceiver`类重写了`onInputEvent`方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/787c1dc7dcbc472b807a80677f7b6806~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=771&h=574&s=68846&e=png&b=ffffff)

我们再进入`enqueueInputEvent`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8354155c0281493aab40969bfb8f0cea~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=780&h=934&s=110493&e=png&b=fffefe)

可以看到，这个方法主要就是将我们的`InputEvent`存储进这个`QueuedInputEvent`中，最后执行`doProcessInputEvents`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b299177479b8433588856897df9d52fd~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=764&h=593&s=64452&e=png&b=fffefe)

在这里，会不断的取出待处理的`InputEvent`，并且执行`deliverInputEvent`，在这个方法中，可以看到其中调用的是`InputStage`的`deliver`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7b6aa1efce6a4a248d1131c2d30d480c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=728&h=499&s=49044&e=png&b=ffffff)

**注意，这里的`InputStage`，使用的也是类似于责任链的模式，`deliver`主要负责调用自身的`OnProcess`方法，然后调用自身的`apply`，然后根据`OnProcess`的返回，决定调用`forward`或者`finish`，如果继续向下传递数据，就会调用`forward`，`forward`会去调用`onDeliverToNext`，`mNext`也会是一个`InputStage`，他也会执行下一个`InputStage`的`deliver`方法，直到`mNext`是空值，就会调用`finishInputEvent`，结束这个`InputEvent`的处理**

我们可以看到在`ViewRootImpl`的`setView`方法中，一共有这些`InputStage`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a53d8be9ed34e92b62a9e37dd4d643e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=652&h=364&s=69746&e=png&b=fffdfd)

最终我们事件的派发，实际上是在`ViewPostImeInputStage`的`OnProcess`方法中:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08c5d644939e4965997ca884ebdecbaf~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=610&h=366&s=39585&e=png&b=fffefe)

因为我们是事件来源是触摸屏，所以会执行`processPointerEvent`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e83a61b64ad440109b5d80fce7472bfb~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=779&h=307&s=42338&e=png&b=fefcfc)

可以看到，这里会调用`dispatchPointerEvent`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96ce5ce2282546b2a529d0492a8d4ed1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=732&h=526&s=70338&e=png&b=fffefe)

在进入方法，可以看到最终调用的是自身的`dispatchTouchEvent`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4abcb8049c664251bb02e485602b2a02~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=691&h=190&s=23451&e=png&b=fffefe)

因为我们目前的这个`ViewRootImpl`中的`mView`实际上是一个`DecorView`，所以最终调用的就是`DecorView`中的`dispatchTouchEvent`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6545f306e354554886e8cc36b3825be~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=585&h=150&s=24535&e=png&b=fffefe)

这里的`mWindow`实际上就是`Activity`持有的`PhoneWindow`，在`Activity`执行`attach`的时候，会将`PhoneWindow`的`CallBack`设置为自身:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/30d0616b51aa40f7bd8697cfbdef1421~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=753&h=385&s=68409&e=png&b=fffefe)

所以最终这里执行的，就是`Activity`的`dispatchTouchEvent`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9e1e79889b4b4df08e1ff311de4af939~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=600&h=490&s=49247&e=png&b=fffefe)

可以看到这里执行的就是持有的`PhoneWindow`的`superDispatchTouchEvent`，进入此方法，可以看到这里执行的是`DecorView`的`superDispatchTouchEvent`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c604ac416824a649960330ee1ecf361~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=498&h=112&s=11100&e=png&b=ffffff)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9cf95784744a469cae5e4409c68af258~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=500&h=77&s=9289&e=png&b=fffefe)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b279a80d7d244a8cbe1c0ceec6ad19ee~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=708&h=224&s=32325&e=png&b=fffefe)

我们可以看到，最终执行的，其实就是`ViewGroup`的`dispatchTouchEvent`，在这个方法中，主要处理的就是根据触摸事件的坐标等信息，找到对应的`ChildView`，并且调用`dispatchTransformedTouchEvent`方法，然后调用子`View`的`dispatchTouchEvent`方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ebd24a5d98fb4293b0beb89b38693f65~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=741&h=121&s=17600&e=png&b=fffefe)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9c75978ea2f4336abfdb0b24f363047~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=756&h=376&s=48803&e=png&b=fffefe)

最终在子`View`的`dispatchTouchEvent`，回调自身的`OnTouchListener`的`onTouch`方法，完成最后的任务:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07e5f070b85441f38772c6cbc932d06e~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=774&h=267&s=31000&e=png&b=ffffff)

# 总结
整个`Inputflinger`的关于触摸的事件处理及分发，主要的流程已经梳理出来了，可以看到里面涉及到了一些`WMS`和`AMS`的内容，还有`finishTouchEvent`的部分，这部分涉及到`ANR`的处理，我都会持续更新，希望看完感觉有帮助的兄弟能点个赞，谢谢。