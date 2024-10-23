# 花屏/绿屏

直播时有时会遇到花屏或绿屏的现象，都有那些原因会导致这种现象呢?
 我梳理了部分原因:

![img](https:////upload-images.jianshu.io/upload_images/1720840-16f17906761dd1b9.png?imageMogr2/auto-orient/strip|imageView2/2/w/955/format/webp)

视频直播花屏&绿屏 原因

#### 花屏

花屏主要分为整个画面都花屏或局部花屏两种情况。

##### 全屏花屏

- 正常花屏

> 有一种花屏是正常的，就是码率特别低的时候出现的大面积马赛克，我们俗称"画面糊了"。
>  比如我们告诉视频编码器**要输出1280\* 720高清分辨率的画面，但同时要求它只用 200 kbps的码率**（码率是指编码器每秒产生的视频数据大小 ），编码器此时能做的事情就是无底线地降低画质，就会导致大面积的马赛克。

- 视频参数问题
   比如当视频源修改过视频参数(如从720P修改1080P),此时客户端用于解码的SPS&PPS如果没有重新获取的话，就会出现整个画面花屏的现象。这种花屏的现象会一直持续下去，不会随着时间而恢复正常画面。

  ![img](https:////upload-images.jianshu.io/upload_images/1720840-4a23ee19cbbe4560.png?imageMogr2/auto-orient/strip|imageView2/2/w/1158/format/webp)

  视频参数变化 导致全屏花屏或绿屏

##### 局部花屏

- SO_SNDBUF的Buffer太小
   当流媒体服务器的SO_SNDBUF的Buffer太小, 在网络环境不好时，导致部分直播数据丢失(比如丢失P帧)，继而会导致部分花屏的现象。

解决方法:
 **增加SO_SNDBUF的Buffer大小**。

```cpp
    SOCKET sSocket = ...
    ...
    int nRcvBufferLen = 1024*1024;
    int nSndBufferLen = 4*1024*1024;
    int nLen          = sizeof(int);

    setsockopt(sSocket, SOL_SOCKET, SO_SNDBUF, (char*)&nSndBufferLen, nLen);
    setsockopt(sSocket, SOL_SOCKET, SO_RCVBUF, (char*)&nRcvBufferLen, nLen);
```

- P帧丢失
   I帧正常丢失P帧的情况下，画面的大部分区域是正常的，只有在发生变化的那部分区域会存在局部花屏。

#### 绿屏

产生绿屏的主要是: 无法渲染的画面有些用黑色填充，有些用绿色填充，有些用上一帧画面填充。
 视频参数改变， 而解码端的SPS&PPS信息未及时重新获取更新，会导致画面无法正常渲染，继而导致绿屏的现象出现





[音视频重要文章和视频](https://www.jianshu.com/u/b971a1d12a6f)
