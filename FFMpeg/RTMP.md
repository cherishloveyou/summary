# RTMP协议详解

RTMP(Real Time Messaging Protocol)是一个应用层协议，靠底层可靠的传输层协议（通常是TCP）来保证信息传输的可靠性的，用来解决多媒体数据传输流的多路复用（Multiplexing）和分包（packetizing）的问题。该协议的突出优点是: 低延时。

### 1. RTMP连接

首先进行tcp三次握手，通过TCP三次握手，可实现RTMP客户端与RTMP服务器的指定端口(默认端口为`1935`)建立一个可靠的网络连接。

> TCP三次握手->RTMP握手->连接->创建流->播放->删除流

建立一个有效的RTMP Connection链接，首先要“ 握手”:

1.  校验客户端和服务器端RTMP协议版本号
2. 是发了一堆随机数据，校验网络状况。

客户端要向服务器发送C0,C1,C2（按序）三个chunk，服务器向客户端发送S0,S1,S2（按序）三个chunk，然后才能进行有效的信息传输。RTMP协议本身并没有规定这6个Message的具体传输顺序，但RTMP协议的实现者需要保证这几点：
客户端要等收到S1之后才能发送C2
客户端要等收到S2之后才能发送其他信息（控制信息和真实音视频等数据）
服务端要等到收到C0之后发送S1
服务端必须等到收到C1之后才能发送S2
服务端必须等到收到C2之后才能发送其他信息（控制信息和真实音视频等数据）

** RTMP Message & 分块（chunk）**

RTMP 传输的数据称为Message，Message包含音视频数据和信令，传输时不是以Message为单位的，而是把Message拆分成Chunk发送，而且必须在一个Chunk发送完成之后才能开始发送下一个Chunk，每个Chunk中带有msg stream id代表属于哪个Message，接受端也会按照这个id来将chunk组装成Message。每个Chunk的默认大小是 128 字节，可以通过Set Chunk Size的控制信息设置Chunk数据量的最大值。通用的做法，RTMP Message Header不拆分到chunk data中，虽然规范上RTMP massage应该作为一个整体被拆分成chunk，但是由于RTMP massage header与chunk massage header信息重复，本着最小传输数据原则，一般做法是在chunk data中去掉此信息。

RTMP消息有两部分，消息头（Message Header）和有效负载（Message body）。

```c
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 | Message Type |               Payload length                   |
 |   (1 byte)   |                   (3 bytes)                    |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                          Timestamp                            |
 |                          (4 bytes)                            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                          Stream ID            |
 |                          (3 bytes)            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                         Message Header
```

Chunk格式包含基本头、消息头、扩展时间戳和负载

```text
 +--------------+----------------+--------------------+--------------+
 | Basic Header | Message Header | Extended Timestamp |  Chunk Data  |
 +--------------+----------------+--------------------+--------------+
 |                                                    |
 |<------------------- Chunk Header ----------------->|
                            Chunk Format
```

为什么RTMP要将Message拆分成不同的Chunk呢？通过拆分，数据量较大的Message可以被拆分成较小的“Message”，这样就可以避免优先级低的消息持续发送阻塞优先级高的数据。

**消息分类**
消息主要分为三类: **协议控制消息、数据消息、命令消息**等。

协议控制消息
Message Type ID = 1 2 3 5 6和Message Type ID = 4两大类，主要用于协议内的控制
数据消息
Message Type ID = 8 9 18
8: Audio 音频数据
9: Video 视频数据
18: Metadata 包括音视频编码、视频宽高等信息。
命令消息
Command Message ID = 20 17
此类型消息主要有NetConnection和NetStream两个类。
Message被拆分成一个或多个Chunk，然后在网络上一个接一个的进行发送。
默认Chunk Size是128字节



**1）RTMP 消息分优先级的设计有什么好处？**

RTMP 的消息优先级是：控制消息 > 音频消息 > 视频消息。当网络传输能力受限时，优先传输高优先级消息的数据。要使优先级能够有效执行，分块也很关键：将大消息切割成小块，可以避免大的低优先级的消息（如视频消息）堵塞了发送缓冲从而阻塞了小的高优先级的消息（如音频消息或控制消息）。

**2）什么是 DTS 和 PTS？它们有什么区别？**

DTS 是[解码时间戳](https://www.zhihu.com/search?q=解码时间戳&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A654279235})；PTS 是显示时间戳。

虽然 DTS、PTS 是用于指导播放端的行为，但它们是在编码的时候由编码器生成的。

当视频流中没有 B 帧时，通常 DTS 和 PTS 的顺序是一致的。但如果有 B 帧时，就回到了我们前面说的问题：解码顺序和播放顺序不一致了。DTS 告诉我们该按什么顺序解码这几帧图像，PTS 告诉我们该按什么顺序显示这几帧图像。

**3）什么是 IDR 帧？它和 I 帧有什么区别？**

[IDR 帧](https://www.zhihu.com/search?q=IDR 帧&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A654279235})全称叫做 Instantaneous Decoder Refresh，是 I 帧的一种。IDR 帧的作用是立刻刷新，重新算一个新的序列开始编码，使错误不致传播。

IDR 帧有如下特性：

- IDR 帧一定是 I 帧，严格来说 I 帧不一定是 IDR 帧（但一般 I 帧就是 IDR 帧）；
- 对于 IDR 帧来说，在 IDR 帧之后的所有帧都不能引用任何 IDR 帧之前的帧的内容。与此相反，对于普通的 I 帧来说，位于其之后的 B 和 P 帧可以引用位于普通 I 帧之前的 I 帧（普通 I 帧有被跨帧参考的可能）；
- 播放器永远可以从一个 IDR 帧播放，因为在它之后没有任何帧引用之前的帧。因此，视频开头的 I 帧一定是 IDR 帧；一个封闭类 GOP 的开头的 I 帧也一定是 IDR 帧。





- RTMP的容器格式FLV，存在不支持新的codec、不支持多音轨、时间戳精度过低等等缺陷；
- RTMP基于TCP做传输，TCP的公平、可靠传输设计并不适用于实时音视频传输。
