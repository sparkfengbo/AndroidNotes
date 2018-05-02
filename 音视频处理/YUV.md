综合网上、书上查找的资料，总结如下.

## YUV介绍

讲到YUV应该先讲到**YIQ**和**RGB**。

**RGB**大家都了解，是通过三原色的组合产生各种颜色，RGB也有很多格式，比如`RGB24`、`RGB32`，分别表示R、G、B分别占用24位、32位，所以他们每帧的图片分辨占用 `w * h * 3 bytes、w * h * 4 `bytes

**[YIQ](https://baike.baidu.com/item/YIQ/1977357?fr=aladdin)**是NTSC电视系统标准，美国的标准。Y提供黑白电视及彩色电视的亮度信号，I和Q两个分量携带颜色信息，指的是色调，用来描述色彩和饱和度的属性。初学者看到Y表示亮度时可能会有一种误解，这里的亮度和我们平时使用手机设置的屏幕亮度是不一样的，Y的亮度可以说是图像的灰度值，灰度值不同于黑白，可以表示黑色和白色之间的各种颜色深度，类似这样。

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/YUV1.jpeg?raw=true)

**所以YIQ这样的标准可以很容易的兼容黑白和彩色电视机了。**

由于多种原因，提出了多种YIQ编码的变体提高视频传输的色彩质量，YUV就是其中一种，也有人叫做YCrCb。

YUV的存储中与RGB格式最大不同在于，RGB格式每个点的数据是连续保存在一起的，而YUV 的数据中为了节约空间，U，V分量空间会减小。


## YUV分类

YUV的格式有两类，打包（packed）格式和平面（planar）格式。打包格式通常是几个相邻的像素组成一个宏像素（macro-pixel）。平面格式三个数组分开存放YUV三个分量，就像是一个三维平面一样。下面会讲。

通常，YUV采样方式有三种，**YUV4:4:4**，**YUV4:2:2**，**YUV4:2:0**。

```
YUV 4:4:4采样，每一个Y对应一组UV分量。 
YUV 4:2:2采样，每两个Y共用一组UV分量。 
YUV 4:2:0采样，每四个Y共用一组UV分量。 
```

单像素多占内存分别是3B、2B、1.5B。

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


![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/yuv5.png?raw=true)


![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/yuv6.png?raw=true)

在YV12中U和V都是连续排布的，而在NV12中,U和V就交错排布的。看到内存中的排布很清楚，先开始都是Y，之后的都是U1V1U2V2的交错式排布。对于像素的压缩的效果是一样的。

![image.png](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/yuv7.png?raw=true)

强烈推荐[雷霄骅的这篇文章](http://blog.csdn.net/leixiaohua1020/article/details/50534150)，操作之后会对YUV、RGB的存储格式有更深的理解