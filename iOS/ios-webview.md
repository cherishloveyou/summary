# Webview

在 iOS 平台加载一个 H5 网页，需要经过哪些步骤：

初始化 webview -> 请求页面 -> 下载数据 -> 解析HTML -> 请求 js/css 资源 -> dom 渲染 -> 解析 JS 执行 -> JS 请求数据 -> 解析渲染 -> 下载渲染图片



这其中可做的优化特别多，前后端能够做的是

> - 降低请求量：减少 HTTP 请求数, 合并资源，minify / gzip 压缩，webP，lazyLoad。
>
> - HTTP 协议缓存请求，离线缓存 manifest，离线数据缓存localStorage。
> - 加快请求速度：预解析DNS，减少域名数，并行加载，CDN 分发。
> - 渲染：JS/CSS优化，加载顺序，服务端渲染模板直出。



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





[基于 prefetch 的 H5 离线包方案](https://juejin.cn/post/7011837444865654797)

[WKWebview秒开的实践及踩坑之路](https://juejin.cn/post/6861778055178747911)

[iOS app秒开H5优化探索](https://juejin.cn/post/6844903809521549320)

[WKWebView离线化方案——实现Service Worker API](https://zhuanlan.zhihu.com/p/148931732)

[解析 WKWebView 的 Cookie 封锁](https://juejin.cn/post/7008547869129408548)

[从零收拾一个hybrid框架](http://awhisper.github.io/2018/01/02/hybrid-jscomunication/)

[HTML5 容器入门解析：支付宝 Hybrid 方案原理与实战](https://gitchat.csdn.net/activity/5d552a71e5a42b77df3537d4?utm_source=so)

[iOS H5启动加载优化方案](https://juejin.cn/post/7087095891152076807)

[iOS 端 h5 页面秒开优化实践](https://juejin.cn/post/6844903954212454413#heading-4)
