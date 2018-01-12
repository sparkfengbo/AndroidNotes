[TOC]

##1.为什么会反编译

[读懂 Android 中的代码混淆](http://droidyue.com/blog/2016/07/10/understanding-android-obfuscated-code-by-proguard/)


##2.如何反编译
####1.所需工具
apktool、dex2jar、jd-gui

[下载地址](http://download.csdn.net/download/hanhailong726188/8323371)

####2.Apktool

运行命令：
`fengbo$ ./apktool d /Users/abcd.apk`
会出现名字是abcd的文件夹，文件夹中出现以下文件和文件夹

```
- smali 		（源代码smali格式源码）
- res			（资源文件，包含String、Drawable等）
- original		（CERT.SF、CERT.RSA、AndroidManifest.xml等，这里的AxxMxx是二进制文件）
- lib			（lib库）
- assets
- apktool.yml
- AndroidManifest.xml
```

注意，apk直接解压后的AndroidManifest.xml是二进制文件，不能直接看，需要Apktool才能看到内容。

如果出现下面的异常

```
Exception in thread "main" brut.androlib.AndrolibException: Could not decode ars
c file
```

产生原因：apktool.jar的版本太低，如果使用高版本不会出现异常http://blog.csdn.net/yaya1943/article/details/50542627

[更新地址](https://ibotpeaches.github.io/Apktool/)

####3.dex2jar和jd-gui
使用Apktool工具反编译得到的只有smali格式的代码，如果不了解smali格式，很难读懂源码。
直接将apk解压后会得到`classes.dex`的文件

执行命令`sh dex2jar.sh classes.dex` 会得到 **classes_dex2jar.jar**的文件。

使用jd-gui工具就可以打开**classes_dex2jar.jar**看到源码了。


[dex2jar下载地址](https://sourceforge.net/projects/dex2jar/)
[jd-gui下载地址](http://jd.benow.ca/)

##3.smali语法分析

>Android虚拟机Dalvik并不是执行java虚拟机JVM编译后生成的class文件，而是执行再重新整合打包后生成的dex文件，dex文件反编译之后就是smali代码，可以说，smali语言是Dalvik的反汇编语言