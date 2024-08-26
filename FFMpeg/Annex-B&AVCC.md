## AneexB & AVCC

典型问题
iOS 硬解264视频（MP4），出现绿屏，或上半部分正常下半部分绿屏。
iOS 硬解265视频，同样也要解决的extradata处理问题。

参：iOS11 VideoToolbox硬解HEVC

首先来看两种格式：

#### Annex-B 和 AVCC/HVCC

H.264码流分Annex-B和AVCC两种格式。
H.265码流是Annex-B和HVCC格式。

(以下内容针对H.264，但大体也适用于H.265/HEVC)

##### 别名

AVCC格式 也叫AVC1格式，MPEG-4格式，字节对齐，因此也叫Byte-Stream Format。用于mp4/flv/mkv, VideoToolbox。
Annex-B格式 也叫MPEG-2 transport stream format格式（ts格式）, ElementaryStream格式。
Annex-B 附录B, 指ITU-T的 Recommendation（h.264和h.265）在附录B中规定码流格式。

##### Annex-B&AVCC结构上的区别：

区别有两点：

一、参数集(SPS, PPS)组织格式

二、分隔

- Annex-B：使用start code分隔NAL(start code为三字节或四字节，0x000001或0x00000001，一般是四字节)； SPS 和 PPS 在码流中分别作为一个 NALU，nalu header->nal_unit_type=7时，nalu payload中是SPS，nalu header->nal_unit_type=8时，nalu payload中是PPS。这种格式下SPS、PPS通常放在关键帧之前。
- AVCC：使用NALU长度（固定字节，通常为4字节）分隔NAL；在头部包含extradata(或sequence header)的结构体。这种格式下SPS/PPS的存放位置和帧数据无关。例如普通模式下的mp4，SPS、PPS放在MOOV中，在帧数据后面.

**SPS（Sequence Paramater Set）**又称作序列参数集。定义了视频序列的总体参数，例如帧率、分辨率、码率等等。

**PPS（Picture Parameter Set）**又称作图像参数集。定义了每个图像的具体参数，例如量化参数、熵编码模式等等。

**VPS（Video Parameter Set）**又称视频参数集。H.265中增加，它为多个 SPS 和 PPS 提供了统一的参数设置，以提高编码效率和灵活性。

**Slice**一幅图像可以被分割为一个或多个片( Slice),每个片的压缩数据都是独立的, Slice头信息无法通过前一个Sice的头信息推断得到。这就要求 Slice不能跨过它的边界来进行帧内或帧间预测,且在进行熵编码前需要进行初始化。但在进行**环路滤波**时,允许滤波器**跨越 Slice的边界**进行滤波。除了 Slice的边界可能受环路滤波影响外, Slice的解码过程可以不使用任何来自其他 Slice的影响,且有利于实现并行运算。**使用 Slice的主要目的是当数据丢失后能再次保证解码同步。**

> **① I Slice:**该 Slice中所有CU的编码过程都使用帧内预测。
> **② P Slice:**在 I Slice的基础上,该 Slice中的CU还可以使用帧间预测测,每个预测块(PB)使用至多一个运动补偿预测信息。 P Slice只使用图像参考列表list0
> **③ B Slice:**在 P Slice的基础上, B Slice中的CU也可以使用帧间预测,但是每个PB可以使用至多两个运动补偿预测信息。 B Slice可以使用图像参考列表list0和list1。

**Tile **H.265/HEVC对H.264/AVC的改进之处还在于Tile概念的提出。一幅图像不仅可以划分为若干个 Slice,也可以划分为若干个Tile。即从水平和垂直方向将一幅图像分割为若干个矩形区域,一个矩形区域就是一个Tile。每个Tile包含整数个CTU,其可以独立解码。划分Tile的主要目的是在增强并行处理能力的同时又不引入新的错误扩散。Tile提供比CTB更大程度的并行(在图像或者子图像的层面上),在使用时无须进行复杂的线程同步



> 在 extradata 中，SPS 和 PPS 的作用是为解码器提供视频序列的配置信息，以确保解码器能够正确地解释和处理视频数据。通过提供这些参数集，解码器能够准确地还原视频序列的特性，从而实现高质量的视频解码。



3.2.1 Annex B

3.2.2 extradata
H.264/AVC extradata 语法
参考：《ISO/IEC 14496-15 NAL unit structured video》AVCDecoderConfigurationRecord结构：（最小长度7字节）


说明：FFmpeg中，extradata解析，见ff_h264_decode_extradata()

注意：第5字节的最后2位，表示的就是NAL size的字节数。在AVCC格式中，每个NAL前面都会有NAL size字段。NAL size可能是1字节、2字节或4字节（4字节较常见），解析extradata重要目的就是确认这个值。（而Annex-B格式，要split NAL，只要去探测0x000001就可以了）

H.264 extradata 示例（AVCC格式）

```
0x0000 | 01 64 00 1E FF E1 00 1F 67 64 00 1E AC C8 60 33  // E1: 1SPS  00 1F: SPS 31byte
0x0010 | 0E F9 E6 FF C1 C6 01 C4 44 00 00 03 00 04 00 00 
0x0020 | 03 00 B8 3C 58 B6 68 01 00 05 68 E9 78 47 2C     // 01: 1PPS  00 05: PPS 5byte

```

```
bits      
8   version ( always 0x01 )  
8   avc profile ( sps[0][1] )  
8   avc compatibility ( sps[0][2] )  
8   avc level ( sps[0][3] )  
6   reserved ( all bits on )      // 即 0xFC | current byte
2   NALULengthSizeMinusOne        // 前缀长度-1 
3   reserved ( all bits on )      // 即 0xE0 | currrent byte
5   number of SPS NALUs (usually 1)  
    -- repeated once per SPS --  
16  SPS size  
N   variable SPS NALU data  
8   number of PPS NALUs (usually 1)  
    -- repeated once per PPS --
16  PPS size  
N   variable PPS NALU data  
```


H.265/HEVC extradata语法
参照HEVCDecoderConfigurationRecord：（最小长度23字节）


HEVC extradata 示例

 extradata    如上
 extrasize     111
 24 | 20           NAL type:  VPS
 25 | 00 01      VPS num:   1
 27 | 00 19      VPS size:  25字节
 54 | 21            NAL type:  SPS
 55 | 00 01      SPS num:   1
 57 | 00 29      SPS size:  41字节
100| 22          NAL type:  PPS

hvcC extradata是一种头描述的格式。而annex-b格式中，则是将VPS, SPS和PPS等同于普通NAL，用start code分隔，非常简单。Annex-B格式的”extradata”：
start code+VPS+start code+SPS+start code+PPS

实践
VideoToolbox 与 AVCC格式
硬解 仅支持avcC格式。 如ES格式，需要转为MPEG-4格式 P58
硬编 输出avcC格式。 P204

MediaCodec 与 Annex-B格式
硬解 支持Annex-B格式，avcC需要做转换，NALU长度替换为start code

Annex-B 转 AVCC 
对于仅接受AVCC格式的播放器(如Quicktime v7.0)，需要进行convert Annex-B to AVCC:

- start code 转为4字节 NAL size
- SPS, PPS创建 extradata

AVCC 转 Annex-B
FFmpeg “extract_extradata” bitstream filter: 3
h264码流转换：
ffmpeg -i INPUT.mp4 -codec copy -bsf:v h264_mp4toannexb OUTPUT.ts
hevc码流转换：
ffmpeg -i INPUT.mp4 -codec copy -bsf:v hevc_mp4toannexb OUTPUT.ts

后续
了解了H.264 extradata以及NAL组织结构，自然引出H.264码流结构的议题，下篇干脆系统分析下H.264, HEVC码流结构。



https://blog.csdn.net/yue_huang/article/details/75126155

https://blog.csdn.net/qq_35585843/article/details/129183886
