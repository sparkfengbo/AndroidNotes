[TOC]

**文章结构**

- 1.使用多进程原因
- 2.如何开启多进程
- 3.多进程可能会引起的问题
- 4.序列化
- 5.常用的IPC通信机制（主要介绍AIDL，附带介绍一下匿名共享内存）

	- 5.4 AIDL
	- 5.7 匿名共享内存
	
- 6.Binder的基本原理



可参考文章

- [Android深入浅出之Binder机制](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html)
- [Android Binder设计与实现 - 设计篇](http://blog.csdn.net/universus/article/details/6211589)
- [Android进程间通信（IPC）机制Binder简要介绍和学习计划](https://blog.csdn.net/luoshengyang/article/details/6618363)
- [Binder学习指南](http://weishu.me/2016/01/12/binder-index-for-newer/)


------

## 1.使用多进程原因

- 一种是由于某些模块的特殊原因需要运行在单独进程，或为了增大一个应用可使用的最大内存需要通过多进程获取多份内存空间。
- 另外一种是当前应用需要向其他应用获取数据。

>系统为每个进程分配的内存是有限的，比如在以前的低端手机上常见是 16M，现在的机器内存更大一些，32M、48M，甚至更高。但是，总是有限的，毕竟一个手机出厂之后 RAM 的大小就定了，总是无法满足所有应用的需求。

>也可以在 Application 中通过使用 largeHeap 属性为自己应用所处的进程争取分配更大的内存，就像这样,但也存在不够用的可能。

```
<application
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:largeHeap="true">
    
    ......
    
</application>
```

此部分可参考《Android开发艺术探索》或[Android 中的多进程，你值得了解的一些知识](https://yifeng.studio/2017/06/16/android-multi-process-things/)


## 2.开启多进程

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.sparkfengbo.app" >
	<service
		android:name=".android.aidltest.AService"
		android:process=":remote"/>
	<service
		android:name=".android.aidltest.BService"
		android:process="com.sparkfengbo.test.hh"/>       
	...         
```

默认进程的进程名是包名.

AService所在进程是`com.sparkfengbo.app:remote`

BService所在进程是`com.sparkfengbo.test.hh`

可以用过 `asb shell ps | grep com.sparkfengbo`的指令查看进程。

**进程名以`:`开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中，而进程名不以`:`开头的进程属于全局进程，其他应用可以通过相同的ShareUID和相同的签名和它跑在同一个进程。**

**如果有相同的ShareUID和相同的签名，两个应用能够互相访问对方的私有数据。**

## 3.多进程可能会引起的问题

- 1.静态成员和单例模式失效
- 2.线程同步机制失效
- 3.SharedPreference可靠性下降
- 4.Application会多次重建

1、2是因为内存空间不一样了，锁或者全局类都不能保证线程同步，不同进程锁的不是同一个对象;3是因为SharedPreference不支持两个进程同时去进行写操作（底层读写XML，容易出现问题）;4是因为当一个组件跑在一个新进程中时，由于系统要创建新的进程同时分配独立的虚拟机，所以这个过程实际就是启动一个应用的过程。

>Android应用为每个应用分配独立的虚拟机，或者为每个进程都分配独立的虚拟机，这样不同进程的组件会拥有独立的虚拟机、Application和内存空间

## 4.序列化

参考《序列化备忘》笔记。

## 5.常用的IPC通信机制

Android是基于Linux的。

Linux提供的IPC机制有 

- 1.管道（Pipe） 
- 2.信号（Signal） 
- 3.消息队列（Message）
- 4.共享内存（Share Memory） 
- 5.插口（Socket）


*下面是Android能够进行IPC的方法的罗列，字号较大的是比较好的实现方式*

###### 5.1 Bundle

**同一应用内**启动Service时，通过Bundle将信息包在Intent中

###### 5.2 使用文件共享

### 5.3 使用Messenger

参考实例代码 [AndroidCodeDemoTest](https://github.com/sparkfengbo/AndroidCodeDemoTest) 中的MessengerTestActivity

大致的结构是Messenger + Handler

**Messenger是以串行的方式处理消息，如果大量的消息同时发送到服务端，服务端仍然只能一个个处理，如果有大量的并发请求，那么用Messenger就不太合适。**

**Messenger的作用主要是传递消息,如果需要跨进程调用服务端的方法，Messenger就不行了。**

底层其实和AIDL是一个原理，还是看看接下来看看AIDL的实现吧。


### 5.4 使用AIDL

参考实例代码 [AndroidCodeDemoTest](https://github.com/sparkfengbo/AndroidCodeDemoTest) 中的AIDLTestActivity

#### 5.4.1 需要注意的是

**1.** 当客户端发起远程请求时，客户端线程被挂起，直至服务端进程返回数据，而服务端被调用的方法运行在服务端的Binder线程池中。所以如果一个远程方法很耗时，那么不能在UI线层发起远程调用。 

**另外，由于服务端的方法运行在Binder线程池中，切记不要在服务端方法中开异步线程。**处理客户端的回调方法也是同样的道理（Server的远程方法运行在Server的Binder线程池中，客户端的被Server调用的回调方法运行在客户端的Binder线程池中）（RemoteCallback相关的代码）。

**2.** 由于服务端的Binder方法运行在Binder线程池中，所以Binder方法不管是否耗时都应该采用同步的方法去实现，因为它已经运行在一个线程中了。

**3.** AIDL中不是所有数据类型都可以使用，能够使用的数据类型有：`基本数据类型`、`String、CharSequence`、`ArrayList`、`HashMap`、`Parcelable`（必须显式的import进来，如果用到自定义的Parcelable则必须新建同名的aidl文件，并声明类型）、`AIDL`。

Parcelable（必须显式的import进来，如果用到自定义的Parcelable则必须新建同名的aidl文件，并声明类型）这里可以参考 [AndroidCodeDemoTest](https://github.com/sparkfengbo/AndroidCodeDemoTest)

AIDL中除了基本类型，其他类型必须标上in、out、inout（不能随便使用，因为在底层实现中有开销）

**4.** AIDL接口中只支持方法，不支持静态变量

**5.** AIDL的包结构在服务端和客户端中必须保持一致，因为客户端需要反序列化服务端中和AIDL接口相关的所有类。建议把AIDL相关的类和文件全部放在同一个包中

**6.** Server可以连接很多个Client

**7.** Server的远程方法运行在Server的Binder线程池中，客户端的被Server调用的回调方法运行在客户端的Binder线程池中。

>为什么可以使用CopyOnWriteArrayList和ConcurrentHashMap？AIDL中支持ArrayList和HashMap，但是AIDL所支持的是List接口，虽然服务端返回的是CopyOnWriteArrayList，但是Binder中会按照List的规范访问数据并最终形成一个ArrayList传递给客户端。

**建议看看AIDL生成的代码**

#### 5.4.2 接口运行在Server的Binder的死亡通知

**方法1**

```
    private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
        @Override
        public void binderDied() {
            if (iBookManagerInterface == null) {
                return;
            }

            TLog.e("DeathRecipient binderDied");
            iBookManagerInterface.asBinder().unlinkToDeath(mDeathRecipient, 0);
            iBookManagerInterface = null;

            tryBindService();
        }
    };
    
    
     private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            TLog.e("service connencted " + name.getPackageName());

            iBookManagerInterface = IBookManagerInterface.Stub.asInterface(service);

            try {
                service.linkToDeath(mDeathRecipient, 0);
                
        ...
```

**方法2**

onServiceDisconnected


**区别：**

onServiceDisconnected在UI线程调用而binderDied在客户端的线程池中调用。


#### 5.4.3 有关AIDL中使用接口类型

**多进程的回调问题**。请参考《Android开发艺术探索》 第2章79-87页，**需要重点关注 RemoteCallback**

1.声明接口，并创建AIDL文件

 ```
 xxxListener.aidl
 
 interface xxxListener {
 	//Method
 }
 
 IBookManager.aidl
 
 void registerListenr(xxxListener listener) ;
 
 ```
 
2.Client端

```
ServiceConnection/onServiceConnected

{
	IBookManager.registerListener(mLisner);
}

onDestory

{
	unregisterListener();
}

```

 
3.Server端

```
private RemoteCallbackList<xxxListener> mListener = new  RemoteCallbackList<xxxListener> ;

mListener.beginBroadcast();

mListener.getBroadcastItem(i);

mListener.finishBroadcast();
```


#### 5.4.5 安全性：验证权限

默认情况下，远程服务任何人都可以连接，所以必须进行权限的验证。这部分《Android开发艺术探索》 第2章 90页


###### 5.5 使用ContentProvider

底层的实现也是Binder，可以参考《Android开发艺术探索》 第2章91页


###### 5.6 使用Socket	
				
参考《Android开发艺术探索》 第2章103页，此处不赘述了


### 5.7 匿名共享内存

此部分参考《Android系统源代码情景分析》（罗升阳）记录。[博客地址](https://blog.csdn.net/luoshengyang/article/details/6651971)



>在Android系统中，提供了独特的匿名共享内存子系统Ashmem（Anonymous Shared Memory），它以驱动程序的形式实现在内核空间中。它有两个特点，一是能够辅助内存管理系统来有效地管理不再使用的内存块，二是它通过Binder进程间通信机制来实现进程间的内存共享。

在Android应用程序框架层，提供了一个MemoryFile接口来封装了匿名共享内存文件的创建和使用，它实现在frameworks/base/core/java/android/os/MemoryFile.java文件中。

详细实例请参考实例代码 [AndroidCodeDemoTest](https://github.com/sparkfengbo/AndroidCodeDemoTest) 中的AshmemActivty。

- [Android系统匿名共享内存Ashmem（Anonymous Shared Memory）在进程间共享的原理分析](https://blog.csdn.net/luoshengyang/article/details/6666491)


## 6.Binder基本原理

此部分参考《Android系统源代码情景分析》（罗升阳）记录。


### 6.1 Binder的优势

Android是基于Linux的，但是为什么需要新设计一个通信机制，Binder？

**Binder的特点和优势**
>传统的跨进程通信机制如Socket，开销大且效率不高，而管道和队列拷贝次数多，更重要的是对于移动设备来说，安全性非常重要，传统的通信机制安全性低，大部分情况下接受方无法得到发送方进程的可信PID/UID，难以甄别身份。
>
>Binder只需要进行一次拷贝操作。
>
>Binder是在OpenBinder的基础上实现的。
>

### 6.2 Binder通信过程
 

**涉及的对象**
Client进程和Server进程的一次通信涉及四种类型对象，分别是

- 位于Binder驱动程序中的Binder实体对象（binder_node）
- 位于Binder驱动程序中的Binder引用对象（binder_ref）
- Binder库中的Binder本地对象（BBinder）
- Binder库中的Binder代理对象（BpBinder）


![过程](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/binder.JPG?raw=true)

**过程**

- 1.运行在Client进程中的Binder代理对象通过Binder驱动程序向运行在Server进程中的Binder本地对象发出进程间通信请求，Binder驱动程序接着就根据Client进程传递来的Binder代理对象的句柄值找到对应的Binder引用对象
- 2.Binder驱动程序根据前面找到的Binder引用对象找到对应的Binder实体对象，并创建一个事物（binder_transaction）来描述传递过来的通信数据发送给它处理
- 3.Binder驱动程序根据前面找到的Binder实体对象来找到运行在Server进程中的Binder本地对象，并且将Client进程传递过来的通信数据发送给它处理
- 4.Binder本地对象处理完成Client进程的通信请求后就将通信结果返回给Binder驱动程序，Binder驱动程序接着找到前面所创建的一个事物
- 5.Binder驱动程序根据前面找到的事物的相关属性来找到发出通信请求的Client进程，并且通知Client进程将通信结果返回给对应的Binder代理对象处理


可以看到涉及的四种对象类型都有相互依赖的关系，所以为了维护这些Binder对象的依赖关系，Binder通信机制采用**引用计数技术**维护每一个Binder对象声明周期。

使用Binder在进程间传递数据的时候，有时候会抛出TransactionTooLargeException这个异常，这个异常的产生是因为Binder驱动对内存的限制引起的。也就是说，我们不能通过Binder传递太大的数据。官方文档里有说明，最大通常限制为1M。参见[TransactionTooLargeException](https://developer.android.com/reference/android/os/TransactionTooLargeException.html)。可以通过MemoryFile解决，参考[Android匿名共享内存和MemoryFile](https://blog.csdn.net/goodlixueyong/article/details/53151959)

---------------------------------------------

**下面是未整理的部分**
(**TODO** 底层原理待补充，太复杂了  )

1.Binder进程间通信机制框架

2.相关的知识路线

- Binder驱动程序

  - Binder的驱动程序实现在内核空间中，目录接口是

	```
~/Android/kernel/goldfish
----drivers
	----staging
		----android
			----binder.h
			----binder.c
	```

	- 设备文件 /dev/binder的打开（open）、内存映射（mmap）、内核缓冲区管理等

- Service Manager的启动过程

- Service Manager代理对象的获取

- Service组件的启动过程

- Service代理对象的获取过程

- Binder进程间通信机制的Java接口

