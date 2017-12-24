#FLV格式

##1.背景
关于FLV的背景和历史，可以参考[维基百科页面](https://zh.wikipedia.org/wiki/Flash_Video)和[百度百科](https://baike.baidu.com/item/flv/6623513?fr=aladdin)页面。

FLV是被众多新一代视频分享网站所采用，是目前增长最快、最为广泛的视频传播格式。是在sorenson公司的压缩算法的基础上开发出来的。后来被Adobe接管。现在有FLV的变种，F4V，支持H264格式。


##2.格式
FLV的格式相对其他格式来讲已经很简单了，[官方文档](http://www.adobe.com/content/dam/Adobe/en/devnet/flv/pdfs/video_file_format_spec_v10.pdf)对FLV的介绍篇幅很短。

总体上讲，FLV文件构成如下

```
FLV文件 = FLV头文件 + FLV文件体（FLV file body） 
```
```
FLV文件体 = 前一个tag大小0（PreviousTagSize0） + tag1 + 前一个tag大小1（PreviousTagSize1） + tag2+前一个tag大小2 + ... + ... 前一个tag大小N-1 + tagN + 前一个tag大小N

```
FLV文件体结构如下图

![](/Users/leegend/Desktop/tmppic/5da56f2096e1ea74c7be9ed76f1b49ac.png
)

###2.1FLV 头文件

![](/Users/leegend/Desktop/tmppic/2662600794e504e78dbfa91c22c065ff.png
)

当版本是1时，最后的DataOffset值为9，DataOffset的字段是保留用于以后扩展的，所以通常情况下，FLV头文件为9字节，如下：

FLV头文件：（9字节）

- 1-3B：前3个字节是文件格式标识（FLV 0x46 0x4C 0x56）。
- 4-4B：第4个字节是版本（0x01）
- 5-5B：第5个字节的前5个bit是保留的必须是0.
  - 第5个字节的前5个bit也是保留的必须是0
  - 第5个字节的第6个bit音频类型标志（TypeFlagsAudio）
  - 第5个字节的第7个bit也是保留的必须是0
  - 第5个字节的第8个bit视频类型标志（TypeFlagsVideo）
- 6-9B: 第6-9的四个字节还是保留的。其数据为0x00000009 .


整个文件头的长度，一般是9（3+1+1+4）

UI代表Unsigned Int，UB代表Unsigned Bit，SI代表SignedInt。

###2.2tag大小

始终为UI32,占用4个字节

tag0大小始终为0，我们的真实数据都是从tag1开始。
所以FLV文件体可以理解为
`tag0 + 若干个【tagN+tagN的大小】`

或

`若干个【前一个tag的大小+tag】+最后一个tag的大小`

前一个tag大小N（PreviousTagSizeN）的“之前“是表示PreviousTagSizeN放在tagN的后面，PreviousTagSizeN的值是
tagN的大小，包括tagN的头。对于FLV version 1来讲，这个值应该是 11 + tagN的DataSize的值（看后面介绍的tag的的格式，DataSize的值表示Data中数据的长度，其他的字段分别占1+3+3+1+3字节，等于11）

###2.3tag基本格式
![](/Users/leegend/Desktop/tmppic/95c53a114d67829c7569dba1ff9619a3.png)
tag类型信息

- 1-1B：tag类型（1字节）；0x8音频；0x9视频；0x18脚本数据
- 2-4B：tag内容大小（3字节）
- 5-7B：时间戳（3字节，毫秒）（第1个tag的时候总是为0,这个值是相对于第一个tag计算的，单位毫秒）
- 8-8B：时间戳扩展（1字节）让时间戳变成4字节（以存储更长时间的flv时间信息），本字节作为时间戳的最高位，也就是总的时间戳变为SI32，高8位为扩展时间戳表示，低24位由说之前的时间戳表示。在flv回放过程中，播放顺序是按照tag的时间戳顺序播放。任何加入到文件中时间设置数据格式都将被忽略。
- 9-11B：streamID（3字节）总是0
- 12-xB：Data ：具体的数据，我们先只关注AUDIODATA、VIDEODATA


###2.4 Audio tags

####2.4.1 AUDIODATA

当tag的TagType = 8时，Data里面是AUDIODATA。AUDIODATA的结构如下

![](/Users/leegend/Desktop/tmppic/b0702099fa07aae7c5d7a1a02536d6af.png)
![](/Users/leegend/Desktop/tmppic/882d4586b175ffc79774be763bb19966.png)

- 1-4b：SoundData的类型（4bit）；值为10时是AAC格式
- 5-6b：采样率（2bit）对AAC来讲总是3
- 7-7b：采样的大小（1bit）（每个采样的大小，属于未压缩的格式，0位8bit采样，1位16bit采样）
- 8-8b：声音的类型（1bit）0是sndMono，单声道，1是sndStereo，立体声。对于AAC，永远是1
- 9-xxb：具体的声音数据，当SoundFormat是10的时候，SoundData为AACAUDIODATA。

需要注意的是，当SoundFormat指定为AAC格式的时候，SoundType必须为1（stereo），SoundRate必须为3（44kHz）。但这并不是说AAC的音频数据一定是stereo，44kHz，Flash Player会忽略这些值，并在AAC的比特流中获得channel和采样率的数据。


####2.4.2 AACAUDIODATA
![](/Users/leegend/Desktop/tmppic/ce0c5cc252e4d87c68c6168814011c12.png)


当Type是0时，代表是AAC Sequence Header，Data就是AudioSpecificConfig，如果Type是1，那么Data是Raw AAC frame data，AAC音频原始数据，不包含AAC头数据ADTS。

####2.4.3 AudioSpecificConfig

AudioSpecificConfig是在ISO14496-3中1.6.2.1定义的。定义的内容比较繁琐。
![](/Users/leegend/Desktop/tmppic/audiospecificconfig.png)


对于使用基本的音频来说可以
[参考维基百科](https://wiki.multimedia.cx/index.php?title=MPEG-4_Audio)的总结，如下

```
The Audio Specific Config is the global header for MPEG-4 Audio:

5 bits: object type
if (object type == 31)
    6 bits + 32: object type
4 bits: frequency index
if (frequency index == 15)
    24 bits: frequency
4 bits: channel configuration
var bits: AOT Specific Config
```
AOT是Audio Object Type，具体的频率值的定义和AOT等的定义参考[维基百科页面](
https://wiki.multimedia.cx/index.php/MPEG-4_Audio#Audio_Object_Types)。

那么我们在直播中的Audio Specific Config包含
5 bits: object type
4 bits: frequency index
4 bits: channel configuration

>一般来讲，FLV文件开始是一个FLV头部，然后是MetaData数据（后面介绍），然后是音视频的Header数据，对于音频对应的是AAC sequence header，对于AVC是AVC sequence header，在直播中发送一次即可。在Header的后面才是AAC或AVC数据。

举例来讲：
![](/Users/leegend/Desktop/tmppic/20160429094448.png)
FLV文件开头是FLV的头部，然后是MetaData元信息，上例中紧接着是一个AudioTag，这个AudioTag，由0x000004A1的地址可知，红框中的内容是AUDIODATA，第一个字节0xAF对应AUDIODATA表格，A的十进制是10，对应AAC，F对应二进制是1111，前两位代表采样频率，对应3即44kHz，第三和第四位分别代表SoundSize和SoundType。

紧接着的一个字节，0x00代表AACPacket的类型是AAC sequence header（AACPacketType正好是UI8，一个字节），那么接下来的0x13 0x88 0x56 0xE5 0xA5 0x48 0x80就是AudioSpecificConfig的内容。

0x13 二进制为 0001 0011，前5位代表AOT，为2即AAC LC，接下来的四位为频率，
0x88 二进制为 1000 1000，那么接下来的4位的值是7，代表22050kHz，接下来的四位为channel，值为1对应单声道。因为AOT为2，所以后面的数据包含了GASpecificConfig的内容。
GASpecificConfig的语义如下：
![](/Users/leegend/Desktop/tmppic/196b2fe11f9f94536b2f6accbe3ad247.png)

0x88 的后三位均为0，所以GASpecificConfig的frameLengthFlag=0，dependsOnCoreCoder=0，extensionFlag=0，0x88表示的内容到GASpecificConfig就结束了。那么56 E5 A5 48 80代表什么呢？我还没有想明白

AudioSpecifgConfig的语义可参考[ISO_IEC_14496-3 subpart1.pdf](http://read.pudn.com/downloads98/doc/comm/401153/14496/ISO_IEC_14496-3%20Part%203%20Audio/C036083E_SUB1.PDF)
GASpecificConfig的语义可参考[ISO_IEC_14496-3 subpart4.pdf](http://read.pudn.com/downloads98/doc/comm/401153/14496/ISO_IEC_14496-3%20Part%203%20Audio/C036083E_SUB4.PDF)





###2.5 Video tags

####2.5.1 VIDEODATA

![](/Users/leegend/Desktop/tmppic/d1de72173b1b7555d2d0a8ce9c3f7a23.png)

- 1-4b：帧类型（4bit）；1是关键帧，IDR帧(不一定是I真) 2是inter frame，P帧或者B帧，可参考[维基](https://en.wikipedia.org/wiki/Inter_frame)
- 5-8b：CodecID（4bit）7是AVC
- 7-7b：采样的大小（1bit）（每个采样的大小，属于未压缩的格式，0位8bit采样，1位16bit采样）
- 8-xb：当CodecID是7时，表示AVCVIDEOPACKET




####2.5.2 AVCVIDEOPACKET


![](/Users/leegend/Desktop/tmppic/ae4cc7814984a2311149a48e62a9ca14.png)

- 1-1B：包类型（1B）；
- 2-4B： CTS， [ISO_IEC_14496-15](http://read.pudn.com/downloads174/sourcecode/multimedia/mpeg/810862/ISO_IEC_14496-15-2004.pdf)中有一段介绍CTS
- 5-xB：


>关于CTS：这是一个比较难以理解的概念，需要和pts，dts配合一起理解。 首先，pts（presentation time stamps），dts(decoder timestamps)，cts(CompositionTime)的概 念： pts：显示时间，也就是接收方在显示器显示这帧的时间。单位为1/90000 秒。 dts：解码时间，也就是rtp包中传输的时间戳，表明解码的顺序。单位单位为1/90000 秒。——根据 后面的理解，pts就是标准中的CompositionTime cts偏移：cts = (pts ­ dts) / 90 。cts的单位是毫秒。 pts和dts的时间不一样，应该只出现在含有B帧的情况下，也就是profile main以上。baseline是没有 这个问题的，baseline的pts和dts一直想吐，所以cts一直为0。 在flv tag中的时戳就是DTS。 研究 一下文档，  ISO/IEC 14496­12:2005(E)      8.15   Time to Sample Boxes，发现 CompositionTime就是presentation time stamps，只是叫法不同。——需要再进一步确认。 在上图中，cp就是pts，显示时间。DT是解码时间，rtp的时戳。 I1是第一个帧，B2是第二个，后面的序号就是摄像头输出的顺序。决定了显示的顺序。 DT，是编码的顺序，特别是在有B帧的情况，P4要在第二个解，因为B2和B3依赖于P4，但是P4的 显示要在B3之后，因为他的顺序靠后。这样就存在显示时间CT(PTS)和解码时间DT的差，就有了CT 偏移。 P4解码时间是10，但是显示时间是40，

####2.5.3 AVCDecoderConfigurationRecord
AVC sequence header

文件中AVCsequenceheader只有一个，出现在第一个video tag。为了能够从FLV文件中获取NALU，必须要知道前面的NALU长度字段所占的字节数（通常是1、2或4个字节），这个内容必须要从AVCDecoderConfigurationRecord中获取，这个遵从标准[ISO/IEC 14496-15中的5.2.4小节](http://read.pudn.com/downloads174/sourcecode/multimedia/mpeg/810862/ISO_IEC_14496-15-2004.pdf)。

AVCDecoderConfigurationRecord结构如下：
![](/Users/leegend/Desktop/tmppic/19b92a7de3228844cc1601accceb61bb.png)

其中：

- AVCProfileIndication;     //sps[1]，即0x67后面那个字节   
- profile_compatibility;     //sps[2]  
- AVCLevelIndication;      //sps[3]    
- lengthSizeMinusOne;     //NALUnitLength的长度减1，一般为3   
- numOfSequenceParameterSets;    //sps个数，一般为1   
- sequenceParameterSetLength;  //sps的长度 
- numOfPictureParameterSets;      //pps个数，一般为1   
- pictureParameterSetLength;   //pps长度     

lengthSizeMinusOne加1就是NALU长度字段所占字节数
![](/Users/leegend/Desktop/tmppic/59fe440be33f9dec94c61eba586fa27e.png)


###2.6元信息

http://blog.csdn.net/leixiaohua1020/article/details/17934487
关于AMF记录的 控制帧信息

维基百科的

flv文件中的元信息，是一些描述flv文件各类属性的信息。这些信息以AMF格式保存在文件的起始部分。adobe官方的标准flv元信息项目如下[5]：
duration
width
height
videodatarate
framerate
videocodecid
audiosamplerate
audiosamplesize
stereo
audiocodecid
filesize

///////
audiochannels

audiodatarate
audiodevice
audioinputvolume

creationdate
(media files only)
fmleversion (Flash Media Live Encoder version)（media files only）


lastkeyframetimestamp (media files only)
lasttimestamp (media files only)
presetname


videodevice
videokeyframe_frequency

---------
     
参考：
https://wenku.baidu.com/view/cfb74db08bd63186bcebbced.html
http://www.360doc.com/content/16/0624/22/9075092_570514283.shtml
http://niulei20012001.blog.163.com/blog/static/7514721120130694144813/
http://blog.jianchihu.net/flv-aac-add-adtsheader.html
http://www.lukas.cl/lukas_eng/site/artic/20150723/asocfile/20150723184649/aacdecoder.pdf
http://blog.jianchihu.net/flv-aac-add-adtsheader.html
http://read.pudn.com/downloads176/sourcecode/windows/multimedia/817042/14496-10.pdf
http://read.pudn.com/downloads174/sourcecode/multimedia/mpeg/810862/ISO_IEC_14496-15-2004.pdf