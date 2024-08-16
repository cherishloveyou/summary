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

**1.1 RTMP Message & 分块（chunk）**

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


**1.2 消息分类**
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



- RTMP的容器格式FLV，存在不支持新的codec、不支持多音轨、时间戳精度过低等等缺陷；
- RTMP基于TCP做传输，TCP的公平、可靠传输设计并不适用于实时音视频传输。
