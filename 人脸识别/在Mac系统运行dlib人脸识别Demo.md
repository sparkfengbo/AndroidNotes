今天进行了一次比较失败的尝试，人脸识别目前应用比较火的有opencv、dlib等。

--------
今天第一次阅读了dlib相关的资料，在mac系统上编译dlib并运行，遇到了一些问题。

问题1：Not supported GUI
解决：这个是mac系统没有GUI支持，需要下载xquartz。

问题2：安装xquartz后提示1651 FATAL [1] dlib.gui_core: Unable to connect to the X display
解决：这个问题需要重新启动电脑

**问题3：dlib是否支持gif格式。**
dlib在config.h.in中留下了`#cmakedefine DLIB_GIF_SUPPORT`在config.h中留下了`#define DLIB_GIF_SUPPORT`的定义接口，但是没有gif支持的代码

**问题4：dlib是否支持流媒体**

**问题5：dlib人脸识别训练数据shape_predictor_68_face_landmarks.dat达到了97m，如果非要这个数据集不可，根本不可能移植到移动设备中**

以上几个问题日后解决。

---------

运行Demo的流程和结果如下

##1.下载##
可以到[Github](https://github.com/davisking/dlib)去clone，也可以到[官网](http://dlib.net/)下载

##2.编译##
按照Github上的指示，
```
mkdir build; 
cd build; 
cmake .. ; 
cmake --build .
```

##3.安装GUI -  XQuartz ##
人脸检测Demo需要GUI，如果mac本地不支持gui在编译后不会生成face_landmark_detection_ex对应的可执行文件。[下载地址](https://www.xquartz.org/)

##4.运行人脸检测Demo##
将example中的faces文件夹复制到build中。
到官网[下载](http://dlib.net/files/)shape_predictor_68_face_landmarks.dat,并复制到build文件夹中。
按照face_landmark_detection_ex.cpp注释的内容运行程序：
```
./face_landmark_detection_ex shape_predictor_68_face_landmarks.dat faces/*.jpg
```
###5.运行结果

![](http://upload-images.jianshu.io/upload_images/952890-6b0fdf492ae1c5ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但看到下图中间的人时你会发现其实他的识别结果并不是非常准确
![](http://upload-images.jianshu.io/upload_images/952890-7242a30006ff955e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

----------

[各人脸检测方法性能比较（不太准确）](opencv、pico、npd、dlib、face++等多种人脸检测算法结果比较)