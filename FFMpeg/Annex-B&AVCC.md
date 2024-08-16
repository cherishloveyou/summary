## FFMpeg一些命令

2.典型问题
iOS 硬解264视频（MP4），出现绿屏，或上半部分正常下半部分绿屏。
iOS 硬解265视频，同样也要解决的extradata处理问题。
参：iOS11 VideoToolbox硬解HEVC

首先来看两种格式：

3.Annex-B 和 AVCC/HVCC
H.264码流分Annex-B和AVCC两种格式。
H.265码流是Annex-B和HVCC格式。

(以下内容针对H.264，但大体也适用于H.265/HEVC)

3.1别名
AVCC格式 也叫AVC1格式，MPEG-4格式，字节对齐，因此也叫Byte-Stream Format。用于mp4/flv/mkv, VideoToolbox。
Annex-B格式 也叫MPEG-2 transport stream format格式（ts格式）, ElementaryStream格式。
Annex-B 附录B, 指ITU-T的 Recommendation（h.264和h.265）在附录B中规定码流格式。

3.2 结构上的区别：
区别有两点：一个是参数集(SPS, PPS)组织格式；一个是分隔。
- Annex-B：使用start code分隔NAL(start code为三字节或四字节，0x000001或0x00000001，一般是四字节)；SPS和PPS按流的方式写在头部。
- AVCC：使用NALU长度（固定字节，通常为4字节）分隔NAL；在头部包含extradata(或sequence header)的结构体。（extradata包含分隔的字节数、SPS和PPS，具体结构见下）

3.2.1 Annex B

3.2.2 extradata
H.264/AVC extradata 语法
参考：《ISO/IEC 14496-15 NAL unit structured video》AVCDecoderConfigurationRecord结构：（最小长度7字节）


说明：FFmpeg中，extradata解析，见ff_h264_decode_extradata()

注意：
第5字节的最后2位，表示的就是NAL size的字节数。在AVCC格式中，每个NAL前面都会有NAL size字段。NAL size可能是1字节、2字节或4字节（4字节较常见），解析extradata重要目的就是确认这个值。（而Annex-B格式，要split NAL，只要去探测0x000001就可以了）

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
 24| 20           NAL type:  VPS
 25| 00 01      VPS num:   1
 27| 00 19      VPS size:  25字节
 54| 21           NAL type:  SPS
 55| 00 01      SPS num:   1
 57| 00 29      SPS size:  41字节
100| 22         NAL type:  PPS
1
2
3
4
5
6
7
8
9
hvcC extradata是一种头描述的格式。而annex-b格式中，则是将VPS, SPS和PPS等同于普通NAL，用start code分隔，非常简单。Annex-B格式的”extradata”：
start code+VPS+start code+SPS+start code+PPS

3.3 实践
VideoToolbox 与 AVCC格式 1
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

4. 后续
了解了H.264 extradata以及NAL组织结构，自然引出H.264码流结构的议题，下篇干脆系统分析下H.264, HEVC码流结构。



https://blog.csdn.net/yue_huang/article/details/75126155

https://blog.csdn.net/qq_35585843/article/details/129183886
