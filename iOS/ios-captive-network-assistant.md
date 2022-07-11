# Captive Network Assistant



**【正常iOS系统点亮WiFi图标过程】**

- 连上WiFi后iOS会自动发起探测帧：http://captive.apple.com/hotspot-detect.html
- 首先DNS解析该域名，然后自动发送一个HTTP/1.0的探测帧请求到http://captive.apple.com/hotspot-detect.html
- 终端接收到苹果服务器探测回应，如果回应报文头部为success，那么认为网络是通的，同时，状态栏的WIFI图标出现，流程结束。

**【正常iOS系统自动弹出portal流程】**

- 连上WiFi后iOS会自动发起探测帧：http://captive.apple.com/hotspot-detect.htm
- 首先DNS解析该域名，然后自动发送一个HTTP/1.0的探测帧请求到 http://captive.apple.com/hotspot-detect.html
- 终端接收到探测回应，回应报文头部不是success，不点亮WiFi图标。
- 终端会自动打开一个页面，在这个页面中再请求一次http://captive.apple.com/hotspot-detect.html，这一次，使用的是HTTP/1.1。
- 此时我们控制器会使用苹果服务器IP给终端回复一个302 moved跳转到[http://auth.wifi.com](http://auth.wifi.com/)
- 接下来便进入到portal页面及认证流程并点亮WiFi图标，流程结束。

**【部分iOS WiFi图标无法点亮、portal页面弹出缓慢原因】**

​      目前出现的苹果终端WiFi图标无法点亮、portal页面弹出缓慢的问题，从问题终端抓包分析，该终端在走到正常流程第三步后，没有再继续下面的流程，发HTTP/1.1的GET包。(也没有发其他数据包，除了探测网关欺骗的ARP包外)，一直等到几十秒或几分钟后，才开始继续接下来的流程。从上面问题终端的数据包截图可以看到，终端在19:26:03收到HTTP/1.0 200 ok包之后，一直等到19:26:47才发出HTTP/1.1 GET包，才开始继续后面的流程。在这持续的44s时间中，WiFi图标未点亮、portal页面也不会弹出，造成WiFi图标无法点亮、portal页面弹出缓慢，客户认为WiFi连接不上的情况。从目前部分区域问题终端抓包分析，均为终端自身不再继续发包（除了探测网关欺骗的ARP包外）导致无法重定向，和Wi-Fi设备无关。

------

在无线环境中经常碰到苹果终端连接无线后WiFi图标无法点亮导致终端无法上网、在起Portal的网络中认证页面无法自动弹出影响使用体验。

#### **一、WiFi图标无法点亮分三种情况：**

- 无线属内网环境，自动获取地址时没有下发DNS地址，导致wifi图标始终无法点亮。

- 无线即无Portal认证、又无加密，此时有些苹果终端每次连接wifi都会延迟10秒左右才能点亮图标。

- 无线环境起了Portal认证，wifi图标延迟10秒到45秒才能点亮。

  

#### 二、WiFi图标点亮以及portal不弹窗分析

1.  概率性不弹窗分析

   参考了苹果官方文档的状态机，发现非首次连接的ssid是有可能被cache的，如果cache，就会直接交给“热点助手（Hotspot helper）”，目前看，只要意图合理合法的APP都有可能注册为热点助手。热点助手对于频繁连接断开的ssid处理行为可能会有特殊处理，比如不再嗅探，直接点亮wifi标识。

2.  概率性弹窗慢分析

   苹果弹窗的嗅探报文并不总是立即发出，偶尔会出现不大于45s的延迟，因此产生了弹窗慢的问题。查看苹果的官方文档，发现苹果在分配到ip以后如果该网络没有cache过会进入网络评估状态，这个过程需要与注册为“热点助手（Hotspot helper）”的APP交互。系统先发一个评估（Evaluate）的命令给每一个热点助手，命令包含SSID和BSSID，每一个命令助手要在45s内做出对该网络的可信度（有none, low, high三档）评估。

        1. 第一个给出high的热点助手会被选为best_helper，后面的弹窗会与该助手交互。
        2. 如果所有的热点助手都没有给出high，选择给评估最高的助手为best_helper，后面的弹窗会与该助手交互。（比如有四个助手，三个给了none，一个给了low，那么这个给low的会被选中）
        3. 如果45s没有热点助手给出任何可信度的评估，苹果会迁移到认证结束（Authenticated）的状态，不会有弹窗。

   从抓包上看，再结合苹果对热点助手的规则，可以推断出：最坏的结果是，首次连接，也许所有的热点助手都没有关于当前网络的记录，所以评估过程不可能在45s内给出high，甚至最少有一个给了none其他的根本没有给出评估结果，超时后系统把这个给出none的选为best_helper，开始后面的弹窗交互，这种情况，一定会延迟45s。也有可能是热点助手收到评估命令后，有些立刻给出了评估，有些给的迟一些，但是还是在45s以内所有的助手都给出了评估，系统选出来了best_helper，开始弹窗流程，这个过程也许用了10s，也许用了20s，这可能就是弹窗延迟时间不定的原因。

以下是典型的热点助手的行为：![http://zhiliao.h3c.com/uploadfile/20170723/17364354390327641665.png]()

我们通过安装一些可能具有热点助手功能的APP进行抓包后发现有些热点助手具有特殊的嗅探报文，比如钱盾，会在以上的嗅探报文基础上额外发出若干如下报文：

GET /hotspot-detect.html HTTP/1.1

Host: captive.apple.com

Accept: */*

Accept-Language: zh-cn

Connection: keep-alive

Accept-Encoding: gzip, deflate

User-Agent: %E9%92%B1%E7%9B%BE/5.2.2 CFNetwork/811.5.4 Darwin/16.6.0

这个报文的User-Agent为“钱盾/5.2.2 CFNetwork/811.5.4 Darwin/16.6.0”。

具体有哪些APP注册了热点助手，这些APP如何评估当前网络，我们是不能完全知情的。也就是说，弹窗的快慢，是由苹果设备的所有热点助手的综合行为决定的。

​    关于PSK加密，这种情况会影响某个APP（比如钱盾）的评估，直接给出high，加快弹窗。

3、评估官网文档关于Captive网络的处理流程如下：

总结：

1.  关联网络建立ip连接评估阶段，弹窗是在评估阶段后面开始的，加密服务模板是在关联网络阶段，并没有证据证明服务模板加密是否可以影响到后面某一个Hotspot Helper对当前网络的评估。
2.  一旦有一个helper返回一个它可以以高可信度处理该网络的结果，其他helper的结果就可以被忽略了，每一个helper有最多45s的时间进行评估，如果45s内没有最优解，系统只能选择可信度最高的作为best_helper。（high confidence 如何理解，文档并没有精确说明）
3. 究竟有哪些APP注册过Hotspot Helper，我们还不能确定，用户手机安装了哪些APP，我们也是不可知的，所以在评估状态下究竟会返回什么结果，何时返回结果都是不确定的。
4.  偶尔不弹窗的原因也得到了解释：当前网络被Cache了，就不会走到评估阶段，也就不发嗅探报文了。

解决办法

1、如果是内网环境也一定要配置DNS地址，并且做PSK加密。

2、不起portal或起了portal无感知时尽量加PSK加密。

3、起了portal的无线环境中，常用的网络优化配置如下，可以提升体验程度（V7）

**针对上述的分析，结合对iPhone手机安卓手机和苹果笔记本的整体要求，整理优化的portal配置如下：**

```text
portal free-rule 1 destination ip any udp 53 
portal free-rule 2 destination ip any tcp 53 //放通DNS查询UDP or TCP 53端口 
portal free-rule 3 destination ip any tcp 5223 //iOS iPhone特殊情况查询DNS方式

portal safe-redirect enable
portal safe-redirect method get post
portal safe-redirect user-agent Android
portal safe-redirect user-agent CaptiveNetworkSupport
portal safe-redirect user-agent MicroMessenger
portal safe-redirect user-agent Mozilla
portal safe-redirect user-agent WeChat
portal safe-redirect user-agent iPhone
portal safe-redirect user-agent micromessenger

portal web-server h3ctest 
url http://A.B.C.D/portal
server-type cmcc 
captive-bypass ios optimize enable //无线对于iOS产品的优化功能，能够欺骗探测报文并且回应Success信息，让iOS Portal流程更加完美
```

```
if-match original-url http://captive.apple.com/hotspot-detect.html user-agent Mozilla temp-pass rediret-ulr http://xx/portal
if-match original-url http://www.apple.com user-agent Mozilla temp-pass rediret-ulr http://xx/portal
```

两条if-match命令是对macbook笔记本专用，主要特点为temp-pass功能，实现第三次探测http的时候进行放通，触发临时放行的地址，通过此配置将临时放行提前，避免出现弹出页面后wifi标识点亮又消失的问题（PS：部分安卓终端在Portal认证时，也会探测一些特殊地址，导致Portal弹窗延迟大，可以将探测的地址写到if-match里。）

最后服务模板增加关键配置temp-pass，开启Portal临时放行功能，并设置临时放行时间为20秒。

```text
 wlan service-template 1
 ssid ABCD 
 portal enable method direct 
 portal domain h3ctest 
 portal bas-ip 172.16.1.105 
 portal apply web-server h3ctest 
 portal temp-pass period 20 enable 
 service-template enable
```

> 1)无论是原理分析还是实际验证，等待45秒弹窗问题根本上是苹果终端自身控制的，需要苹果操作系统自身做出改变。
> 2)可以尝试PSK加密，加快弹窗过程。



### [**How to modernize your captive network**](https://developer.apple.com/cn/news/?id=q78sq5rv) 

### [**热点网络子系统**](https://developer.apple.com/library/archive/documentation/NetworkingInternet/Conceptual/Hotspot_Network_Subsystem_Guide/Contents/Introduction.html#//apple_ref/doc/uid/TP40016639-CH1-SW1) 

### [**新华三公众号无线优化**](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzA4NTQzNzMxNA==&action=getalbum&album_id=1615036232995045377&scene=173&from_msgid=2654034130&from_itemidx=1&count=3&nolastread=1#wechat_redirect) 

### [**苹果终端WiFi图标点亮慢和Portal弹窗机制分析以及处理办法和建议**](https://zhiliao.h3c.com/theme/details/19694)

###  [**WiFi连接问题**](http://helian.info/s/712.html)

