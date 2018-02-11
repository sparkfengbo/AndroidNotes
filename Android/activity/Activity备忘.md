[TOC]

##1.生命周期遗忘点

1.从整个生命周期来说，onCreate和onDestroy是配对的，标志Activity的创建和销毁，并且只有一次调用，onStart和onStop是配对的，标志Activity是否可见，onResume和onPause是配对的，标志Activity是否在前台

2.onStart和onResume都表示Activity可见，但onStart时Activity还在后台，onResume时才到前台

3.onPause中可以执行一些存储数据、停止动画等工作，但是不能太耗时，否则会影响新Activity的展示，因为只有前Activity的onPause执行完，新Activity的onResume才会执行 
eg:  (A)onPause -> (B)onCreate -> (B)onStart-> (B)onResume->(A)onStop


##2.异常声明周期遗忘点


##3.启动模式遗忘点


##4.任务与任务栈

##5.IPC等

##6.某些属性
###6.1 android:exported
Android四大组件都有这个属性，AndroidManifest中如果包含有intent-filter 默认值为true; 没有intent-filter默认值为false。

Activity
在Activity中该属性用来标示：当前Activity是否可以被另一个Application的组件启动：true允许被启动；false不允许被启动。

如果被设置为了false，那么这个Activity将只会被当前Application或者拥有同样user ID的Application的组件调用。

###6.1.1
**eg 1**

App A

```
<activity
		 android:name=".ExportedActivity"
		 android:label="@string/title_activity_main2"
		 android:exported="true" >
            <intent-filter>
                <action android:name="com.fengbo.export"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
</activity>

```

AppB

```
Intent intent = new Intent();
intent.setAction("com.fengbo.export");
startActivityForResult(intent, 0);
//这里能够正常启动，并且ExportedActivity和AppB的Activity在一个任务栈中
```

**eg 2**

App A

```
<activity
		 android:name=".ExportedActivity"
		 android:label="@string/title_activity_main2"
		 android:exported="true" 
		 android:permission="com.sparkfengbo.test">
            <intent-filter>
                <action android:name="com.fengbo.export"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
</activity>

```
此时 会提示

```
E/AndroidRuntime: FATAL EXCEPTION: main
                                                                                           Process: com.sparkfengbo.app.exporttestapplication, PID: 18945
                                                                                           java.lang.SecurityException: Permission Denial: starting Intent { act=com.fengbo.export cmp=com.sparkfengbo.app.javabcsxtest/.ExportedActivity } from ProcessRecord{cc2edf0 18945:com.sparkfengbo.app.exporttestapplication/u0a753} (pid=18945, uid=10753) requires com.sparkfengbo.test
                                                                                           ...
                                                                                               
```
说明权限不够 （**这里如果 APP A 和App B 的sharedUID相同就不会有提示说权限不够的情况**）

**eg 3**

App A

```
<activity
		 android:name=".ExportedActivity"
		 android:label="@string/title_activity_main2"
		 android:exported="true" 
		 android:permission="com.sparkfengbo.test">
            <intent-filter>
                <action android:name="com.fengbo.export"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
</activity>

```

AppB

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.sparkfengbo.app.exporttestapplication">

    <uses-permission android:name="com.sparkfengbo.test" />
</manifest>
```

此时能够正常启动

**最后一个例子**
上述的方法显然是很不安全的，因为AndroidManifest是很容易就反编译的。

所以在App A中的AndroidManifest中添加权限, protectionLevel为signature。此时的结果是App B的签名必须和App A使用的签名完全相同才可以吊起App A的Activity。（不添加签名也不可以的）

```
<permission android:name="com.sparkfengbo.test" android:protectionLevel="signature"/>
```
###6.1.2
> - Activity 在Activity中该属性用来标示：当前Activity是否可以被另一个Application的组件启动：true允许被启动；false不允许被启动。
> - Service 该属性用来标示，其它应用的组件是否可以唤醒service或者和这个service进行交互：true可以，false不可以。当Android sdk 的最小版本为16或者更低时他的默认值是true。如果是17和以上的版本默认值是false。
> - Provider 当前内容提供者是否会被其它应用使用
> - Receiver 当前broadcast Receiver 是否可以从当前应用外部获取Receiver message 。true，可以；false 不可以。不只有这个属性可以指定broadcast Receiver 是否暴露给其它应用。你也可以使用permission来限制外部应用给他发送消息。

###6.2 android:shareUID


> Android系统会为每个应用分配一个唯一的UID，具有相同UID的应用才能共享数据。
> 如果有相同的ShareUID和相同的签名，两个应用能够互相访问对方的私有数据。
> 

  在Android里面每个app都有一个唯一的UID，并且为它创建一个沙箱，以防止影响其他应用程序。UID 在应用程序安装到设备中时被分配，并且在这个设备中保持它的永久性。其manifest中的userid就是对应一个Linux用户(Android 系统是基于Linux)的.
所以不同APK(用户)间互相访问数据默认是禁止的.
  这样权限就被设置成该应用程序的文件只对该用户可见，只对该应用程序自身可见，而我们可以使他们对其他的应用程序可见，这会使我们用到shareUID，也就是让两个apk使用相同的userID，这样它们就可以看到对方的文件。为了节省资源，具有相同ID的apk也可以在相同的linux进程中进行(注意，并不是一定要在一个进程里面运行)，共享一个虚拟机。
  例如，大家都知道app中SharedPreferences文件是保存在`/data/data/<package name>/shared_prefs`的xml文件，当SharedPreferences的权限设置为`Context.MODE_PRIVATE`时，文件的内容只有该app可见，但是可以看到有这样一句注释
  
  ```
   /**
     * File creation mode: the default mode, where the created file can only
     * be accessed by the calling application (or all applications sharing the
     * same user ID).
     */
    public static final int MODE_PRIVATE = 0x0000;
  ```
  也就是说具有相同的shareUID是可以共享数据的。
  
  下面简要介绍一下写的demo
  
  **App A**
  
  **AndroidManifest.xml**
  
  ```
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.sparkfengbo.app.javabcsxtest"
          android:sharedUserId="android.fengbo">  
  ```
  
  **MainActivity**
  
 
  ```
   SharedPreferences sharedPreferences = this.getSharedPreferences("setting", MODE_PRIVATE);
   SharedPreferences.Editor editor = sharedPreferences.edit();
   editor.putString("fengbo", "Hello fengbo");
   editor.commit();
  ```
  
  **App B**
  
  **AndroidManifest.xml**
  
  ```
  <manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.sparkfengbo.app.exporttestapplication"
          android:sharedUserId="android.fengbo">
</manifest> 
  ```
  
  **MainActivity**
  
 
  ```
   try {
            File xmlFile = new File("/data/data/com.sparkfengbo.app.javabcsxtest/shared_prefs/setting.xml");
            if(xmlFile.exists()) {
                //看log证明存在，如果去掉sharedUID，文件不存在，因为没权限读。可以通过此file读取数据
                Log.e("fengbo", "ex");
            }

            Context friendContext = createPackageContext(
                    "com.sparkfengbo.app.javabcsxtest",
                    Context.CONTEXT_IGNORE_SECURITY);
            SharedPreferences sharedPreferences = friendContext.getSharedPreferences("setting", MODE_PRIVATE);
            String testStr = sharedPreferences.getString("fengbo", "null");
            Log.e("fengbo", testStr);
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        } 

  ```
  
  这样AppB能够得到AppA在SharedPreferences中存储的字符串。

  
 可参考文章
 
 - [Android-sharedUserId数据权限 ](http://wallage.blog.163.com/blog/static/17389624201011010539408/)  
 - [Android权限之sharedUserId、签名（实例说明）](http://blog.csdn.net/yuanlong_zheng/article/details/7338276)  参考例子

##7.Activity启动过程
1.根Activity的启动过程
MainActivity由Launcher启动，Launcher通过AMS启动MainActivity，MainActivity、Launcher、AMS运行在不同的进程。

过程1：Launcher向AMS发送启动MainActivity组件的进程间通信请求。

  - Step1：Launcher.StartActivitySafely
  
  >在应用程序启动器Launcher页面点击快捷图标时会调用此方法，MainActivity的信息会包含在Intent中，action是“android.intent.action.MAIN”,category是“android.intent.category.LAUNCHER”,cmp是包名+类名，Intent中的信息是系统启动时，PackageManagerService会对AndroidManifest.xml进行解析，系统启动完成后Launcher组件启动，像PMS查询action是“Intent.ACTION_MAIN”,category是“Intent.CATEGORY_LAUNCHER”的Activity组件，创建快捷图标。Intent额外会添加FLAG_ACTIVITY_NEW_TASK标志位。
  Launcher 继承自Activity
  
  - Step2:Activity.startActivity
  - Step3:Activity.startActivityForResult
  
  ```
  Instrumentation.ActivityResult ar =
       mInstrumentation.execStartActivity(this,    
       mMainThread.getApplicationThread(), 
       mToken, this, intent, requestCode, options);
  ```
  mInstrumentation用来监控应用程序和系统之间的交互操作，由于启动Acitivity是应用程序和系统之间的交互，所以在execStartActivity执行，以便监控这个过程。
  mMainThread类型是ActivityThread（ActivityThread用来描述一个应用程序进程，用来管理执行应用进程主线程的Activity、Broadcast的执行），getApplicationThread返回ActivityThread内部的ApplicationThread的Binder对象。ApplicationThread继承自ApplicationThreadNative，ApplicationThreadNative是抽象类，继承自Binder。
  mToken是IBinder对象，在attach方法中赋值。指向AMS中类型为ActivityRecord的Binder本地对象。
  
  - Step4：Instrumentation.execStartActivity
  
  ```
  int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, null, options);
  ```
  
  ActivityManagerNative.getDefault()获得ActivityManagerService的代理对象
  
  ```
  private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
  ```
  
 - Step5:ActivityManagerProxy.startActivity
  `mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);`
  这里Parcel会保存一些参数，caller指向Launcher的ApplicationThread对象，intent保存MainActivity的信息，resultTo指向AMS内部的ActivityRecord对象，保存了Launcher组件的详细信息。
  以上在Launcher中执行的。
  
  过程2：AMS首先将要启动的MainActivity组件的信息保存，再向Launcher组件发送进入终止状态的进程间通信请求。
  
 - Step6:ActivityManagerService.startActivity
 - Step7:ActivityStack.startActivityMayWait
   通过PackageMangerService解析intent内容，保存在ActivityInfo中
 - Step8:ActivityStack.startActivityLocked
  通过caller查找ProcessRecord信息，对应的是Launcher所在进程的信息，通过resultTo（mToken）查找ActivityRecord信息，
  
  **至此，ActivityStack类得到了请求AMS将启动的Activity组件的信息和源Activity组件的信息。**
  
 - Step9:ActivityStack.startActivityUncheckedLocked
 这里创建一个新的TaskRecord，即新的任务栈，这个TaskRecord保存在ActivityRecord的成员变量里，这个TaskRecord会交给AMS处理。在startActivtyLocked中，将ActivityRecord放在历史栈的队首。
 - Step10:ActivityStack.resumeTopActivityLocked
 - Step11:ActivityStack.startPausingLocked
 ```
 prev.app.thread.schedulePauseActivity(prev.appToken, prev.finishing,
                        userLeaving, prev.configChangeFlags);
 ```
  向Launcher所在的进程发送一个中止Launcher的通知，以便Launcher执行数据保存等操作。
  ```
  Message msg = mHandler.obtainMessage(PAUSE_TIMEOUT_MSG);
            msg.obj = prev;
            prev.pauseTime = SystemClock.uptimeMillis();
            mHandler.sendMessageDelayed(msg, PAUSE_TIMEOUT);
  ```
  同时发送一个消息，如果Launcher执行中止超时，做一些处理，防止一直等待。
  
- Step12:ApplicationThreadNative.java
  以上7步都是在AMS中执行的，接下来13-17是在Launcher中执行的
- Step13:ApplicationThread.schedulePauseActivity
- Step14:ActivityThread.queueOrSendMessage
- Step15:H.handleMessage
- Step16:ActivityThread.handlePauseActivity
  应用进程中启动每一个Activity组件都使用一个ActivityClientRecord对象描述，和AMS中的ActivityRecord对象对应。handlePauseActivity会调用performPauseActivity方法，调用Launcher的onPause方法
- Step17:ActivityManagerProxy.activityPaused
  发送ACTIVITY_PAUSED_TRANSACTION的请求

  过程3：Launcher组件进入中止状态后，就会向AMS发送一个已经进入中止状态的进程间通信请求，以便AMS继续执行启动MainActivity组件的操作。
  
- Step18:ActivityManagerService.activityPaused
- Step19:ActivityStack.activityPaused
  移除了PAUSE_TIMEOUT_MSG消息，将Launcher对应的ActivityRecord对象的state设置为ActivityState.PAUSED。
- Step20:ActivityStack.completePauseLocked
- Step21:ActivityStack.resumeTopActivityLocked
  在Step10 中调用了resumeTopActivityLocked，但那时mResumedActiviy不等于null，Launcher尚未进入Paused状态，此时进入Paused状态后，调用startSpecificActivityLocked方法。
- Step22:ActivityStack.startSpecificActivityLocked
  每个Activity都有一个UID和PID，其中用户ID是安装该Activity组件PMS分配的，进程名称是由android：process属性决定的。
  **AMS在启动Activity时会检查系统中是否存在一个对应的应用程序进程，如果存在，则直接通知该进程将Activity组件启动起来，否则，先将这个用户ID和进程名称来创建一个应用程序进程，然后再通知启动activity**
  过程4：AMS发现用来运行MainActivity组件的应用程序不存在，因此创建一个新的应用程序进程。
- Step23:ActivityManagerService.startProcessLocked
- Step24:ActivityThread.main
这个方法很重要，这里会调用Looper.prepareMainLooper。这里还会创建ApplicationThread，是一个Binder对象，AMS通过ApplicationThread进行通信
- Step25:ActivityManagerproxy.attachApplication
发送ATTACH_APPLICATION_TRANSACTION通信请求
- Step26:ActivityManagerService.attachApplication
- Step27:ActivityManagerService.attachApplicationLocked
删除了PROC_START_TIMEOUT_MSG消息
- Step28:ActivityStack.realStartActivityLocked
- Step29:ApplicationThreadProxy.scheduleLaunchActivity
  发送SCHEDULE_LAUNCH_ACTIVITY_TRANSACTION的请求，以上4步在AMS中执行
  
- Step30:ApplicationThread.scheduleLaunchActivity  
- Step31:ActivityThread.queueOrSendMessage
处理类型是LAUNCH_ACTIVITY的消息  
- Step32:H.handleMessage
 ```
 ActivityClientRecord r = (ActivityClientRecord)msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);
 ```
 查找LoadedApk
- Step33:ActivityThread.handleLaunchActivity  
- Step34:ActivityThread.performLaunchActivity  
- Step35:MainActivity.onCreate    
  
  
  
  
  
    
