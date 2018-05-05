**阅读本篇文章，你会了解到**

- JNI的概念及作用
- JNI设计概述，包括

  - JNIEnv的作用
  - Native方法命名规则
  - JNI是如何找到Native方法并调用的

## 一、JNI概览

JNI全称为Java Native Interface， 它是一个本地编程接口。它允许跑在Java虚拟机上的Java代码和用其他语言，如C、C++或汇编等写的库去互通。

**什么时候会需要JNI？**

- 当你的应用需要某些依赖平台特性的功能，而Java类库并不支持。
- 你已经用其他语言写了一个库，希望通过JNI能使Java代码能够使用这些库
- 你希望使用像汇编这样的底层语言去实现对运行时间要求高的代码

>Java是跑在虚拟机上的，它的运行速度并不如C、C++、汇编快，并且不同的虚拟机的特性也不相同，某些依赖虚拟机所运行平台的特性还是需要JNI去实现，并且像直播、视频这样的应用，也会用到大量的编解码的库，这些就需要通过JNI去实现。

**JNI能做什么？**

你可以通过native方法去：

- 创建、审查、更新Java对象
- 调用Java方法
- 捕获抛出异常
- 加载类，获取类的信息
- 执行运行时类型检查


如果你还想更详细的了解JNI的发展历史背景，可以参考JNI官方文档[Chapter 1 Introduction](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/intro.html#wp9502)



## 二、JNI设计概述
如果你接触过JNI，相信你一定见过类似下面的这段代码：

函数的原型在`com.sparkfengbo.app.androidexample.jnitest.JniTestActivity`中声明

```
private native Father jniCallNative(int i, String name, Father obj);
```

JNI的函数为：

```
JNIEXPORT jobject JNICALL Java_com_sparkfengbo_app_androidexample_jnitest_JniTestActivity_jniCallNative__ILjava_lang_String_2Lcom_sparkfengbo_app_androidexample_jnitest_Father_2
(JNIEnv *env, jobject thiz, jint i, jstring str, jobject father)
{
  jint javaFatherNum = (*env)->GetIntField(env, father, gFatherJavaField.numId);
}
```

很多初学者看不明白这样的方法，连函数名看的都一头雾水，很多人因为类似这样的原因望而却步，阻碍了前进的道路。先从这个方法一步步讲起，请往下看。

### 1. 数据类型

像`jstring`，`jint`都是在<jni.h>定义的数据类型，和Java相似。

### 2. JNIEXPORT、JNICALL

JNIEXPORT是什么？JNICALL是什么？

>JNIEXPORT、JNICALL是在<jni.h>中进行宏定义的，类似这样很多的内容都可以在`/Android/sdk/ndk-bundle/platforms/android-14/arch-arm/usr/include/`,前提是你安装了ndk。

```
#define JNIIMPORT
#define JNIEXPORT  __attribute__ ((visibility ("default")))
#define JNICALL
```

### 3. native方法名命名规则

一个native方法是下面内容的组合：
- 前缀`Java_`
- 用`_`连接的完全限定类名(`_`代替`/`)
- 方法名
- 重载的本地方法在参数前加两个下划线`__`

因为一个变量或方法的名称或类型标志符不能用数字开头，所以使用`_0`, `_9`代替转义字符。

![](http://upload-images.jianshu.io/upload_images/952890-fc567225aba58d9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



```
jobject Java_com_sparkfengbo_app_androidexample_jnitest_JniTestActivity_jniCallNative__ILjava_lang_String_2Lcom_sparkfengbo_app_androidexample_jnitest_Father_2
```
1.首先，一个Java_前缀

2.后面紧接着声明native方法的类名`com_sparkfengbo_app_androidexample_jnitest_JniTestActivity`
JniTestActivity、Son、Father在工程中的路径是
![](http://upload-images.jianshu.io/upload_images/952890-b388a35fa89cc386.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.紧接着，是native方法的方法名`jniCallNative`
这个native方法在JniTestActivity中是这样声明的，有三个重载方法。
![](http://upload-images.jianshu.io/upload_images/952890-3cf8195ea66e1a33.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4.上面说过，如果是重载方法的话，在参数前面会加两个下划线`__`，你们仔细看一下是不是参数前的下划线要长一些？

5.上面的native方法是针对第三个重载方法，有三个参数，第一个参数int，为`I`(这里如果有疑问，请参考[JNI的数据类型和方法签名](aaa))，第二个参数String为`Ljava_lang_String;`这个分号需要转义，根据上文的表格，分号转义使用`_2`，所以后面又加上了`java_lang_String_2L`，后面的Father参数也是同样的道理
6.返回一个Father对象，对应native方法，返回的是jobject



### 4. native方法参数


#### 4.1 **参数中的JNIEnv**

在使用JNI的时候需要在Java文件中声明一个native方法，对应于C或CPP文件的native方法，只不过在Native方法中的参数会多出两个。

`JNIEnv *env`是一个指针,实际上在官方文档中`JNIEnv *env`是一个叫做JNI interface pointer的东西。在JNI中定义了很多[native方法](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/functions.html#wp9502)，或者称作`interface function`，例如让native调用Java方法的`Call<type>Method`，native更改Java中的对象的`SetObjectField()`。

接口指针(interface pointer)是一个指向指针(暂且称作P)的指针，而P指向一个指针数组，这个数组里的每个指针指向一个接口方法(interface function)，每个接口方法在数组里预定义。如下图。所以通过env就可以调用JNI中定义的native方法。
![](http://upload-images.jianshu.io/upload_images/952890-fb3c48a64d80928e.gif?imageMogr2/auto-orient/strip)

>需要说明的是，接口指针只有在当前的线程是合法的。native方法不能将接口指针从一个线程传到另一个线程。同一个Java线程中执行多次调用时，VM被允许传递同一个interface pointer给native方法。然而，native方法能被不同的Java线程调用，这样可能接收不同的JNI interface pointers。

#### 4.2 **参数中的jobject thiz**

当然了，你的native方法的第二个参数可以自己定义，没必要非要叫做thiz。

这个参数根据native方法是static还是nonstatic有所区别。如果这个native方法不是静态方法，那这个thiz就是对调用native方法的这个Java对象的引用。如果native方法是静态方法，那么thiz就是对Java类的引用。


>VM在so库中的方法查找匹配的方法名称，首先查找short name（不带参数签名的），然后寻找long name（带有参数签名的）。然而，如果本地方法和一个非本地方法有同样的名称。这也不是问题，因为一个非本地方法（Java 方法）不在so库里面。

如果你还想了解更多内容，可以参考JNI官方文档[Chapter2 Design Overview](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/design.html#wp9502)