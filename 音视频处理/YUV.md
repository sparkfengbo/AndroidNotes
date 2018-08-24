综合网上、书上查找的资料，总结如下.

## YUV介绍

讲到YUV应该先讲到**YIQ**和**RGB**。

**RGB**大家都了解，是通过三原色的组合产生各种颜色，RGB也有很多格式，比如`RGB24`、`RGB32`，分别表示R、G、B分别占用24位、32位，所以他们每帧的图片分辨占用 `w * h * 3 bytes、w * h * 4 `bytes

[YIQ](https://baike.baidu.com/item/YIQ/1977357?fr=aladdin)是NTSC电视系统标准，美国的标准。Y是提供黑白电视及彩色电视的亮度信号（Luminance），即亮度（Brightness），I代表In-phase，色彩从橙色到青色，Q代表Quadrature-phase，色彩从紫色到黄绿色。初学者看到Y表示亮度时可能会有一种误解，这里的亮度和我们平时使用手机设置的屏幕亮度是不一样的，Y的亮度可以说是图像的灰度值，灰度值不同于黑白，可以表示黑色和白色之间的各种颜色深度，类似这样。


![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/YUV1.jpeg?raw=true)

**所以YIQ这样的标准可以很容易的兼容黑白和彩色电视机了。**

由于多种原因，提出了多种YIQ编码的变体提高视频传输的色彩质量，YUV就是其中一种，也有人叫做YCrCb。

[YUV](https://baike.baidu.com/item/YUV)主要用于优化彩色视频信号的传输，使其向后相容老式黑白电视。与RGB视频信号传输相比，它最大的优点在于只需占用极少的频宽（RGB要求三个独立的视频信号同时传输）。其中“Y”表示明亮度（Luminance或Luma），也就是灰阶值；而“U”和“V” 表示的则是色度（Chrominance或Chroma），作用是描述影像色彩及饱和度，用于指定像素的颜色。(“色度”则定义了颜色的两个方面─色调与饱和度，分别用Cr和Cb来表示。)

YUV的存储中与RGB格式最大不同在于，RGB格式每个点的数据是连续保存在一起的，而YUV 的数据中为了节约空间，U，V分量空间会减小。

>在技术文档里，YUV经常有另外的名字, YCbCr ,其中Y与YUV 中的Y含义一致，Cb , Cr 
同样都指色彩,，只是在表示方法上不同而已，Cb Cr 
就是本来理论上的“分量/色差”的标识。C代表分量(是component的缩写)Cr、Cb分别对应r(红)、b(蓝)分量信号，Y除了g(绿)分量信
号，还叠加了亮度信号。
> 还有一种格式是YPbPr格式,它与YCbPr格式的区别在于,其中YCbCr是隔行信号，YPbPr是逐行信号。


## YUV分类

YUV的格式有两类，打包（packed）格式和平面（planar）格式。打包格式通常是几个相邻的像素组成一个宏像素（macro-pixel）。平面格式三个数组分开存放YUV三个分量，就像是一个三维平面一样。下面会讲。


>YUV 的存储中与 RGB 格式最大不同在于，RGB 格式每个点的数据是连继保存在一起的。即 R，G，B 是前后不间隔的保存在 2-4byte 空间中。而 YUV 的数据中为了节约空间，U，V 分量空间会减小。每一个点的 Y 分量独立保存，但连续几个点的 U，V 分量是保存在一起的，（反正人眼一般也看不出区别）. 这几 个点合起来称为 macro-pixel， 这种存储格式称为 Packed 格式。
>
>另外一种存储格式是把一幅图像中 Y，U，V 分别用三个独立的数组表示。这种模式称为 planar 模式。
>
>YUV 格式有两大类：planar 和 packed。对于 planar 的 YUV 格式，先连续存储所有像素点的 Y，紧接着存储所有像素点的 U，随后是所有像素点的 V。
对于 packed 的 YUV 格式，每个像素点的 Y,U,V 是连续交 * 存储的。



通常，YUV采样方式有三种，**YUV4:4:4**，**YUV4:2:2**，**YUV4:2:0**。

```
YUV 4:4:4采样，每一个Y对应一组UV分量。 
YUV 4:2:2采样，每两个Y共用一组UV分量。 
YUV 4:2:0采样，每四个Y共用一组UV分量。 
```

单像素多占内存分别是3B、2B、1.5B。（通常每个采样是8bit，YUV444是YUV，8 + 8 + 8； YUV422因为每两个Y共用一组UV，相当于 8 + 4 + 4； 而 YUV420 相当于 8+ 2 + 2）

**例如：**

####  YUVY 格式 （属于YUV422）

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/yuv2.png?raw=true)


YUYV为YUV422采样的存储格式中的一种，相邻的两个Y共用其相邻的两个Cb、Cr，分析，对于像素点Y'00、Y'01 而言，其Cb、Cr的值均为 Cb00、Cr00，其他的像素点的YUV取值依次类推。


####  YV12、YU12格式（属于YUV420）

YV12，YU12格式（属于YUV420）是一种Plane模式。

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/yuv3.png?raw=true)

将Y、U、V分量分别打包，依次存储。其每一个像素点的YUV数据提取遵循YUV420格式的提取方式，即4个Y分量共用一组UV。注意，上图中，Y'00、Y'01、Y'10、Y'11共用Cr00、Cb00，其他依次类推。


####  NV12、NV21（属于YUV420）

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/yuv4.png?raw=true)

NV12和NV21属于YUV420格式，是一种two-plane模式，即Y和UV分为两个Plane，但是UV（CbCr）为交错存储，而不是分为三个plane。其提取方式与上一种类似，即Y'00、Y'01、Y'10、Y'11共用Cr00、Cb00。

###  NV12、YV12的区别

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/yuv5.png?raw=true)


![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/yuv6.png?raw=true)

在YV12中U和V都是连续排布的，而在NV12中,U和V就交错排布的。看到内存中的排布很清楚，先开始都是Y，之后的都是U1V1U2V2的交错式排布。对于像素的压缩的效果是一样的。

![image.png](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/yuv7.png?raw=true)

强烈推荐[雷霄骅的这篇文章](http://blog.csdn.net/leixiaohua1020/article/details/50534150)，操作之后会对YUV、RGB的存储格式有更深的理解


- [图文详解 YUV420 数据格式](http://www.360doc.com/content/16/0517/16/496343_559909505.shtml)

-----

>什么是 YCbCr?
>

>　　YCbCr 表示隔行分量端子，是属于 YUV 经过缩放和偏移的翻版，常说的 YUV 也称 作 YCbCr。其中 Y 与 YUV 中的 Y 含义一致，Cb , Cr 同样都指色彩,，只是在表示方法上不同而已，Cb Cr 就是本来理论上的 “分量 / 色差” 的标识。C 代表分量 (是 component 的缩写)Cr、Cb 分别对应 r(红)、b(蓝) 分量信号，Y 除了 g(绿)分量信 号，还叠加了亮度信号。
>
>
>   其中 YCbCr 是隔行信号，YPbPr 是逐行信号。YCbCr 是在计算机系统中应用最多的一种信号，其应用领域很广泛，JPEG、MPEG 均采用此格式。
> 
>　　　
>什么是 YPbPr?

>Y'CbCr 在模拟分量视频 (analog component video) 中也常被称为 YPbPr，YPbPr 是将模拟的 Y、PB、PR 信号分开，使用三条线缆来独立传输，保障了色彩还原的准确性，YPbPr 表示逐 行扫描色差输出. YPbPr 接口可以看做是 S 端子的扩展，与 S 端子相比，要多传输 PB、PR 两种信号，避免了两路色差混合解码并再次分离的过程，也保持了 色度通道的最大带宽，只需要经过反矩阵解码电路就可以还原为 RGB 三原色信号而成像，这就最大限度地缩短了视频源到显示器成像之间的视频信号通道，避免了 因繁琐的传输过程所带来的图像失真，保障了色彩还原的准确，目前几乎所有大屏幕电视都支持色差输入。