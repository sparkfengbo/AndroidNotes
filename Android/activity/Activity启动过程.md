
## 7.Activity启动过程
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
  
  
  
  
  
    
