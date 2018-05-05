JNI的基础知识请参考[这篇文章]()

>此篇主要介绍了JNI中
- 基本类型
- 引用类型
- 重要数据结构
- 类型签名
- 方法签名


####一、基本类型

JNI中数据的基本类型和Java、C语言很相似，包括`jboolean`,`jbyte`,`jchar`,`jlong`等，只不过前面加了一个`j`。实际上Native中的这些类型都会与Java的类型一一映射,参见下面的表格。

|JavaType|NativeType|Description|
|:----|:---|:----|
|boolean|jboolean|unsigned 8 bits|
|byte|jbyte|signed 8 bit|
|char| jchar |unsigned 16 bits|
|short| jshort |signed 16 bits|
|int| jint |signed 36 bits|
|long| jlong |signed 64 bits|
|float| jfloat |32-bit|
|char| jdouble |64-bit|
|void| void |N/A|

实际上它是在哪定义的呢？在你的c或cpp文件中会引用头文件`<jni.h>`，在`<jni.h>`中有这样的定义：
```
typedef unsigned char   jboolean;       /* unsigned 8 bits */
typedef signed char     jbyte;          /* signed 8 bits */
typedef unsigned short  jchar;          /* unsigned 16 bits */
typedef short           jshort;         /* signed 16 bits */
typedef int             jint;           /* signed 32 bits */
typedef long long       jlong;          /* signed 64 bits */
typedef float           jfloat;         /* 32-bit IEEE 754 */
typedef double          jdouble;        /* 64-bit IEEE 754 */
```
>如果你安装了ndk，那么`<jni.h>`在`/Android/sdk/ndk-bundle/platforms/android-14/arch-arm/usr/include/`下，在`arch-arm`下包含了很多需要引用的头文件和so库。

此外，为方便，定义了
```
#define JNI_FALSE 0 
#define JNI_TRUE 1 
```

`jsize`用来描述索引和大小的：`typedef jint jsize; `

--------

####二、引用类型
JNI包含一系列的引用类型，和Java对象对应。如下图所示：
![](http://upload-images.jianshu.io/upload_images/952890-9bc1437a8b8b2735.gif?imageMogr2/auto-orient/strip)

>需要指出的是在C语言中 `jobject `和 `jclass `是对等的:
`typedef jobject jclass;`,
但是在C++中是这样
```
class _jobject {};
class _jclass : public _jobject {}; 
... 
typedef _jobject *jobject; 
typedef _jclass *jclass; 
```

----------

####三、数据结构
**Field and Method IDs**
用来查找类定义中成员的FieldID和方法的MethodID，需要注意ID是大写的。
```
struct _jfieldID;                       /* opaque structure */ 
typedef struct _jfieldID *jfieldID;     /* field IDs */  
struct _jmethodID;                      /* opaque structure */ 
typedef struct _jmethodID *jmethodID;   /* method IDs */ 
```

**The Value Type **
`jvalue`联合体用来在传递参数数组定义元素类型的。`jvalue`联合体定义如下：
```
typedef union jvalue {  
    jboolean      z;  
    jbyte         b;  
    jchar         c;  
    jshort        s;  
    jint          i;  
    jlong         j;  
    jfloat        f;  
    jdouble       d;  
    jobject       l; 
} jvalue; 
```
比如，在调用`Call<type>MethodA`方法时，参数中传递调用方法需要传递的参数。
```
jvalue *args = new jvalue[2];  
args[0].i = 100L;  
args[1].d = 1.56;  
env->CallBooleanMethodA(obj, methodID, args);  
delete [] args;  
```

>`Call<type>MethodA`的声明为
```
NativeType Call<type>MethodA(JNIEnv *env, jobject obj,
jmethodID methodID, const jvalue *args);
```
有关Native方法请参考[这篇文章](aaa)

-----

####四、类型签名
可能有人会有疑问，为什么需要类型签名，类型签名是干什么的？
在我之前写的[例子](aaaa)中，我想要得到Father类的构造方法的methodID。

Father类构造函数如下：
```
public Father(int num, String name, Son son){
        this.num = num;
        this.name = name;
        this.son = son;
}
```
下面的代码中传入获得的Father的jclass对象，传入构造方法的方法名`<init>`，之后传入了这个构造方法的方法签名，因为Java的重载，如果没有方法签名，JavaVM是不知道这个方法名所指定的到底是哪个方法。
```
jmethodID mID = (*env)->GetMethodID(env, fatherClass,"<init>","(ILjava/lang/String;Lcom/sparkfengbo/app/androidexample/jnitest/Son;)V");
```
那这个签名到底有什么规则呢？JNI使用Java VM的类型签名表示方法。

|Type Signature|Java Type|
|:----|:---|
|Z| boolean |
|B| byte |
| C | char |
| S | short |
|I| int |
| J | long |
| F | float |
| D | double |
| J | long |
|L fully-qualified-class ;|fully-qualified-class|
|[ type|type[]|
|( arg-types ) ret-type|method type|

举个例子吧，
如果有一个方法
```
long f (int n, String s, int[] arr); 
```
那对照上表，它的签名是
```
(ILjava/lang/String;[I)J 
```
**说明一下**：
1.签名的前面写参数，记得写上括号
2.参数包含int，对照表为I
3.然后是String，对照表，String是一个类，String的fully-qualified-class是`java/lang/String`，所以后面接`Ljava/lang/String;`，**不要忘了分号**
4.第三个参数是int数组，填入`[I`
5.返回值是long，用`J`代替

再对照一下Father类的构造函数的类型签名，是不是清晰多了？
对于多维数组，他有几维，它的前面就有几个`[`，比如二位int数组，它的签名就是[[I

如果你还想更详细的了解关于Modified UTF-8 Strings的知识，可以参考JNI官方文档[Chapter 3 JNI Types and Data Structures](http://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/types.html#wp9502)