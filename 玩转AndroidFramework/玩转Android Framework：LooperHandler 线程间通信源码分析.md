---
theme: smartblue
---

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**


# Looper的工作方式

提到`Handler`，那么我们就绕不开`Looper`，在Android Java层，我们会经常看到如下的代码:
```Java
Looper.prepare()
Looper.loop()
```
这段代码，主要是为了阻塞当前线程，等待消息再来唤醒和处理的，他的作用基本类似于:
```Java
while(true){
    //Do something
}
```
那么接下来，我们就来看下它到底是怎么实现的。

我们先看`Looper.prepare`的源码:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1113f0cac2934e9ea4b4b2f26cd2ee9b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=686&h=256&s=28067&e=png&b=fffefe)

可以看到，这个静态方法只是创建了一个新的`Looper`，并且将它设置给`sThreadLocal`。

从上面的代码我们可以得知，`sThreadLocal`的类型是`ThreadLocal<Looper>`:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/330f41b12f8646aa80da633ddc6b13a7~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=628&h=53&s=8866&e=png&b=fffefe)

那么也就意味着，这段代码的意思就是创建了一个`Looper`并且赋值给当前线程的`sThreadLocal`，我们再看`Looper`的构造方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fbf7875184f34cd3bfac723e7d5c22ef~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=380&h=104&s=11547&e=png&b=fffefe)

可以看到它主要的作用就是创建了一个新的消息队列，并且把当前线程保存下来。

我们再看`Looper.loop`的源码:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df5861f335d44918af989685fda0c94a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=786&h=734&s=73930&e=png&b=fffefe)

可以看到，首先它调用了`myLooper`方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c5f2a2e77da8411fa0b408c044f856e4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=420&h=86&s=9288&e=png&b=fffefe)

获取到了当前线程的`Looper`实例，然后设置了一些值，最后用了一个死循环中调用了`loopOnce`方法，我们再进入此方法:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2761d07898a4e0fa8dc7114db2cc6b0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=598&h=319&s=33643&e=png&b=fffefe)

可以看到，最开始调用的是`Looper`初始化的时候就初始化好的`MessageQueue`的`next`方法，我们进入此方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a863769da5a148c2bc23efa624b07e4c~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=626&h=578&s=59343&e=png&b=fffefe)

可以看到里面又是一个死循环，并且其中调用了`nativePollOnce`，这是一个Native方法，我们进入Native层:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca8ada08ae704c52a4b15bce130bf3f1~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=723&h=117&s=20836&e=png&b=fffefe)

可以看到这里根据Java层传进来的指针恢复出来了一个`NativeMessageQueue`，并且调用了它的`pollOnce`方法。

我们首先来看`MessageQueue`的`mPtr`，可以看到它是在`MessageQueue`的构造方法里初始化的:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/86fc70ae95854a80a1d154dd69f574de~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=328&h=105&s=8826&e=png&b=fffefe)

我们在`Native`层也可以看到，它指向的就是一个`NativeMessageQueue`:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b02773a2b5e4aabaf3b1fc05152a6d4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=664&h=227&s=29604&e=png&b=fefcfc)

进入刚才提到的`NativeMessageQueue`的`pollOnce`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3293537b32de4b3d91accb106c0b67f0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=704&h=311&s=31224&e=png&b=fffefe)

可以看到，它最终调用的是`Native`层`Looper`的`pollOnce`，我们再进入此方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2f8b3b4bbd74c1fb072d4d3a158fa51~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=718&h=783&s=76038&e=png&b=ffffff)

再进入`pollInner`:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/18ad6ef2aa3441f4acbd1e7c5b61c7fe~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=1078&h=671&s=83248&e=png&b=ffffff)

可以看到，最终是调用了`epoll_wait`，此时线程进入等待状态。

从这里我们就可以知道，我们Java层的`Looper`的`loop`最终会进入`Native`层的`Looper`，然后执行它的`pollInner`，然后在其中使用`epoll`机制来等待消息的唤醒。

# Handler的工作方式

首先，我们进入`Handler`的构造方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a59232c17cb04739bf95d98a19bf9b72~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=793&h=466&s=54269&e=png&b=ffffff)

可以看到，当我们初始化`Handler`的时候，`Handler`会保存当前线程的`Looper`和`Looper`中的`MessageQueue`，这个`Looper`，是我们在调用`Looper`的`prepare`方法的时候创建的。

我们再看`sendMessage`方法:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/307e054df26f44ecaf6a6083e0ca7751~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=511&h=86&s=10922&e=png&b=fefdfd)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8383327202a846198b0921828efc9480~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=743&h=150&s=19270&e=png&b=fffefe)

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d055ac272ccc4caeb1ed0e143209136a~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=620&h=233&s=26858&e=png&b=fffefe)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8fda833bfaa24d82a189ea4d76ba292b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=676&h=229&s=23517&e=png&b=ffffff)

可以看到，这里最终调用的，是我们在初始化的时候就获取到的`MessageQueue`的`enqueueMessage`，我们进入此方法:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5bfe8696baea474e92e5637f732af7ed~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=696&h=749&s=64749&e=png&b=fffefe)

可以看到，这里会把消息加入队列，并且调用`nativeWake`方法，我们进入Native层，找到此方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80b23d3f4e53488e8b4e179b221edee4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=747&h=112&s=16536&e=png&b=fffefe)

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2afda79b48d4272abbb4315adb3db82~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=299&h=76&s=5397&e=png&b=fffefe)

可以看到这里会调用Native层对应`Looper`的`wake`:

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4cabaa0305664718b050f3909d4fee75~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=884&h=321&s=36428&e=png&b=ffffff)

最终这里也是利用`epoll`机制，进行了唤醒。

# Looper处理消息

我们再回到Java层的`Looper`，找到它的`loopOnce`方法，当我们获取到Message的时候，可以看到在后面的处理中有这样的代码:

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/02a91c91096e41b3b252efec2e16a720~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=638&h=630&s=77948&e=png&b=fffefe)

这里调用了`Message`的`target`的`dispatchMessage`方法，这个`target`的赋值，我们可以在`Handler`的`enqueueMessage`中找到：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc179fe3ff474bd19aa4a8691e6bf9f2~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=673&h=239&s=25766&e=png&b=ffffff)

也就是说，这个`target`指的就是这个`Handler`自身，我们进入它的`dispatchMessage`方法:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c98aa90bd36344bd8ea6c388f55e3ef4~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=443&h=277&s=20738&e=png&b=fffefe)

可以看到在这里，做了一个判断，如果`Message`有`callback`，那么就会调用对应`Message`的`callback`，如果初始化的时候传进来了`Callback`，那么执行此`Callback`的`handleMessage`，否则调用自身的`handleMessage`。

那么到这里，我们就知道了为什么`Handler`可以进行跨线程通信，因为每个`Handler`在初始化的时候都保存了当前线程的`Looper`对象，一般这个`Looper`在Android中都是`MainLooper`，也就是主线程的`Looper`，然后让它去处理自身的回调，这就保证了回调一定是执行在主线程的，因为`Handler`的初始化，一般就发生在主线程。