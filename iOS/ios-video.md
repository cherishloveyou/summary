# iOS音视频

1、数据采集(音频/视频) AVFoundation

2、美颜/滤镜



**在直播项目中,一般常见有8个步骤.**

- 音视频采集
- 视频滤镜
- 音视频编码
- 推流
- 流媒体服务器处理
- 拉流
- 音视频解码
- 音视频播放

这个在开发者面试一些有意向或者目前业务中包含直播需求的公司,最为常见的面试题.不管在我们过往的工作经验是否有直播或音视频相关经验.这个一块都是你必须能了解.希望大家可以简单的了解.

### 2.2 相关框架的学习与使用场景

![img](https:////upload-images.jianshu.io/upload_images/4624551-9129a8b51add1a96.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)



- 采集视频,音频
  - 使用iOS原生框架 **`AVFoundation.framework`**
- 视频滤镜处理
  - 使用iOS原生框架 **`CoreImage.framework`**
  - 使用第三方框架 **`GPUImage.framework`**

> **`CoreImage` 与 `GPUImage` 框架比较:**
> 在实际项目开发中,开发者更加倾向使用于`GPUImage`框架.
> 首先它在使用性能上与iOS提供的原生框架,并没有差别;其次它的使用便利性高于iOS原生框架,最后也是最重要的`GPUImage`框架是开源的.而大家如果想要学习`GPUImage`框架,建议学习`OpenGL ES`,其实`GPUImage`的封装和思维都是基于`OpenGL ES`.

- **视频\音频编码压缩**
  - 硬编码
    - 视频: `VideoToolBox`框架
    - 音频: `AudioToolBox` 框架
  - 软编码
    - 视频: 使用`FFmpeg`,`X264`算法把视频原数据YUV/RGB编码成H264
    - 音频: 使用`fdk_aac` 将音频数据PCM转换成AAC
- **推流**
  - 推流: 将采集的音频.视频数据通过流媒体协议发送到流媒体服务器
  - 推流技术
    - 流媒体协议: RTMP\RTSP\HLS\FLV
    - 视频封装格式: TS\FLV
    - 音频封装格式: Mp3\AAC
- **流媒体服务器**
  - 数据分发
  - 截屏
  - 实时转码
  - 内容检测
- **拉流**
  - 拉流: 从流媒体服务器中获取音频\视频数据
  - 流媒体协议: RTMP\RTSP\HLS\FLV
- **音视频解码**
  - 硬解码
    - 视频: `VideoToolBox`框架
    - 音频: `AudioToolBox` 框架
  - 软解码
    - 视频: 使用`FFmpeg`,`X264`算法解码
    - 音频: 使用`fdk_aac` 解码
- **播放**
  - `ijkplayer` 播放框架
  - `kxmovie` 播放框架



[H264](https://baike.baidu.com/item/H.264/1022230?fromtitle=H264)是一种视频编码标准，H264协议里定义了三种帧：

- I帧：完整编码的帧，也叫关键帧
- P帧：参考之前的I帧生成的只包含差异部分编码的帧
- B帧：参考前后的帧编码的帧叫B帧

H264采用的核心算法是帧内压缩和帧间压缩，帧内压缩是生成I帧的算法，帧间压缩是生成B帧和P帧的算法。
H264原始码流是由一个接一个的NALU（Nal Unit）组成的，NALU = 开始码 + NAL类型 + 视频数据
开始码用于标示这是一个NALU 单元的开始，必须是"00 00 00 01" 或"00 00 01"
NALU类型如下：

| 类型    | 说明                             |
| ------- | -------------------------------- |
| 0       | 未规定                           |
| 1       | 非IDR图像中不采用数据划分的片段  |
| 2       | 非IDR图像中A类数据划分片段       |
| 3       | 非IDR图像中B类数据划分片段       |
| 4       | 非IDR图像中C类数据划分片段       |
| 5       | IDR图像的片段                    |
| 6       | 补充增强信息（SEI）              |
| 7       | 序列参数集（SPS）                |
| 8       | 图像参数集（PPS）                |
| 9       | 分割符                           |
| 10      | 序列结束符                       |
| 11      | 流结束符                         |
| 12      | 填充数据                         |
| 13      | 序列参数集扩展                   |
| 14      | 带前缀的NAL单元                  |
| 15      | 子序列参数集                     |
| 16 – 18 | 保留                             |
| 19      | 不采用数据划分的辅助编码图像片段 |
| 20      | 编码片段扩展                     |
| 21 – 23 | 保留                             |
| 24 – 31 | 未规定                           |

一般我们只用到了1、5、7、8这4个类型就够了。类型为5表示这是一个I帧，I帧前面必须有SPS和PPS数据，也就是类型为7和8，类型为1表示这是一个P帧或B帧。

**帧率：**单位为fps(frame pre second)，视频画面每秒有多少帧画面，数值越大画面越流畅
**码率：**单位为bps(bit pre second)，视频每秒输出的数据量，数值越大画面越清晰
**分辨率：**视频画面像素密度，例如常见的720P、1080P等
**关键帧间隔：**每隔多久编码一个关键帧
**软编码：**使用CPU进行编码。性能较差
**硬编码：**不使用CPU进行编码，使用显卡GPU,专用的DSP、FPGA、ASIC芯片等硬件进行编码。性能较好



### 硬编码步骤

 1、通过VTCompressionSessionCreate创建编码器
 2、通过VTSessionSetProperty设置编码器属性
 3、设置完属性调用VTCompressionSessionPrepareToEncodeFrames准备编码
 4、输入采集到的视频数据，调用VTCompressionSessionEncodeFrame进行编码
 5、获取到编码后的数据并进行处理
 6、调用VTCompressionSessionCompleteFrames停止编码器
 7、调用VTCompressionSessionInvalidate销毁编码器



1、PTS（Presentation Time Stamp）：即显示时间戳，这个时间戳用来告诉播放器该在什么时候显示这一帧的数据。
2、DTS（Decoding Time Stamp）：即解码时间戳，这个时间戳的意义在于告诉播放器该在什么时候解码这一帧的数据。



[iOS视频开发](https://www.jianshu.com/p/eccdcf43d7d2)

https://juejin.cn/post/6844903549587947527

https://juejin.cn/post/6844903853775650824

https://juejin.cn/post/6844903888638705678
