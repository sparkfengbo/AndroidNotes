[TOC]

## 1.生命周期遗忘点

1.从整个生命周期来说，onCreate和onDestroy是配对的，标志Activity的创建和销毁，并且只有一次调用，onStart和onStop是配对的，标志Activity是否可见，onResume和onPause是配对的，标志Activity是否在前台

2.onStart和onResume都表示Activity可见，但onStart时Activity还在后台，onResume时才到前台

3.onPause中可以执行一些存储数据、停止动画等工作，但是不能太耗时，否则会影响新Activity的展示，因为只有前Activity的onPause执行完，新Activity的onResume才会执行 
eg:  (A)onPause -> (B)onCreate -> (B)onStart-> (B)onResume->(A)onStop


## 2.异常生命周期遗忘点

[TODO]

## 3.启动模式遗忘点

[TODO]

## 4.任务与任务栈

[TODO]

## 5.IPC等

[TODO]

## 6.某些属性
### 6.1 android:exported

Android四大组件都有这个属性，AndroidManifest中如果包含有intent-filter 默认值为true; 没有intent-filter默认值为false。

**Activity**


在Activity中该属性用来标示：**当前Activity是否可以被另一个Application的组件启动：true允许被启动；false不允许被启动。**

如果被设置为了false，那么这个Activity将只会被当前Application或者拥有同样user ID(通过ShareUID)的Application的组件调用。

### 6.1.1 一些小例子


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

App B

```
Intent intent = new Intent();
intent.setAction("com.fengbo.export");
startActivityForResult(intent, 0);
//这里能够正常启动，并且**ExportedActivity和AppB的Activity在一个任务栈中**
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
    java.lang.SecurityException: Permission Denial: 
    starting Intent { act=com.fengbo.export
    cmp=com.sparkfengbo.app.javabcsxtest/.ExportedActivity } 
    from ProcessRecord{cc2edf0 18945:
    com.sparkfengbo.app.exporttestapplication/u0a753}
     (pid=18945, uid=10753) requires com.sparkfengbo.test
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
### 6.1.2 exported属性在其他组件的意义


> - Activity 在Activity中该属性用来标示：当前Activity是否可以被另一个Application的组件启动：true允许被启动；false不允许被启动。
> - Service 该属性用来标示，其它应用的组件是否可以唤醒service或者和这个service进行交互：true可以，false不可以。当Android sdk 的最小版本为16或者更低时他的默认值是true。如果是17和以上的版本默认值是false。
> - Provider 当前内容提供者是否会被其它应用使用
> - Receiver 当前broadcast Receiver 是否可以从当前应用外部获取Receiver message 。true，可以；false 不可以。不只有这个属性可以指定broadcast Receiver 是否暴露给其它应用。你也可以使用permission来限制外部应用给他发送消息。

### 6.2 android:shareUID


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
