如果想深入学习音视频开发的话，需要从基础做起，一步一步的建立起自己的音视频开发领域的知识。

最基础的基本知识，需要了解PCM、AAC、YUV、H264，如果你在Android平台下开发的音视频应用的话，需要了解Android NDK开发的基础知识，例如JNI、CMAKE脚本等，当然还需要复习一下c++的知识。此外还需要对ffmepg有一个总体的认识。

通过以上的准备，可以开始准备尝试写一个demo了，内容包括使用ffmepg进行音频编码（PCM-AAC）、视频编码（YUV-H264）、合成（AAC + H264 - MP4）。当完成以上的内容就可以再做一些直播的demo、音频特效的合成、视频特效的合成等内容了。

我们先从基础开始。


## 需要提前准备的知识

### 1.音视频基础

- [关于PCM](https://github.com/sparkfengbo/AndroidNotes/blob/master/%E9%9F%B3%E8%A7%86%E9%A2%91%E5%A4%84%E7%90%86/PCM.md)
- [关于AAC](https://github.com/sparkfengbo/AndroidNotes/blob/master/%E9%9F%B3%E8%A7%86%E9%A2%91%E5%A4%84%E7%90%86/AAC.md)
- [关于YUV](https://github.com/sparkfengbo/AndroidNotes/blob/master/%E9%9F%B3%E8%A7%86%E9%A2%91%E5%A4%84%E7%90%86/YUV.md)
- [关于H264](https://github.com/sparkfengbo/AndroidNotes/blob/master/%E9%9F%B3%E8%A7%86%E9%A2%91%E5%A4%84%E7%90%86/H264.md)


### 2.直播协议相关

- [关于FLV](https://github.com/sparkfengbo/AndroidNotes/blob/master/%E9%9F%B3%E8%A7%86%E9%A2%91%E5%A4%84%E7%90%86/FLV.md)
- [关于RTMP](https://github.com/sparkfengbo/AndroidNotes/blob/master/%E9%9F%B3%E8%A7%86%E9%A2%91%E5%A4%84%E7%90%86/RTMP.md)
- [视频压缩的基本概念](https://github.com/sparkfengbo/AndroidNotes/blob/master/%E9%9F%B3%E8%A7%86%E9%A2%91%E5%A4%84%E7%90%86/%E8%A7%86%E9%A2%91%E5%8E%8B%E7%BC%A9%E7%9A%84%E5%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5.md)


### 3.Android NDK相关

、、TODO

- [JNI]
- [CMAKE]()

### 4.ffmpeg相关

- [雷晓华]()



### 步骤1：下载


TODO 下划线（原因  S16 失败！）  错了 是fdk_aac没有找到


**版本**： [FFmpeg 3.0.11 "Einstein"](http://ffmpeg.org/download.html#releases)




### 步骤2：编译


可参考 [手把手图文并茂教你用Android Studio编译FFmpeg库并移植](https://blog.csdn.net/hejjunlin/article/details/52661331)

但是上面的文章只编译了 arm下的so库，如果想编译 arm64v8a、x86等需要其他的脚本，在github上发现了这个库 [FFmpeg4Android/build_script](https://github.com/mabeijianxi/FFmpeg4Android),里面包含了各个版本的so库的编译脚本。

**注意NDK和TOOLCHAIN变量的设置**

**[TODO]** 编译各平台的so库

### 步骤3：Cmake 如何添加第三方so库

- [NDK学习( 二)，在NDK开发中引入第三方库（AndroidStudio Cmake）](https://blog.csdn.net/mxw3755/article/details/56676923)
- [Android studio 多个so库配置 ffmpeg库配置 cmake编译](https://blog.csdn.net/m0_37677536/article/details/78561085)
- [Android Studio NDK 入门教程（4）--优雅的在C++中输出Logcat](https://blog.csdn.net/venusic/article/details/52294815)


### 步骤4：验证ffmpeg已打进APK中



[fmpeg中第三方库的编译_libx264和librtmp
](https://blog.csdn.net/jiandanjiuhao_88/article/details/54694029)

[编译Android下可用的全平台FFmpeg(包含libx264与libfdk-aac)](https://blog.csdn.net/mabeijianxi/article/details/74544879)

https://blog.csdn.net/vnanyesheshou/article/details/54560684


https://www.aliyun.com/jiaocheng/13764.html

https://www.jianshu.com/p/1bb2fdb85392

https://blog.csdn.net/mabeijianxi/article/details/72983362


下面开始一些demo的笔记，首先强烈推荐两个博客


-https://blog.csdn.net/mabeijianxi
https://blog.csdn.net/leixiaohua1020/article/details/25430449

## 2. 第一个Demo，将PCM编码为AAC

参考

- [最简单的基于FFMPEG的音频编码器（PCM编码为AAC）](https://blog.csdn.net/leixiaohua1020/article/details/25430449)
- []()



遇到了一些问题，因为在Mac上使用Android Studio编译受限于NDK的版本等因素，会报出奇奇怪怪的问题，我的方法是打开Gradle Console 找到第一条error信息解决，然后慢慢的就可以运行了。

具体的代码参考 Github上的工作，里面的注释写的尽量详细吧。

其中有两个error让我记得特别清晰，其中一个说是C++的标准什么的 “#d”Pxxx   中间加空格。

另一个是show_childe_help的方法命名的问题，参数AVclass *class改成 AVclass *avclass。