## SRT实时传输协议
Secure Reliable Transport(SRT)是安全、可靠、低延时的多媒体实时传输协议。SRT协议继承自UDT协议，包括协议设计和代码库。UDT是基于UDP的文件传输协议，最初是针对高带宽、高延迟场景（如远距离光纤传输）设计，用于弥补TCP的不。SRT协议使用AES进行数据加密，运用FEC进行前向纠错，并且有流量控制、拥塞控制。类似于QUIC协议，SRT采用UDP代替TCP，在应用层提供发送确认机制、ARQ自动重传，减少端到端的延迟。

SRT探测实时网络带宽状况，有利于补偿网络拥塞引起的jitter网络抖动和带宽下降。为了实现低延迟码流传输，SRT协议会携带delay、jitter、丢包等信息。SRT提供多路复用机制，允许多个请求共享相同的端口。

### 一、数据包结构

SRT的数据包分为data packet和control packet两种类型，结构如下：

### 二、Handshake

##### 1、握手数据包结构

Handshake握手属于控制包类型(control type=0x0000)，用于交换双方配置信息、连接参数。

##### 2、加密算法

Encryption Field包括AES的128位、192位和256位

##### 3、握手类型

握手类型占32位

### 三、Keep-Alive

Keep-Alive属于控制包，在上一次数据包超时后发送，用于心跳保活

### 四、ACK于NAK

##### 1、ACK确认应答

Acknowledgment(ACK) 属于控制包，用于表示数据包到达确认。所有ACK包可能会携带额外信息，比如RTT、链路容量、传输速率。其中，ACK包类型包括：

- Full ACK：每10ms发送一次
- Light ACK：仅保护应答包序号
- Small ACK：可用缓冲区大小

##### 2、NAK否定应答

Negative Acknowlegement(NAK)也是属于控制包，用于表示否定应答(没有收到数据包)。与TCP的NACK否定应答不同的是，SRT采用NAK表示否定应答。NAK控制包结构如下：

### 五、丢包请求

Message Drop Request用于丢包请求，比如TTL触发一个或多个数据包重传超时。发送端发起丢包请求，告诉接收端这些数据包要丢弃。

### 六、拥塞控制

 拥塞控制包括两种类型：Live Congestion Control(LiveCC)和File Congestion Control(FileCC)。

##### 1、慢启动

Slow Start，慢启动阶段，拥塞窗口CWND_SIZE初始化为16。拥塞窗口上限为MAX_CWND_SIZE，最大接收缓冲区为12MB。

ACK包的经历步骤如下：

如果上次发送速率的间隔小于RC_INTERVAL，保持当前发送速率
更新lastRCTime: lastRCTime = currTime
更新拥塞窗口大小：CWND_SIZE += ACK_SEQNO - LAST_ACK_SEQNO
更新上次ACK序号：LAST_ACK_SEQNO = ACK_SEQNO
如果CWND_SIZE > MAX_CWND_SIZE，进入拥塞避免阶段
如果接收到NAK包，或者RTO重传超时，也会结束慢启动，进入拥塞避免。

##### 2、拥塞避免

拥塞避免阶段，收到ACK包经历的步骤如下：

如果上次发送速率的间隔小于RC_INTERVAL，保持当前发送速率
更新lastRCTime: lastRCTime = currTime
更新拥塞窗口大小:CWND_SIZE = RECEIVE_RATE * (RTT + RC_INTERVAL) / 1000000 + 16
如果发生丢包，PKT_SND_PERIOD保持不变，bLoss置为False
如果没有丢包，更新PKT_SND_PERIOD
如果设置最大带宽MAX_BW，则限制PKT_SND_PERIOD为最小允许周期

### 七、Shutdown

Shutdown控制包，用于表示断开连接。



### **SRT的功能特性**

- 两种传输模式：file文件传输模式和live直播流模式；
  这里我们主要关注live模式。live模式既支持loss模式也支持lossless无损模式。
- SRT作为传输协议，可以使用任意流媒体封装格式；
  但要注意，loss模式要求容器格式必须有错误恢复resync机制，可选范围基本只剩下TS格式或者H.264、annexb之类的裸流。
- 双向传输；
  这一能力使得SRT在一些场景可以替换TCP，比如用SRT做RTMP的传输层。但要注意，SRT设计上是假设双向传输的流彼此无依赖关系，而RTMP存在双向的信令交互，RTMP over SRT模式会出现一些非常隐蔽的缺陷，见后文详述。
- 支持ARQ和FEC两种丢包恢复机制；
- 支持connection bonding，目前处于实验阶段。

**固定延迟**

SRT最突出的特性是端到端固定延迟（Fixed end-to-end latency）。一般的传输协议，从一端send()到另一端receive()所占用的时间是波动的，SRT抹平了网络的抖动，可以保证从srt_sendmsg()到srt_recvmsg()的时间基本恒定。srt_sendmsg()给数据打一个时间戳，在接收端检查时间戳，决定什么时候可以把数据交给上层应用。核心思想并不复杂：buffer抗抖动，根据时间戳保持固定延迟。具体实现还考虑了time drift问题。端到端延迟约等于配置的SRTO_RECVLATENCY + 1/2 RTT0，RTT0是握手阶段计算得到的RTT。**配置的latency要足够补偿RTT的波动，同时考虑丢包重传次数**。

传输层的固定延迟可以简化接收端消费者的缓冲设计。以播放器为例，播放器一般有多级缓冲，IO模块内或IO到demux有缓冲，解码后到渲染也有缓冲队列，而核心的缓冲模块一般放在demux后解码前，原因是可以计算缓冲数据的时长。解码后到渲染前也可以计算时长，但解码后的数据量大，不适合缓冲较长时间数据，不能控制缓冲水位。IO模块做到固定延迟缓冲，播放器可以省去IO到demux、demux到解码的缓冲模块，只需要简单的数据管理。多级缓冲设计延迟大，IO模块的缓冲区是整个播放器缓冲的一小部分，丢包恢复能力弱。当把缓冲区交给传输模块管理后，丢包恢复能力增强。



**弱网抗性**

SRT支持ARQ和FEC两种抗弱网机制。另外，Link bonding中的broadcast模式也能提高数据传输的可靠性。

#### ARQ

SRT的ARQ设计同时使用了ACK和NACK两种机制。接收端在两种情况下会发送ACK：

- 每10 ms发送一次全量ACK（full ACK）；
- 每收到64个包发送一个轻量ACK（light ACK）。

轻量ACK是针对高码率场景考虑的：高码率下，10 ms对应的包太多了，ACK不及时则发送端不能释放已发送的数据。

除了通常的ACK、NACK包之外，SRT还有ACKACK包：

- 发送端收到ACK后立即发送ACKACK，中间不做delay；接收端把发送ACK到收到ACKACK的时间作为RTT；
- 接收端收到ACKACK后，停止重发同一个ACK包，否则会多次发送。

用于测量RTT时，可以把ACK看做ping，ACKACK看成pong。

接收端有两种情况会发送NACK：

- 收到的包sequence number有空洞
- 周期性检查丢包情况并发送NACK，时间间隔为max( (RTT+4*RTTVar)/2, 20)。

周期性NACK可能导致一个包不只一次被重发，另一方面保证了低延迟的特性。



### **拥塞控制/Pacing**

SRT file模式有拥塞控制，live模式只有Pacing控制，没有拥塞控制。

SRT的Pacing是根据最大发送带宽来计算发包的时间间隔。最大发送带宽由三种策略确定：

1. 手动配置最大发送带宽max_bw；
2. 根据输入码率和overhead，计算max_bw = input_bw * (1 + overhead)；
3. 配置overhead，自动估计输入带宽，max_bw = input_bw_estimate * (1+ overhead)。
