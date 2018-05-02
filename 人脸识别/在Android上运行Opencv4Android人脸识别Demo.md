在官网[下载](http://opencv.org/releases.html)对应的Android SDK，发现里面有如下的结构

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/ai/facede2.png?raw=true)

- apk文件夹
  apk文件夹下包含了各平台的OpenCVManager.apk，sample里的apk安装后需要OpenCVManager.apk才能运行。因为sample里的apk依赖OpenCVManager.apk里的libopencv_java3.so文件。我在HTC one手机上尝试多次都不成功。
- samples
  包含示例的源代码和apk文件
- sdk
  
--------

下面是导入sample源码，通过ndk编译cpp代码生成so，而不依赖OpenCVManager.apk。

## 环境
OS：mac
IDE：Android Studio
SDK：o4a 3.2.0

## 步骤
在AS中import project，选择人脸识别的demo（face-detection）。AS会自动将依赖的OpencvSDK导入到工程中，结构如下。

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/ai/facedetection.png?raw=true)

此时直接编译会报出很多错误，包括类似这样的错误：

```
Error:(205, 45) 错误: 找不到符号
符号:   变量 GL_TEXTURE_EXTERNAL_OES
位置: 类 GLES11Ext

Error:(61, 34) 错误: 找不到符号
符号:   类 CameraManager
位置: 类 Camera2Renderer
```

这是因为编译的Andriod SDK版本不对，build.gradle中compileSdkVersion选择21，因为`android.hardware.camera2`是在API 21中添加的。

再次编译报出错误
```
Error:Execution failed for task ':openCVSamplefacedetection:compileDebugNdk'.
> Error: Your project contains C++ files but it is not using a supported native build system.
  Consider using CMake or ndk-build integration with the stable Android Gradle plugin:
   https://developer.android.com/studio/projects/add-native-code.html
  or use the experimental plugin:
   http://tools.android.com/tech-docs/new-build-system/gradle-experimental.
```

因为openCVSamplefacedetection模块下，有cpp文件我们需要用到，但工程没有ndk-build的任务或cmake的任务。
此时在src/main/jni文件夹右击选择`link c++ project`，选择`ndk-build`，然后选择Android.mk文件，AS会自动在gradle中创建ndk-build的任务。

再次编译报错：
```
Error:(14, 0) ../../sdk/native/jni/OpenCV.mk: No such file or directory
<a href="openFile:/Users/leegend/GitHub/face-detection1/openCVSamplefacedetection/src/main/jni/Android.mk">Open File</a>
```
因为Android.mk中引用了OpenCV.mk文件，找不到，参考[官方文档](http://docs.opencv.org/2.4/doc/tutorials/introduction/android_binary_package/dev_with_OCV_on_Android.html#application-development-with-static-initialization)，将Andriod.mk修改成如下

```
OPENCV_CAMERA_MODULES:=on
OPENCV_INSTALL_MODULES:=on
OPENCV_LIB_TYPE:=SHARED
OPENCV_ANDROID_SDK := /Users/leegend/Downloads/OpenCV-android-sdk/

```
到jni文件夹下运行ndk-build。再次编译程序，可以正常运行。
