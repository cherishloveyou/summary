# 网络优化



- **[HTTPS带来的负担](https://blog.csdn.net/zhuyiquan/article/details/71430020?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-9&spm=1001.2101.3001.4242#1-https带来的负担)**

1. **增加的传输延时**

  HTTP首次请求时只要和服务端TCP三次握手建立连接，便可以开始应用数据传输了,使用HTTPS传输增加的开销不仅仅是两次TLS握手的过程。

   1. 用户习惯于使用HTTP请求你的网站。要保护用户的安全，首先要让用户强制302/301到HTTPS。这次跳转至少增加1个RTT的延时；
   2. 302跳转后要再次TCP建连，增加1个RTT的延时；
   3. 开始两阶段TLS握手，细节如下图所示，增加至少两个RTT的延时
   4. 另外客户端如果第一次获取服务端的证书链信息，还需要通过Oscp来验证证书的吊销状态，又需要至少1个RTT延时。
   5. 最终，开始应用层数据的传输。

  RTT(Round-Trip Time): 往返时延。在计算机网络中它是一个重要的性能指标，表示从发送端发送数据开始，到发送端收到来自接收端的确认（接收端收到数据后便立即发送确认），总共经历的时延。

2. **服务端额外开销**

     TLS握手过程中密钥交换和加密对CPU都会产生额外的计算开销。选择不同的算法（身份验证算法、密钥交换算法、加密算法）开销不同。比如，2048位RSA作为密钥交换算法对CPU压力就会很大，而ECDHE_RSA（椭圆曲线密钥交换）开销就小的多，RSA可以仍保留用于身份验证。 

- **服务端性能优化**

    2.1 HSTS的合理使用

    ​    HSTS（HTTP Strict Transport Security, HTTP严格传输安全协议）表明网站已经实现了TLS，要求浏览器对用户明文访问的Url重写成HTTPS，避免了始终强制302重定向的延时开销。

    2.2 会话恢复的合理使用

    ​    用会话恢复机制是指在一次完整协商的连接断开时，客户端和服务端会将会话的安全参数保存一段时间。后续的连接，双方使用简单握手恢复之前协商的会话。大大减少了TLS握手的开销。

    2.3 Ocsp Stapling的合理使用

    ​    OCSP 是**server 把自己的站点证书和中间证书以及根证书打包一起下发到客户端**，省去客户端查询的过程。
    ​    OCSP实时查询会增加客户端的性能开销。因此，可以考虑通过OCSP stapling的方案来解决：OCSP stapling是一种允许在TLS握手中包含吊销信息的协议功能，启用OCSP stapling后，服务端可以代替客户端完成证书吊销状态的检测，并将全部信息在握手过程中返回给客户端。增加的握手信息大小在1KB以内，但省去了用户代理独立验证吊销状态的时间。

    2.4 TLS协议的合理配置

    2.5 False Start的合理使用

    2.6 SNI功能的合理使用

    2.7 HTTP 2.0的合理使用

       HTTP2和HTTP1.1 比起来有什么优势

        1. HTTP/2采用二进制格式而非文本格式
        2. HTTP/2是完全多路复用的，而非有序并阻塞的——只需一个连接即可实现并行
        3. 使用报头压缩，HTTP/2降低了开销
        4. HTTP/2让服务器可以将响应主动“推送”到客户端缓存中

    多路复用

    在 HTTP 1.1 中，发起一个请求是这样的：浏览器请求 url -> 解析域名 -> 建立 HTTP 连接 -> 服务器处理文件 -> 返回数据 -> 浏览器解析、渲染文件。
    这个流程最大的问题是，每次请求都需要建立一次HTTP连接，也就是我们常说的3次握手4次挥手，这个过程在一次请求过程中占用了相当长的时间，而且逻辑上是非必需的，因为不间断的请求数据，第一次建立连接是正常的，以后就占用这个通道，下载其他文件，这样效率多高啊！为了解决这个问题， HTTP 1.1 中提供了 Keep-Alive，允许我们建立一次 HTTP 连接，来返回多次请求数据。
        但是这里有两个问题：
        HTTP 1.1 基于串行文件传输数据，因此这些请求必须是有序的，所以实际上我们只是节省了建立连接的时间，而获取数据的时间并没有减少最大并发数问题，假设我们在 Apache 中设置了最大并发数 300，而因为浏览器本身的限制，最大请求数为 6，那么服务器能承载的最高并发数是 50而 HTTP/2 引入二进制数据帧和流的概念，其中帧对数据进行顺序标识，这样浏览器收到数据之后，就可以按照序列对数据进行合并，而不会出现合并后数据错乱的情况。同样是因为有了序列，服务器就可以并行的传输数据。HTTP/2 对同一域名下所有请求都是基于流，也就是说同一域名不管访问多少文件，也只建立一路连接。同样Apache的最大连接数为300，因为有了这个新特性，最大的并发就可以提升到300，比原来提升了6倍。

    2.8 SSL硬件加速卡合理使用

    

- **客户端性能优化**

  3.1 移动端HTTP2加速代理

  3.2 HttpsDns解决DNS攻击劫持
  
    针对网络连接和读写操作的超时时间，我们提出了网络质量检测机制。目前做到的是根据用户是在 2G/3G/4G/Wi-Fi 的网络环境来设置不同的超时参数，以及网络服务的并发数量。2G/3G/4G 网络环境对并发 TCP 连接的数量是有限制的（2G 网络下运营商经常只能允许单个 Host 一个 TCP 连接），因此网络服务重要参数能够根据网络质量状况来动态设定对性能和体验都非常重要。
  
  由于我们的 App 网络服务主要基于 TCP 连接，为了将 DNS 时间降至最低，我们内置了 Server IP 列表，该列表可以在 App 启动服务中下发更新。App 启动后的首次网络服务会从 Server IP 列表中取一个 IP 地址进行 TCP 连接，同时 DNS 解析会并行进行，DNS 成功后，会返回最适合用户网络的 Server IP，那么这个 Server IP 会被加入到 Server IP 列表中被优先使用。
  
  
------

#### 提供网络服务重发机制

​    移动网络不稳定，如果一次网络服务失败，就立刻反馈给用户你失败了，体验并不友好。我们提供了网络服务重发机制，即当网络服务在连接失败、写 Request 失败、读 Response 失败时自动重发服务；长连接失败时就用短连接来做重发补偿，短连接服务失败时当然还是用短连接来补偿。这种机制增加了用户体验到的服务成功概率。当然不是所有网络服务都可以重发，例如当下订单服务在读取 Response 失败时，就不能重发，因为下单请求可能已经到达服务器，此时重发服务可能会造成重复订单，所以我们添加了重发服务开关，业务段可以自行控制是否需要。

#### 减少数据传输量

​    我们优化了 TCP 服务 Payload 数据的格式和序列化 / 反序列化算法，从自定义格式转换到了 Protocol Buffer 数据格式，效果非常明显。序列化 / 反序列算法也做了调整，如果大家使用 JSON 数据格式，选用一个高效的反序列化算法，针对真实业务数据进行测试，收益明显。图片格式优化在业界已有成熟的方案，例如 Facebook 使用的 WebP 图片格式，已经被国内众多 App 使用。

#### iOS14本地网络检测

​    复用开网接口-1009，同时启动DNS查询，某些情况下*DHCP*未分配地址也会导致请求不通，通过实验发现未分配地址的时候的IP段和正常假如的IP不一样(非正常169.254.x.x，正常的局域网IP是192.168.x.x)。需要本地网络功能，主要因为是高铁WiF搭建的局域网的网络拓扑，中间多一层隔离，开网请求需要访问局域网中一个地址进行开网请求。

#### DNS优化

​    1、我们知道运营商本来是根据域名来确定一个URL的，我们将域名改为IP之后虽然不用运营商帮我们解析了，但是运营商在收到一串数字的时候也是懵逼状态，我们还是需要将域名传给他们，但是不能用正常的方式传，我们需要把原来的域名加到http请求的Header中的host字段下，根据Http协议的规定，如果在URL中无法找到域名的话就会去Header中找，这样一来我们既把域名告诉了运营商同时也直接制定了IP地址，这个是必须配置的，不然的话是请求不成功的。

```objective-c
[mutableRequest setValue:self.request.URL.host forHTTPHeaderField:@"HOST"];
```

加上Header再去请求就没问题了，不过有些特殊的情况下会需要带上cookie，同样也是加到Header中

```objective-c
[mutableRequest setValue:YOUR Cookie forHTTPHeaderField:@"Cookie"];
```

   2、关于AFNetworking的问题，现在大部分网络请求是基于AFNetworking的，这里有一个坑，我们知道我们注册CustomProtocol的时候是这样

```objective-c
//注册自定义protocol
[NSURLProtocol registerClass:[CustomURLProtocol class]];
NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
configuration.protocolClasses = @[[CustomURLProtocol class]];
```

在系统的configuration加入我们的CustomProtocol，protocolClasses是一个数组里面可以放很多各种不同的CustomProtocol，我们看一下AFNetworking的初始化方法。

```objective-c
AFHTTPSessionManager *sessionManager = [AFHTTPSessionManager manager];
```

我相信大家通常都会这么来创建，但是这里我要说下manager并不是一个单列，最后都会调到一个方法

```objective-c
- (instancetype)initWithSessionConfiguration:(NSURLSessionConfiguration *)configuration {
    self = [super init];
    if (!self) {
        return nil;
    }
    if (!configuration) {
        configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    }
    self.sessionConfiguration = configuration;
    self.operationQueue = [[NSOperationQueue alloc] init];
    self.operationQueue.maxConcurrentOperationCount = 1;
    self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];
}
```

大家注意第二个判断，如果没有传入configuration的话他会创建一个默认的，这样以至于我们之前在configuration的protocolClasses中注册类全部被这个新的configuration替换掉了，所以无法解析。这里我采取的办法就是runtime hook，因为hook第三方的代码并不是一个很好的办法，所以我直接hook NSURLSession的sessionWithConfiguration方法，因为通过观察AFNetworking的源码最终都是走到这里的。Hook之后把自己的configuration换进去，像这样

```objective-c
+ (NSURLSession *)swizzle_sessionWithConfiguration:(NSURLSessionConfiguration *)configuration {

    NSURLSessionConfiguration *newConfiguration = configuration;
    // 在现有的Configuration中插入我们自定义的protocol
    if (configuration) {
        NSMutableArray *protocolArray = [NSMutableArray arrayWithArray:configuration.protocolClasses];
        [protocolArray insertObject:[CustomProtocol class] atIndex:0];
        newConfiguration.protocolClasses = protocolArray;
    } else {
        newConfiguration = [NSURLSessionConfiguration defaultSessionConfiguration];
        NSMutableArray *protocolArray = [NSMutableArray arrayWithArray:configuration.protocolClasses];
        [protocolArray insertObject:[CustomProtocol class] atIndex:0];
        newConfiguration.protocolClasses = protocolArray;
    }
    return [self swizzle_sessionWithConfiguration:newConfiguration];
}
```
然后就完美解决了。不过要注意下系统的是有两个方法的

```
/*
 * Customization of NSURLSession occurs during creation of a new session.
 * If you only need to use the convenience routines with custom
 * configuration options it is not necessary to specify a delegate.
 * If you do specify a delegate, the delegate will be retained until after
 * the delegate has been sent the URLSession:didBecomeInvalidWithError: message.
 */
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration;
+ (NSURLSession *)sessionWithConfiguration:(NSURLSessionConfiguration *)configuration delegate:(nullable id <NSURLSessionDelegate>)delegate delegateQueue:(nullable NSOperationQueue *)queue;

```
这两个方法不能确定最终会走那个，所以为了保险起见都hook下，hook的方式是一样的

   3、AVPlayer请求，AVPlayer是我们iOS系统中系统自带的播放视频的框架，用到地方也很多，但是这个是比较坑的，因为AVPlayer虽然也有http/https/file……请求这个概念，但是AVPlayer所有的请求都不会走URL Loading System，也就是说所有由AVPlayer发出的请求都不能被我们的CustomProtocol拦截，这时候大家也许会问，不对呀，我们正常调试的时候可以被拦截到的啊。其实苹果官方上是说AVPlayer在真机调试和模拟器调试时候走的完全不是一套策略，也就是说在模拟器运行时候是完全正常的，可以被拦截到也可以被解析，但是在真机上面就恰恰相反了，因为我们最后还是以真机为准，所以我们采取的办法还是hook，因为我们需要在媒体URL传给AVPlayer前就要将相关东西配置好，域名替换啊，加host啊之类的，所以我们要找AVPlayer的入口，先看初始化方法，我发现项目中使用一个AVURLAsset来初始化AVPlayer，那么AVURLAsset又是什么呢？继续查到AVURLAsset的初始化方法，可以发现这个方法：

```objective-c
/*!
  @method	initWithURL:options:
  @abstract	Initializes an instance of AVURLAsset for inspection of a media resource.
  @param URL
			An instance of NSURL that references a media resource.
  @param	options
			An instance of NSDictionary that contains keys for specifying options for the initialization of the AVURLAsset. See AVURLAssetPreferPreciseDurationAndTimingKey and AVURLAssetReferenceRestrictionsKey above.
  @result	An instance of AVURLAsset.
*/
- (instancetype)initWithURL:(NSURL *)URL options:(nullable NSDictionary<NSString *, id> *)options NS_DESIGNATED_INITIALIZER;
```

其中URL就是我们传给AVPlayer播放的URL，找到目标就Hook下就可以了，具体过程就不多说了还是字符串替换，但是有一点需要注意的是，我之前上文说过做完IP对域名的替换之后还需要设置下request的Host，但是这个地方只有一个URL并没有Request该如何处理呢？其实这个方法里面的opinion参数就是处理这个的，可以添加cookie之类的类似与httpheader的东西，可以添加这几个Key

```objective-c
AVF_EXPORT NSString *const AVURLAssetPreferPreciseDurationAndTimingKey NS_AVAILABLE(10_7, 4_0);
AVF_EXPORT NSString *const AVURLAssetReferenceRestrictionsKey NS_AVAILABLE(10_7, 5_0);
AVF_EXPORT NSString *const AVURLAssetHTTPCookiesKey NS_AVAILABLE_IOS(8_0);
AVF_EXPORT NSString *const AVURLAssetAllowsCellularAccessKey NS_AVAILABLE_IOS(10_0);
```

但是并没有发现和Host相关的Key，其实这个key是有的就是AVURLAssetHTTPHeaderFieldsKey只是因为这个Key没暴露出来。这个地方不太确定是不是苹果的私有API，网上查了大量的资料也没有个说法，甚至我亲自去苹果开发者去问，苹果也没有给任何答复，各种说法都有，具体使用的话就是

```
[self swizzle_initWithURL:videoURL options:@{AVURLAssetHTTPHeaderFieldsKey : @{@"Host":host}}]
```

这样使用是没有任何问题的，但是毕竟是没有暴露出来的方法，我们不能这样明目张胆的使用，其实对于字符串来说还是比较好规避的，只要不要明文出现这个KEY就可以，我在这里使用了一个加密，吧key变成密文然后这个地方通过解密获取，就像这样：

```
//加密后的KEY
const NSString * headerKey = @"35905FF45AFA4C579B7DE2403C7CA0CCB59AA83D660E60C9D444AFE13323618F";
.
//getRequestHeaderKey方法为解密方法
return [self swizzle_initWithURL:videoURL options:@{[self getRequestHeaderKey] : @{@"Host":host}}];
```
这样之后就大功告成了，AVPlayer可以在DNS被劫持的情况下播放了，

   4、POST请求这块也算是一个大坑，我们知道http的post请求会包含一个body体，里面包含我们需要上传的参数等一些资料，对于POST请求我们的NSURLProtocol是可以正常拦截的，但是我们拦截之后发现无论怎么样我们获得的body体都为nil！后来查了一些资料发下又是苹果爸爸在做手脚。NSURLProtocol在拦截NSURLSession的POST请求时不能获取到Request中的HTTPBody，这个貌似早就国外的论坛上传开了，但国内好像还鲜有人知，据苹果官方的解释是Body是NSData类型，即可能为二进制内容，而且还没有大小限制，所以可能会很大，为了性能考虑，索性就拦截时就不拷贝了。为了解决这个问题，我们可以通过把Body数据放到Header中，不过Header的大小好像是有限制的，我试过2M是没有问题，不过超过10M就直接Request timeout了。。。而且当Body数据为二进制数据时这招也没辙了，因为Header里都是文本数据，另一种方案就是用一个NSDictionary或NSCache保存没有请求的Body数据，用URL为key，最后方法就是别用NSURLSession，老老实实用古老的NSURLConnection算了。。。你以为这么就结束了吗？并没有，后来查了大量的资料发现，既然post请求的httpbody没有苹果复制下来，那我们就不用httpbody，我们再往底层去看就会发现HTTPBodyStream这个东西我们可以通过他来获取请求的body体具体代吗如下

```
#pragma mark - 处理POST请求相关POST  用HTTPBodyStream来处理BODY体
- (NSMutableURLRequest *)handlePostRequestBodyWithRequest:(NSMutableURLRequest *)request {
    NSMutableURLRequest * req = [request mutableCopy];
    if ([request.HTTPMethod isEqualToString:@"POST"]) {
        if (!request.HTTPBody) {
            uint8_t d[1024] = {0};
            NSInputStream *stream = request.HTTPBodyStream;
            NSMutableData *data = [[NSMutableData alloc] init];
            [stream open];
            while ([stream hasBytesAvailable]) {
                NSInteger len = [stream read:d maxLength:1024];
                if (len > 0 && stream.streamError == nil) {
                    [data appendBytes:(void *)d length:len];
                }
            }
            req.HTTPBody = [data copy];
            [stream close];
        }
    }
    return req;
}
```

 这样之后的req就是携带了body体的request啦，可以愉快地做post请求啦。

   5、WKWebview是新出的浏览器控件，这里就不多说了，WKWebview不走URL Loading System，所以也不会被拦截，不过也是有办法的，但是因为这次项目中没有用到，所以没有过多的去研究，后续我会写一篇关于这个博客，不是很难，依旧是runtime大法。

   6、SNI环境，这个可是坑了我好久好久的东西，所以我会放在最后去说，SNI环境因为涉及到证书验证所以是在https的基础上来说的，SNI（Server Name Indication）是为了解决一个服务器使用多个域名和证书的扩展。一句话简述它的工作原理就是，在连接到服务器建立SSL链接之前先发送要访问站点的域名（Hostname），这样服务器根据这个域名返回一个合适的证书。其实关于SNI环境在这里就不过多解释，**[阿里云文档](https://help.aliyun.com/document_detail/30143.html)**有很明白的解释，同时他也有安卓和iOS在SNI环境下的处理文档，我们发现安卓部分写的很详细，可是已到了iOS这边就这样了：

![阿里云文档截图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/707ed312457c44808beaa2e1fa190ace~tplv-k3u1fbpfcp-zoom-1.image)

三行文字加三个链接就完事了。其实在遇到这个坑的时候我也查过很多相关资料，无非就是这三行话加这三个链接复制来复制去，没有实质性的进展，大部分公司或者是项目没有这么重的Httpdns需求，所以也就不会有这个环境，即使遇到了也就直接关闭httpdns了，后来只能自己去用CFNetwork一点点实现。具体代码就不跟大家粘贴了因为涉及到一些公司内部的代码，不过我会把我**[主要的参考资料](https://github.com/Dave1991/alicloud-ios-demo/blob/master/httpdns_ios_demo/httpdns_ios_demo/CFHttpMessageURLProtocol.m)**发给大家。这里有个小技巧，因为都在说CFNetwork是比较底层的网络实现，好多东西需要开发者自行处理比如一些变量的释放之类的，所以我们能少用尽量少用，因为CFNetwork是为SNI(https)环境服务,所以我们在拦截判断的时候可以区分是用上层的网络请求转发还是用底层的CFNetwork来转发，

```objective-c
 if ([self.request.URL.scheme isEqualToString:@"https"]) {
    //使用CFnetwork
    curRequest = req;
    self.task = [[CustomCFNetworkRequestTask alloc] initWithURLRequest:originalRequest swizzleRequest:curRequest delegate:self];
    if (self.task) {
        [self.task startLoading];
    }
 } else {
    //使用普通网络请求
    NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration defaultSessionConfiguration];
    self.session = [NSURLSession sessionWithConfiguration:configuration delegate:self delegateQueue:[NSOperationQueue mainQueue]];
    NSURLSessionTask *task = [self.session dataTaskWithRequest:req];
    [task resume];
 }
```

   7、在NSURLProtocol中的那几个类方法中是可以发送同步请求的，但是在实例方法发送同步请求就会卡死，所以实例方法中不能有任何的阻塞，进行同步操作。不然就卡死。



-------------------------------

### 一、网络检测

 弱网优化的目的，是调高用户的使用体验。
 所以很重要的一点就是：不同网络加载状态的过程反馈，或"积极"反馈给用户（下文“用户体验优化”里，会提到加载进度从50%起的"积极"反馈做法）。

##### 1、无网络提示。

可为 Controller 监听网络状态的改变：
 例如用的 AFNetworking 的 `AFNetworkReachabilityStatus。`
 为无网络状态等进行告知用户的处理。

##### **2、加载前需要添加“正在加载中的动画/视图”**

例如 MBProgressHUD 的使用。

##### 3、加载后需要先移除网络状态视图，并增加判定空数据处理。[如有可能，判定是网络原因，还是无数据的原因。]

##### 4、善用状态切换的通知，为界面做出不同的变化。

在 2G、3G、4G、wifi，(现在还有 5g) 等不同的网络做出不同的状态图切换，或者交互切换。



### 二、网络请求优化

##### 1、制定最合适的超时时间：

对总读写超时(从请求到响应的超时)、首包超时、包包超时(两个数据段之间的超时)时间制定不同的计算方案，加快对超时的判断，减少等待时间，尽早重试。这里的超时时间还可以根据网络状态动态设定，例如在网络状态为 2G、3G、4G、wifi 下设置不同的超时时间。

##### 2、多子模块请求的“延迟性”

以用户等待容忍度不超过 2s 为原则，像首页这种多个业务模块一起呈现的页面，如果一次性请求完所有的接口数据，估计用户已经 Home 回到主页了，所以可以对多子模块，进行分段的“延迟”请求。

- 优先模块：请求数据量少、业务上需要优先显示的。
- 延后模块：数据量大、类似列表的多条数据，适合放置加载动画，时长上用户可接受性强，所以除了放在后面外，可做分页处理、滑动后的延迟加载处理。

##### 3、固定模块加入缓存机制、或增量更新机制

对首页及特定一级页面进行数据缓存，在一定的有效时间内再次请求可以直接从缓存读取数据，也可避免空白页出现影响体验。
 或者进行判断数据是否有增量变化，有点的话在插入动画的前提下，进行数据的更新。

##### 4、多模块的重新加载操作

像一些多模块，模块之间相关联的复杂页面，多个模块会有多个请求，当某个请求失败需要添加“重新加载”的按钮时，建议所有请求重新请求一遍，防止模块之间关联的数据出现偏差，或者 UI 布局错乱。
 例如，链家的 APP(仅供讨论，无其他意思哈，链家请见谅😬 )，弱网环境下，中间某模块 UI 产生混乱，如果在用户下拉重新加载请求的情况下，如果不是首页这整个模块请求后完，进行布局重新计算，那么后面的 UI 就一直是这个状态。
 所以，如果有做**网络请求失败后，重新加载的按钮/下拉操作**，建议是：

- **多模块再各自请求一遍。**
- **复杂 UI 重新计算一下。**

原因是：弱网环境，本身请求到的数据可能也不齐全，多个请求或许只能拿到部分数据，而大部分情况是，各模块是相辅相成的。

##### 5、预加载设置“临界值”

使用 Threshold 进行预加载是一种最为常见的预加载方式，知乎客户端就使用了这种方式预加载条目，而其原理也非常简单，根据当前 UITableView 的所在位置，除以目前整个 UITableView.contentView 的高度，来判断当前是否需要发起网络请求：在当前页面已经划过了 70% 的时候，就请求新的资源，加载数据；

但是，仅仅使用这种方法会有另一个问题，尤其是当列表变得很长时，十分明显，会导致新加载的页面都没有看，应用就会发出另一次请求，获取新的资源。

##### 6、从请求这个动作下手

**优化DNS查询：**应尽量减少DNS查询，做DNS缓存，避免域名劫持、DNS污染，同时把用户调度到“最优接入点”。
 **减小数据包大小和优化包量：**通过压缩、精简包头、消息合并等方式，来减小数据包大小和包量。
 **优化ACK包：**平衡冗余包和ACK包个数，达到降低延时，提高吞吐量的目的。

这几个，都是在优化请求的数据处理和查询上做优化的。感兴趣的话，可以翻到最下面的资料网翻阅，或通过查询 HTTP 协议，TCP/UDP 协议这部分网络优化内容，分析总结并对网络请求这块进行细致优化。

##### 7、断线重连

在无线网络中有太多的原因导致数据连接中断了。这里可以使用CDN。
（CDN 是构建在数据网络上的一种分布式的内容分发网。 CDN 的作用是采用流媒体服务器集群技术，克服单机系统输出带宽及并发能力不足的缺点，可极大提升系统支持的并发流数目，减少或避免单点失效带来的不良影响。）

##### 8、减少数据连接的创建次数

由于创建连接是一个非常昂贵的操作，所以应尽量**减少数据连接的创建次数**，且在一次请求中应尽量以批量的方式执行任务。如果多次发送小数据包，应该尽量保证在2秒以内发送出去。在短时间内访问不同服务器时，尽可能地复用无线连接。


### 三、用户体验优化

##### 1、内容分先后显示

例如，一个业务模块文字图片都有的情况。
 一个完整的请求和加载，可能一直卡在50%-90%的时候，那么先加载文字，再加载图片。

##### 2、进度的驱使

不管网络条件如何，加载进度始终是从50%起，并且停留在大约98%进度左右的地方。
 (当然， 不建议在数据量比较大的请求中做这样的显示机制，因为会让用户产生网络波动较大的体验。)

##### 3、固定的 UI 显示布局，加载时可预加载虚拟布局视图

类似知乎。在加载时，“正在加载中的动画/视图”，改为主页面显示预加载的占位图。

![img](https:////upload-images.jianshu.io/upload_images/1707017-d103db496fc1b795.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/300/format/webp)

交互上，数据的加载会让用户对 APP 的 UI 设计有熟悉感。

##### 4、弱网加载失败/空数据，可添加“重新加载”的按钮，或可增加下拉刷新操作。

例如：请求无数据/网络失败，添加‘重新加载“按钮，让用户意识到处于“可控”状态，降低用户焦躁情绪。

## **四、图片加载优化**

##### 1、使用更快的图片格式

严格说也不算弱网下的优化，但一个更快的图片格式真的很重要！这里建议采用 WebP 格式。（WebP 格式，谷歌（google）开发的一种旨在加快图片加载速度的图片格式。图片压缩体积大约只有 JPEG 的2/3，并能节省大量的服务器带宽资源和数据空间。但 WebP 是一种有损压缩。相较编码 JPEG 文件，编码同样质量的 WebP 文件需要占用更多的计算资源。）

##### 2、根据网络状态呈现不同精度的图

如（对于原图是 600X480 的图片）：

- 2/3G 使用低清晰度图片：下发 300X240，精度为 80 的图片；
- 4G 普通清晰度图片下发 600X480，精度为 80 的图片；
- WiFi 高清晰度图片（最好根据网速来判断，WiFi 也有慢的）：下发 600X480，精度为 100 的图片。

##### 4、SDWebImage 参数选项

根据使用场景，参照 [SDWebImageOptions常量说明](https://www.jianshu.com/p/d639aadad1b0)，对图片的加载进行。

##### 5、干脆不加载图片...

这也是一个方法，弱网情况下，在一些不影响操作，并能通过简单文字的描述告知用户该区域的内容，可以不加载图片，待到网络流畅状态再进行图片的加载。

当然这种方法要视情况而定，或者一般都在 APP 的设置选项，增加一个“弱网状态不显示图片”的按钮。



[**HttpDns集成方案**](https://juejin.cn/post/6959014022263865357) 

[**iOS网络深度优化总结**](https://www.jianshu.com/p/a470ab485e39 ) 

[**iOS 开发关于弱网优化**](https://www.jianshu.com/p/bd26837f2bd3/ ) 

[**HTTPS 的TLS过程**](https://juejin.cn/post/7006975134380589086)



 

 