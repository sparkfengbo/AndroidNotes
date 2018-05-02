**文章多处链接需要科学上网**


本文按顺序主要讲解了ADB的原理，使用Wi-Fi连接设备，ADB常用命令，在Java代码中执行shell命令，使用ddmlib进行扩展。


### ADB的原理
> 参考 [官方文档](https://developer.android.com/studio/command-line/adb.html#commandsummary)


ADB（Android Debug Bridge）是一个通用的命令行工具，能让你和模拟器或连接的Android手机通信。

ADB的结构是一个client-server的结构，包含3个部分：
 - A Client ： 发送命令。客户端在你开发的PC上运行，当你在shell里使用Adb命令的时候就会开启一个client。（其实你的shell就是一个client）

 - A daemon : 在设备上执行命令。守护进程在设备上后台运行。(也就是一个叫做aabd的东西,运行在Andriod设备的底层)

 - A server ： 管理客户端（client）和守护进程（daemon）的连接。server在开发app的PC上后台运行。


*你可以在 <sdk>/platform-tools 找到adb工具*
</br>
### ADB是如何工作的？

当你开启一个adb client，client会首先检查adb server是否运行，如果没有的话先启动一个adb server，当server启动后，server默认绑定到本地（PC）的TCP端口5037（这个端口号可以设置，后文有述）并开始监听从client发送的命令。（所有的adb client都会用5037端口和server通信）

然后server会建立和所有正在运行的设备或模拟器的通信连接。server通过扫描5555至5585之间的奇数号端口查找设备（这就是说设备所使用的端口号一定是5555-5585之间的奇数），如果server找到了一个守护进程daemon（运行在设备上的），那么server就会在这个端口建立一个连接（server是client和daemon的中间的桥梁）
>注意：每一个设备需要一对连续的端口号，奇数端口号用来建立adb连接，偶数端口号用于控制台连接（原文是console connections，据我理解应该是指 [控制模拟器用的console连接](https://developer.android.com/studio/run/emulator-commandline.html#console))

像这样：

```
Emulator 1, console: 5554
Emulator 1, adb: 5555
Emulator 2, console: 5556
Emulator 2, adb: 5557
and so on...

```

上一个结构图：

![结构图](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/adb1.jpeg?raw=true)

*上面的图片来自于一篇很早比较详细的文章 [android adb adbd analyse](http://blog.csdn.net/liranke/article/details/4999210#_Toc248316011 )
*

有可能有人会问：5555-5585的奇数端口号是指定设备的，那么设备有没有**上限**呢？

答案是没有，原因如下：

- 1.adb可以使用WiFi连接，也就是通过无线网络连接。下文会讲无线连接如何使用
- 2.adb可以为adb server指定端口号，指定端口号后可以开启多个server(不过Android Studio在调试程序时只识别5037端口的server)。

**像这样：**

可以通过大写的-P指定端口号，指定端口号后会开启一个新的server，这样的缺点就是，如果以后想查看5038端口server的一系列操作，比如查看连接的设备也必须加 -P 5038，否则查看的只是5037的server连接的设备。
如果之前已经开启了5037的server，那么现在你的PC上现在已经有了两个server，这里注意，你的设备只能和其中一个server通信。
![大写的 -P 指定端口号 ](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/adb2.png?raw=true)



![de960f3f9c07003951b1fbe15e3afa88.png](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/adb3.png?raw=true)

正式因为上面结构图的结构，才能使得adb能够通过wifi进行连接。

### 使用wi-fi连接的使用方法：

1.将你的Android设备和你的开发机器连接到**同一个Wi-Fi网络环境**下，记住是**同一个**。不是**所有**的无线节点都可以**匹配**，你也许需要使用支持adb的防火墙配置。（我将防火墙关闭了）

2.将你的设备和电脑通过USB连接

3.将你的手机设置成在端口5555监听TCP/ip连接（在PC上执行，端口号自己指定）

`$ adb tcpip 5555 `

4.**断开USB数据线**

5.查看你的手机的IP地址（在手机连接的WiFi网络的心里了能看到，每个手机的配置不一样~）

6.通过指定IP地址进行连接
`$ adb connect <device-ip-address>`

7.看看效果吧，确定你的电脑是否已经连接上你的手机了

```
$ adb devices
List of devices attached
<device-ip-address>:5555 device
```

**如果没有连接上**

- 确定连在同一个wifi环境下
- 重试`adb connect`或重启adb server `adb kill-server` & `adb start-server`

熟悉Android Studio的人应该知道有一个叫做 [ADBWIFI](https://github.com/layerlre/ADBWIFI) 的调试插件，里面用到的也是这些命令原理，刚开始我还觉得能自己开发一款Android Studio的插件很神奇，其实你打开那个链接琢磨琢磨，你也可以。里面用到了ddmlib这个jar包，文章的最后**简要**介绍一下（感兴趣的话可以好好研究。。）

### ADB常用命令
语法：`adb [-d|-e|-s <serialNumber>] <command>`

如果你只连接一个物理设备，可以通过-d快速指定物理设备，如果你连接了只连接了一个模拟器，可以通过-e快速指定模拟器。
- devices : 连接的设备列表，你可以看到serialNumber
- help ： 命令帮助
- version ： adb版本
- [logcat [option] [filter-specs]](http://www.hanshuliang.com/?post=32) ：在屏幕上打印log，如果这个命令不会可以输入`adb logcat --help`查看怎么使用
- bugreport ： 打印dumpsys, dumpstate, logcat的信息，为了报告bug，类似`adb bugreport > xxx.log`
- start-server : 开启一个adb server
- kill-server : 关闭adb server
- install <path-to-apk> ：安装apk（specified as a full path to an .apk file)）
- pull <remote> <local> ： 从你设备的remote拷贝文件到你PC上的local
- push <local> <remote> ： 从你PC上的local拷贝文件到你设备的remote
- forward <local> <remote> ： 将你本地的特定端口的信息转发给你设备的remote端口上。
like this：
> adb forward tcp:6100 tcp:7100    PC上所有6100端口通信数据将被重定向到手机端7100端口server上
adb forward tcp:6100 local:logd   PC上所有6100端口通信数据将被重定向到手机端UNIX类型socket上
- get-serialno : 得到设备的序列号，其实就是devices的结果的前半部分
- get-state ： 得到设备的状态[offline， device， no device]
- wait-for-device ： 直到设备online之后才会继续执行，否则阻塞执行。like this：
>` adb wait-for-device install <app>.apk` 安装apk需要设备启动之后才能执行，和其他adb命令配合使用

*jdwp 和 ppp两个命令没搞清楚如何使用，欢迎知道的告诉我，互相学习，感谢 ：）*

---
### adb shell

[shell命令](https://developer.android.com/studio/command-line/shell.html#shellcommands)运行在android的设备上，命令的二进制文件在手机的`/system/bin/...`下

语法：`adb [-d|-e|-s <serialNumber>] shell <shell_command>`

**am**

在shell命令下，你可以通过activity manager 工具(am)执行系统操作，包括开始一个activity， 强制关闭进程，广播intent，设置设备屏幕参数等。
语法是`am <command>`，eg : `adb shell am start -a android.intent.action.VIEW`

内容比较多，建议翻墙详细看，原文挺简单的，我就不翻译了 ：）

和am搭配使用的有：

|Comand| Description|
|:-----:|-------|
| start [options] <INTENT>|Start an [Activity](https://developer.android.com/reference/android/app/Activity.html) specified by [<INTENT>](http://blog.csdn.net/zhou_junhua/article/details/12781333) |
| startservice [options] <INTENT>|Start the [Service](https://developer.android.com/reference/android/app/Service.html) specified by <INTENT>.|
|force-stop <PACKAGE>|Force stop everything associated with <PACKAGE> (the app's package name).|
|kill [options] <PACKAGE>|Kill all processes associated with <PACKAGE> (the app's package name). This command kills only processes that are safe to kill and that will not impact the user experience.|

**pm**

在shell命令下，你可以通过package manager(pm)执行和包相关的操作。语法是`pm <command>`，eg : `adb shell pm uninstall com.example.MyApp`,和am类似，这里就不一一展开了，需要请看[官方文档](https://developer.android.com/studio/command-line/shell.html#pm)。

**截图**

像这样eg: `$ adb shell screencap /sdcard/screen.png`
你还可以这样，截屏后从手机copy一份。
```
$ adb shell
shell@ $ screencap /sdcard/screen.png
shell@ $ exit
$ adb pull /sdcard/screen.png
```

**录屏**

仅支持 Android 4.4 (API level 19)及以上

>**Note:** Audio is not recorded with the video file.
仅仅是画面而已

还有很多参数可以设置，这里不展开
eg: `$ adb shell screenrecord /sdcard/demo.mp4`

**其他**

**[dumpsys](https://source.android.com/devices/input/diagnostics.html)**

```
dumpsys [options]
         meminfo 显示内存信息
         cpuinfo 显示CPU信息
         account 显示accounts信息
         activity 显示所有的activities的信息
         window 显示键盘，窗口和它们的关系
         wifi 显示wifi信息
         and so on

```
eg:`adb shell dumpsys meminfo [packageName]`

By the way....你可以在**java代码中执行这些命令**，并将结果写到文件中，然后将文件发送到你的服务器上进行分析~~


参考：
[Running Shell commands though java code on Android?](http://stackoverflow.com/questions/6882248/running-shell-commands-though-java-code-on-android)
[writing dumpstate to file android](http://stackoverflow.com/questions/9011056/writing-dumpstate-to-file-android)


### ADB扩展

这里只简单的说一些。
 [ADBWIFI](https://github.com/layerlre/ADBWIFI) 插件的源码下载下来后你可以看到里面有一个ddmlib的类库，它的位置在你的android-sdk下面的/sdk/tools/lib 目录下，这个目录下还有ddmuilib.jar，ddms.jar等。

![52463105e63b60970431a4018c01daab.png](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/adb4.png?raw=true)

那么这些工具有什么用呢？


通过这些工具你可以在你的代码中


1.创建ADB

```
AndroidDebugBridge bridge = AndroidDebugBridge.createBridge();
```

2.获得ADB连接的设备

```
IDevice devices[] = bridge.getDevices();
```
3.操作设备

```
device.installPackage(path, true, args);  //device instance of IDevice
device.uninstallPackage(pakagename);  
```

4.执行adb命令

```
device.executeShellCommand(cmd, receiver);
// receiver extends MultiLineReceiver
// cmd like "dumpsys meminfo [packageName]" adb shell command
```

是不是很酷？如果能够再深入下去应该能发掘更多有意思的东西，感兴趣的自己研究研究吧，东西挺多的 ：）

可以参考 [使用ddmlib实现android 性能监控](http://blog.csdn.net/kittyboy0001/article/details/47317855) 或直接阅读 [ADBWIFI](https://github.com/layerlre/ADBWIFI) 的源代码