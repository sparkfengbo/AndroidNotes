参考[维基百科](https://zh.wikipedia.org/wiki/%E9%80%B2%E9%9A%8E%E9%9F%B3%E8%A8%8A%E7%B7%A8%E7%A2%BC)、[知乎问题](https://www.zhihu.com/question/20629995)等资料对AAC格式进行总结，没有特别深入到细节，目的是让大家对AAC有一个总体的认识。

## 1.AAC是什么

**高级音频编码**（英语：**A**dvanced **A**udio **C**oding，AAC），出现于1997年，基于[MPEG-2](https://zh.wikipedia.org/wiki/MPEG-2)的[音频](https://zh.wikipedia.org/wiki/%E9%9F%B3%E8%A8%8A)[编码](https://zh.wikipedia.org/wiki/%E7%B7%A8%E7%A2%BC)技术。由Fraunhofer IIS、杜比实验室、AT&T、Sony、Nokia等公司共同开发。2000年，[MPEG-4](https://zh.wikipedia.org/wiki/MPEG-4)标准出现后，AAC重新集成了其特性，加入了SBR技术和PS技术，为了区别于传统的MPEG-2 AAC又称为MPEG-4 AAC。

## 2.AAC封装格式分类

（参考[鹏小鹕](https://www.zhihu.com/people/peng-xiao-hu-60)在问题[AAC-LC 是什么格式？和 AAC 有什么区别？](https://www.zhihu.com/question/20629995)中的回答）

参考知乎的问题，AAC的音频文件格式常用的有以下两种：
- ADIF

 Audio Data Interchange Format 音频数据交换格式。这种格式的特征是可以确定的找到这个音频数据的开始，不需进行在音频数据流中间开始的解码，即它的解码必须在明确定义的开始处进行。故这种格式常用在磁盘文件中。
 
![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/aac1.png?raw=true)
- ADTS

  Audio Data Transport Stream 音频数据传输流。这种格式的特征是它是一个有同步字的比特流，解码可以在这个流中任何位置开始。
简言之。ADIF只有一个文件头，ADTS每个包前面有一个文件头。
![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/aac2.png?raw=true)

所以在直播等领域用的最多的还是ADTS。

## 3.ADTS格式

ADTS的头信息分为两部分，一个是固定头信息、紧接着是可变头信息。

固定头信息

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/aac3.png?raw=true)

可变头信息

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/aac4.png?raw=true)

合起来

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/aac5.png?raw=true)

更直观一点：

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/live/aac6.png?raw=true)


所以必须要了解ADTS封装的格式，否则就不能自己将AAC打包到ADTS。

在FFmpeg的adtsenc.c 中定义了[adts_write_frame_header](http://ffmpeg.org/doxygen/3.2/adtsenc_8c.html#a7bd77f94eef0965c207e6448d81e8d3c)方法，可以写ADTS头，大家可以参考。

### 3.1 部分参数介绍

序号5的profile是AOT - 1，AOT参考[这里](https://wiki.multimedia.cx/index.php/MPEG-4_Audio#Audio_Object_Types)

- 0: Null
- 1: [AAC](https://wiki.multimedia.cx/index.php/AAC) Main
- 2: [AAC](https://wiki.multimedia.cx/index.php/AAC) LC (Low Complexity)
- 3: [AAC](https://wiki.multimedia.cx/index.php/AAC) SSR (Scalable Sample Rate)
- 4: [AAC](https://wiki.multimedia.cx/index.php/AAC) LTP (Long Term Prediction)
- 5: SBR ([Spectral Band Replication](https://wiki.multimedia.cx/index.php/Spectral_Band_Replication))
- 6: [AAC](https://wiki.multimedia.cx/index.php/AAC) Scalable
- 7: [TwinVQ](https://wiki.multimedia.cx/index.php/TwinVQ)
- 8: [CELP](https://wiki.multimedia.cx/index.php?title=CELP&action=edit&redlink=1) (Code Excited Linear Prediction)

....省略若干type

adts_buffer_fullness：0x7FF 说明是码率可变的码流。


---------

## 4.AAC规格

因为AAC是一个庞大家族，他们共分为9种规格，以适应不同场合的需要，也正是由于AAC的规格（Profile）繁多，导致普通电脑用户感觉十分困扰：

- MPEG-2 AAC LC低复杂度规格（Low Complexity）
- MPEG-2 AAC Main主规格
- MPEG-2 AAC SSR可变采样率规格（Scaleable Sample Rate）
- MPEG-4 AAC LC低复杂度规格（Low Complexity），现在的手机比较常见的MP4文件中的音频部分就包括了该规格音频文件
- MPEG-4 AAC Main主规格
- MPEG-4 AAC SSR可变采样率规格（Scaleable Sample Rate）
- MPEG-4 AAC LTP长时期预测规格（Long Term Predicition）
- MPEG-4 AAC LD低延迟规格（Low Delay）
- MPEG-4 AAC HE高效率规格（High Efficiency）

上述的规格中，主规格（Main）包含了除增益控制之外的全部功能，其音质最好，而低复杂度规格（LC）则是比较简单，没有了增益控制，但提高了编码效率，至“SSR”对“LC”规格大体是相同，但是多了增益的控制功能，

另外，MPEG-4 AAC/LTP/LD/HE，都是用在低比特率下编码，特别是“HE”是有Nero AAC编码器支持，是近来常用的一种编码器，不过通常来说，Main规格和LC规格的音质相差不大，因此目前使用最多的AAC规格多数是“LC”规格，因为要考虑手机目前的内存能力未达合理水准。


MPEG-4 AAC LC（Low Complexity）是最常用的规格，我们叫“低复杂度规格”，我们简称“LC-AAC”，这种规格在中等码率的编码效率以及音质方面，都能找到平衡点。所谓中等码率，就是指：96kbps-192kbps之间的码率。因此，如果要使用LC-AAC规格，请尽可能把码率控制在之前说的那个区间内。


附加一篇参考文章：[AAC规格（LC，HE，HEv2）及性能对比](http://blog.csdn.net/leixiaohua1020/article/details/11971419)

---------

**参考**

-  [AAC格式简介](http://blog.csdn.net/leixiaohua1020/article/details/11822537)

- [AAC-LC 是什么格式？和 AAC 有什么区别？](https://www.zhihu.com/question/20629995)

- [视音频数据处理入门：AAC音频码流解析](http://blog.csdn.net/leixiaohua1020/article/details/50535042)

**ADTS：**

- [ADTS](https://wiki.multimedia.cx/index.php/ADTS)

- [MPEG-4 Audio](https://wiki.multimedia.cx/index.php/MPEG-4_Audio#Channel_Configurations)

- [AAC的ADTS头文件信息介绍](http://blog.csdn.net/jay100500/article/details/52955232)
