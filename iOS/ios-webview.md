# Webview

在 iOS 平台加载一个 H5 网页，需要经过哪些步骤：

**初始化 webview -> 请求页面 -> 下载数据 -> 解析HTML -> 请求 js/css 资源 -> dom 渲染 -> 解析 JS 执行 -> JS 请求数据 -> 解析渲染 -> 下载渲染图片**

由于在 dom 渲染前的用户看到的页面都是白屏，优化思路具体也是去分析在 dom 渲染前每个步骤的耗时，去优化性价比最高的部.

这其中可做的优化特别多，前后端能够做的是

> - 降低请求量：减少 HTTP 请求数, 合并资源，minify / gzip 压缩，webP，lazyLoad。
>
> - HTTP 协议缓存请求，离线缓存 manifest，离线数据缓存localStorage。
> - 加快请求速度：预解析DNS，减少域名数，并行加载，CDN 分发。
> - 渲染：JS/CSS优化，加载顺序，服务端渲染模板直出。



由于`H5`加载速度非常依赖网络环境，所以为了提高用户体验，针对`H5`加载速度的优化非常重要。`离线包`是最常用的优化技术，通过提前下载`H5`渲染需要的`HTML/JS/CSS`资源，加载时直接使用本地缓存资源避免额外的网络请求提高加载速度。

离线包技术发展到现在已经比较成熟。离线包技术主要是分为两部分，一部分是客户端离线包容器，另一部分是线上离线包平台。

#### 离线包容器

- `资源请求拦截` - 拦截`H5`资源请求，当存在本地缓存资源时直接返回使用
- `资源缓存` - 资源下载、资源缓存策略、增量更新策略

#### 离线包平台

- `资源管理` - 配置`H5`页面对应的离线资源、公共离线资源、`CDN`存放离线资源包
- `发布系统` - 实时发布、灰度能力、版本控制。

#### NSURLProtocol 方案

使用`NSURLProtocol`拦截所有`WebView`内发出的请求。

**方案存在的问题**

###### Body丢失

因为`WKWebView`本身是使用`多进程`模式，`WebView`资源网络请求并不在`APP`进程中。`iOS`系统目前的实现，当拦截HTTP网络请求时会丢失`Body`，所以需要处理`Body`丢失的问题。一种方式是`替换`掉`WebView`内部的网络 API，例如`Fetch`/`XMLHttpRequest`，但是并不能覆盖所有场景。另一种方式是网络请求走`原生API`桥接的方式，但是这需要`H5`进行适配有一定的`侵入性`。

###### 使用私有API

`WKWebView`本身并不支持网络请求拦截，当我们需要拦截网络请求时，需要使用系统`私有API`通过`ObjC Runtime`的方式动态调用。存在一定的审核风险，例如`Apple`审核时不允许使用被拒。另外因为并不是系统暴露出的 API，内部实现未来可能会改变。

#### WKURLSchemeHandler 方案

`WKURLSchemeHandler`是`iOS11`引入的新特性，可以通过此 API 来拦截`H5`的网络请求。

**方案存在的问题**

##### 不支持HTTP/HTTPS协议

- `不支持HTTP/HTTPS协议` - 因为`WKURLSchemeHandler`API 本身的设计，只能拦截自定义协议并不支持`HTTP/HTTPS`协议。一种方式是原生加载`H5`时使用自定义协议或`H5`内资源使用自定义协议。另一种方式是`hook`系统方法支持`HTTP/HTTPS`协议，但是这会带来一定的风险和不确定性。

##### Cookie 问题

`WKURLSchemeHandler`不会处理响应里的`Set-Cookie`，所以需要自行处理。

##### Body丢失问题

此方案同样存在`Body`丢失问题。

#### Local Server 方案

`Local Server`方式是通过在`APP`运行时启动一个`本地服务器`，请求`H5`时访问本地服务器，本地服务器检查是否可以使用本地离线资源。

**方案存在的问题**

##### 虚拟链接

- `虚拟链接` - 因为需要使用`虚拟链接`访问本地服务器，所以会带来`cookie同步`等问题需要解决

##### 资源消耗

- 本地服务器有额外的`内存`、`CPU`消耗

#### PWA 方案

`PWA`提供了一整套`Service Worker` API来实现离线`H5`能力，包括资源的下载、更新、缓存策略等。只不过`iOS`系统本身没有提供默认的实现，需要自实现一整套相关的 `Service Worker` API，复杂度和工作量比较高。



# prefetch方案介绍

实现思路是利用`H5`浏览器自带的`prefetch`能力。通过将离线包资源聚合到单个`HTML`中，`APP`启动后使用`WebView`提前加载`HTML`，`WebView`会下载资源到设备中。同时可以直接复用`WebView`自带的离线缓存能力和差异化资源更新能力。

通过运行自动化脚本的方式，基于`Puppeteer`和`Performance Timing`API，自动计算出需要下载的离线包资源及时更新。使用浏览器自带的`PerformanceTiming` API判定。`domInteractive`是浏览器完成对所有`HTML`的解析并且DOM构建完成的时间点。在`domInteractive`之前加载的资源既为`阻塞`首屏渲染的资源。同时需要过滤掉一些不需要缓存的资源，目前我们只收集`JS/CSS`会阻塞渲染的资源。

### iOS系统

#### 不支持prefetch

`iOS`系统web内核并不支持`prefetch`特性，所以针对iOS我们采用`preload`来代替。`Android`平台下发`link-prefetch`，`iOS`平台下发`link-preload`进行差异化处理。

> 提示：`prefetch`相比`preload`性能更好。`prefetch`下载的优先级没有`preload`高，避免影响其他网络请求速度。`preload`会将`JS`/`CSS`进行解析添加到内存，造成一定的额外消耗。

#### preload 不支持 HTML

`iOS`系统`preload`特性并不支持`HTML Document`的提前加载。不过这一点对于我们影响不大，因为目前我们业务`H5`的`HTML`通常会做一定的服务端渲染逻辑，并不支持缓存策略。(例如聚合一部分的公共 JS)





### NSURLProtocol

当我们需要拦截URL请求时，我们只要通过 - registerClass: 方法注册我们的NSURLProtocol类，然后去重写NSURLProtocol类中的方法，就能对我们发起的请求做处理。离线缓存主要是缓存所有的response，data数据。



自定义NSURLCache的子类CustomURLCache，重写对request的cache response相应方法，实现request对server请求之前进行拦截。然后把request作为key值在URLCache中查询是否缓存有对应的response，如果有，则直接返回不再需要进行请求;如果没有，那么返回空值让web进行请求，同时开启一个请求任务对request进行下载，下载成功后把数据和request作为value和key缓存到URLCache。



1.+ (BOOL)canInitWithRequest:(NSURLRequest *)request；

此处可以拦截需要处理的URL，已经处理过的请求，需要标记请求是否被处理过，防止死循环

2.+ (NSURLRequest *)canonicalRequestForRequest:(NSURLRequest *)request；

这个方法用来统一处理请求 request 对象的，可以修改头信息，或者重定向。没有特殊需要，则直接return request;

如果要在这里做重定向以及头信息的时候注意检查是否已经添加，因为这个方法可能被调用多次，也可以在后面的方法中做。

3.- (void)startLoading

把当前请求的request拦截下来以后，在这个方法里面对这个request做各种处理，比如添加请求头，重定向网络，使用自定义的缓存等。

重点：需要标记已经处理过的 request：

4.- (void)stopLoading

取消流程



# WKWebview

网络请求是在非主进程里发起，所以 NSURLProtocol 无法拦截到网络请求。除非使用私有API来实现。使用WKBrowsingContextController和registerSchemeForCustomProtocol。 通过反射的方式拿到了私有的 class/selector。通过把注册把 http 和 https 请求交给 NSURLProtocol 处理。

正常拦截处理，但是又发现拦截不了post请求(拦截到的post请求body体为空)，即使在canInitWithRequest:方法中设置对于POST请求的request不处理也不能解决问题

- 基于 LocalWebServer 实现 WKWebView 离线资源加载
- 使用WKURLSchemeHandler实现 WKWebView 离线资源加载



#### WKURLSchemeHandler注册

setURLSchemeHandler注册时机只能在WKWebView创建WKWebViewConfiguration时注册。

WKWebView 只允许开发者拦截自定义 Scheme 的请求，不允许拦截 “http”、“https”、“ftp”、“file” 等的请求，否则会crash。

【补充】WKWebView加载网页前，要在user-agent添加个标志，H5遇到这个标识就使用customScheme,否则就是用原来的http或https



#### 与 JavaScript 的交互

分别讲解 UIWebView 与 WKWebView 是如何与 JS 交互的。

##### UIWebView 与 JS 交互

- iOS 6 之前，`UIWebView` 是不支持共享对象的，Web 端需要通知 Native，需要通过修改 location.url，利用跳转询问协议来间接实现，通过定义 URL 元素组成来规范协议。
- iOS 7 之后，新增了 `JavaScriptCore` 库，内部有一个 `JSContext` 对象，可以用它来实现共享。

综上所述，UIWebView 与 JS 的交互，是通过 JavaScriptCore 库中的 `JSContext` 对象。

**JS 执行原生代码**

JS 是不能执行 OC 代码的，但是可以`间接的执行`，即 JS 将需要执行的操作封装到网络请求中，然后原生代码中拦截这个请求，获取 url 中的字符串解析即可，这里需要用到 `WebView(_:shouldStartLoadWith:navigationType:)` 这个代理方法。

**原生调用 JS 代码**

原生条用 JS 代码需要用到 UIWebView 的一个方法，即 `stringByEvaluatingJavaScript(from:)` 函数。



##### WKWebView 与 JS 交互

- 在 WKWebView 上，Web 的 window 对象提供 `WebKit` 对象实现共享；
- 而 WKWebView 绑定共享对象，是通过特定的构造方法实现，即通过指定 `WKUserContentController` 对象的 `ScriptMessageHandler` 经过 `Configuration` 参数构造时传入；
- 而 Handler 对象需要实现指定协议，实现指定的协议方法，当 JS 端通过 `window.Webkit.messageHandlers` 发送 Native 消息时，handler 对象的协议方法被调用，然后通过协议方法的相关参数传值。

综上所述，WKWebView 与 JS 的交互，是通过 `WKScriptMessageHandler` 协议的代理方法进行的。

**JS 执行原生代码**

需要通过 `WKScriptMessageHandler` 代理中的函数实现，即需要使用 `userContentController:didReceiveScriptMessage` 代理方法。

**原生调用 JS 代码**

通过 WKWebView 中的一个方法实现，即 `evaluateJavaScript(_:completionHandler:)` 函数。



[如何接管WKWebView的网络请求？](https://juejin.cn/post/7109771712526286862)

[探索 WKWebView 新增功能](https://juejin.cn/post/7101545525576466439)

[基于 prefetch 的 H5 离线包方案](https://juejin.cn/post/7011837444865654797)

[WKWebview秒开的实践及踩坑之路](https://juejin.cn/post/6861778055178747911)

[iOS app秒开H5优化探索](https://juejin.cn/post/6844903809521549320)

[WKWebView离线化方案——实现Service Worker API](https://zhuanlan.zhihu.com/p/148931732)

[解析 WKWebView 的 Cookie 封锁](https://juejin.cn/post/7008547869129408548)

[从零收拾一个hybrid框架](http://awhisper.github.io/2018/01/02/hybrid-jscomunication/)

[HTML5 容器入门解析：支付宝 Hybrid 方案原理与实战](https://gitchat.csdn.net/activity/5d552a71e5a42b77df3537d4?utm_source=so)

[iOS H5启动加载优化方案](https://juejin.cn/post/7087095891152076807)

[iOS 端 h5 页面秒开优化实践](https://juejin.cn/post/6844903954212454413#heading-4)
