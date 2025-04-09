---
theme: smartblue
---

本文将介绍有关AIDL相关的一些知识。

系统版本: **Ubuntu 22.04 lts**

AOSP分支: **android-14.0.0_r28**

# 什么是AIDL
AIDL，全称 `Android Interface Definition Language`，是一种接口定义语言，这里是关于AIDL的官方文档:

> https://developer.android.com/develop/background-work/services/aidl?hl=zh-cn

# 为什么我们需要AIDL
要回答这个问题，要先看看AIDL替我们做了什么，我们先创建一个`IRemoteService.aidl`，内容如下:
```Java
interface IRemoteService {
    void Wow(inout int[] test);
}
```
然后创建一个Service，实现所生成的Stub:
```Kotlin
class RemoteService : Service() {

    override fun onCreate() {
        super.onCreate()

        bindService(Intent(this, RemoteService::class.java), object : ServiceConnection{
            override fun onServiceConnected(name: ComponentName?, service: IBinder?) {

            }

            override fun onServiceDisconnected(name: ComponentName?) {

            }

        }, BIND_AUTO_CREATE)
    }

    override fun onBind(intent: Intent): IBinder {
        // Return the interface.
        return binder
    }


    private val binder = object : IRemoteService.Stub() {
        override fun Wow(test: IntArray) {

        }
    }
}
```
然后进入`IRemoteService`，可以看到基于我们所提供的AIDL文件，为我们生成的代码:
```Java
public interface IRemoteService extends android.os.IInterface
{
  /** Default implementation for IRemoteService. */
  public static class Default implements com.example.aidlapplication.ui.theme.IRemoteService
  {
    @Override public void Wow(int[] test) throws android.os.RemoteException
    {
    }
    @Override
    public android.os.IBinder asBinder() {
      return null;
    }
  }
  /** Local-side IPC implementation stub class. */
  public static abstract class Stub extends android.os.Binder implements com.example.aidlapplication.ui.theme.IRemoteService
  {
    /** Construct the stub at attach it to the interface. */
    public Stub()
    {
      this.attachInterface(this, DESCRIPTOR);
    }
    /**
     * Cast an IBinder object into an com.example.aidlapplication.ui.theme.IRemoteService interface,
     * generating a proxy if needed.
     */
    public static com.example.aidlapplication.ui.theme.IRemoteService asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.example.aidlapplication.ui.theme.IRemoteService))) {
        return ((com.example.aidlapplication.ui.theme.IRemoteService)iin);
      }
      return new com.example.aidlapplication.ui.theme.IRemoteService.Stub.Proxy(obj);
    }
    @Override public android.os.IBinder asBinder()
    {
      return this;
    }
    @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
    {
      java.lang.String descriptor = DESCRIPTOR;
      if (code >= android.os.IBinder.FIRST_CALL_TRANSACTION && code <= android.os.IBinder.LAST_CALL_TRANSACTION) {
        data.enforceInterface(descriptor);
      }
      switch (code)
      {
        case INTERFACE_TRANSACTION:
        {
          reply.writeString(descriptor);
          return true;
        }
      }
      switch (code)
      {
        case TRANSACTION_Wow:
        {
          int[] _arg0;
          _arg0 = data.createIntArray();
          this.Wow(_arg0);
          reply.writeNoException();
          reply.writeIntArray(_arg0);
          break;
        }
        default:
        {
          return super.onTransact(code, data, reply, flags);
        }
      }
      return true;
    }
    private static class Proxy implements com.example.aidlapplication.ui.theme.IRemoteService
    {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
        mRemote = remote;
      }
      @Override public android.os.IBinder asBinder()
      {
        return mRemote;
      }
      public java.lang.String getInterfaceDescriptor()
      {
        return DESCRIPTOR;
      }
      @Override public void Wow(int[] test) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          _data.writeIntArray(test);
          boolean _status = mRemote.transact(Stub.TRANSACTION_Wow, _data, _reply, 0);
          _reply.readException();
          _reply.readIntArray(test);
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
      }
    }
    static final int TRANSACTION_Wow = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
  }
  public static final java.lang.String DESCRIPTOR = "com.example.aidlapplication.ui.theme.IRemoteService";
  public void Wow(int[] test) throws android.os.RemoteException;
}
```
这些代码，理论上来讲，我们都可以自己写，即完全抛弃AIDL，无论客户端或者服务端，我们都自己实现所有的传输逻辑，但是你可以看到，这么简单的一个方法就需要我们写这么多的代码，尤其传输数据的部分，我们需要使用`Parcel`去顺序的进行读写，如果手写这部分代码，就十分容易弄错，而且还要服务端和客户端进行协同，如果我们有很多的Service，这些Service需要暴露更多的方法给客户端，那么我们需要手写的代码将会增加多少可想而知，而且还要通知客户端及时更新对应的逻辑，这样就十分复杂。

但是通过AIDL，我们可以简化这些工作，在Android源码中，我们可以看到大量的AIDL文件，可以说AIDL大大降低了应用层使用Binder通讯的重复和复杂性。

# AIDL模板代码如何被生成
在Gradle编译过程中，实际上是使用了Android SDK提供的aidl工具去根据aidl文件去生成模板代码的，打开`/Sdk/build-tools/34.0.0/`，可以看到aidl程序:

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84cf1ed04989436bb5bc9536fcb67e2b~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=673&h=490&s=48023&e=png&b=fcfcfc)

运行此程序，可以看到，aidl不光可以生成java代码，C++甚至Rust代码都可以生成:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1892ea0a853d43019a33cd07ce44a592~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=694&h=437&s=53687&e=png&b=300a25)

# in, out, inout和oneway关键字
在AIDL中，我们可以使用`oneway`，`in`，`out`，`inout`等关键字来修饰我们的方法或参数，下面我将讲解这些参数代表着什么。

## in,out,inout
当我们编写AIDL文件时，如果方法的参数为非基础类型参数，那么就会提示我们要标记参数的类型，否则会报错，比如下面的AIDL:
```Java
interface IRemoteService {
    void Wow(int[] test);
}
```
编译的时候会提示错误:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/305b2c55b49f4ef6af64eb1f12645b85~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=913&h=219&s=7517&e=png&b=ffffff)

当我们把参数用`in`，`out`或者`inout`标记以后，就不会报错了:
```Java
interface IRemoteService {
    void Wow(inout int[] test);
}
```

那么`in`，`out`，`inout`实际指代什么呢? 它其实代表着我们的数据流向。

`in`表示数据只能由客户端流向服务端，用`in`标识的参数，当服务端修改此参数时，客户端读取到的参数仍然是原来的值，不会被修改。

`out`表示数据只能由服务端流向客户端，用`out`标识的参数，服务端接到的参数会是空对象，而服务端修改此参数的时候，客户端能读取到修改后的值。

`inout`则表示数据会双向流动，即客户端和服务端对参数的修改，彼方都可以接收到。

## oneway
我们可以使用`oneway`来修饰我们的`Wow`方法，**注意，由`oneway`修饰的方法，非基础类型参数必须标记为`in`**:
```Kotlin
interface IRemoteService {
    oneway void Wow(in int[] test);
}
```

然后当我们的客户端在调用`Wow`时，就不会等待服务端的处理，会立刻进行下一步，不再等待服务端的处理。

# 为什么需要这些关键字
现在，我们理解了这些关键字的作用，那么问题来了，为什么需要这些关键字? 比如方法为什么需要设置为`oneway`，为什么参数不全设置成`inout`。

要理解这个问题，首先就要理解，Binder通讯的本质是什么，Binder通讯的本质，就是基于Binder驱动的一种进程间通讯，也就是说，我们最终数据的传递是需要Native层控制Binder驱动替我们去处理的，那么这些关键字最终影响的，其实就是我们调用Binder驱动传递数据的方式和逻辑，比如打开`/frameworks/native/libs/binder/IPCThreadState.cpp`，找到`transact`方法，就可以看到在调用写入数据`writeTransactionData`之后，如果被标记为`oneway`，那么就会向`waitForResponse`传入空的参数，在`waitForResponse`中其实就可以看到，在数据传输之后就会直接退出循环，在Java层的表现就会变成调用被`oneway`标识的方法之后，没有阻塞立刻执行下一步:

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/caeaa28fe85349e5b46e753da893058d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=908&h=725&s=82092&e=png&b=ffffff)


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/61d6490970ab413b93843927de1a15ca~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=518&h=85&s=7940&e=png&b=ffffff)

# 我们该什么时候使用这些关键字
使用这些关键字的自由度，其实是比较高的，但是秉承的原则应该是非必须原则，比如客户端如果想控制一些服务端的状态，比如`setProperty1(boolean)`，那么这种方法就可以标记为`oneway`，因为这类方法一般不需要服务端的返回，同样的，有一些参数是客户端希望服务端去设置并读取的，那么就可以标记为`out`，总之在应用层，应该考虑尽量减少数据传输的负担。