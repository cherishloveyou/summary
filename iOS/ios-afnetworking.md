# AFNetworking

AFNetworking是封装的NSURLSession的网络请求。
AFNetworking由五个模块组成：分别由**NSURLSession,Security,Reachability,Serialization,UIKit**五部分组成

> NSURLSession：网络通信模块(核心模块) 对应 AFNetworking中的 AFURLSessionManager和对HTTP协议进行特化处理的AFHTTPSessionManager,AFHTTPSessionManager是继承于AFURLSessionmanager的
> Security：网络通讯安全策略模块  对应 AFSecurityPolicy
> Reachability：网络状态监听模块 对应AFNetworkReachabilityManager
> Seriaalization：网络通信信息序列化、反序列化模块 对应 AFURLResponseSerialization
> UIKit：对于IOSUIKit的扩展库

调用流程分析：
AFHTTPSessionManager: 发起网络请求(例如GET);
AFHTTPSessionManager内部调用dataTaskWithHTTPMethod:方法(内部处理requestSerializer);
dataTaskWithHTTPMethod内部调用父类AFURLSessionManager的dataTaskWithRequest: uploadProgress: downloadProgress: completionHandler方法;
AFURLSessionManager中的dataTaskWithRequest方法内部设置全局session和创建task;
AFURLSessionManager中的dataTaskWithRequest方法内部给task设置delegate(AFURLSessionManagerTaskDelegate);
taskDelegate代理的初始化: 绑定task / 存储task下载的数据 / 下载或上传进度 / 进度与task同步(KVO)
task对应的AFURLSessionManagerTaskDelegate实现对进度处理、Block调用、Task完成返回数据的拼装的功能等;
setDelegate: forTask: 加锁设置通过一个字典处理Task与之代理方法关联; 添加对Task开始、重启、挂起状态的通知的接收.
[downloadTask resume]后执行开始, 走代理回调方法(内部其实是NSURLSession的各种代理的实现);
task完成后走URLSession: task: didCompleteWithError: 回调对返回的数据进行封装;
同时移除对应的task; removeDelegateForTask: 加锁移除8中的字典和通知;

线程一般一次只执行一个任务，执行完毕就会退出；但是如果在子线程执行异步操作（如网络请求）后马上退出，就会导致返回的数据无法正常接收和处理，因此我们需要线程保活，确保异步的操作能够顺利完成。要执行任务后不退出，即常驻线程，可以使用RunLoop实现。

首先，我们需要先搞清楚线程与RunLoop的关系：
- RunLoop和线程是一一对应的；
- 子线程创建后，对应的RunLoop并没有创建和运行，而需要主动去获取才会创建，调用run方法才会开始运行；
- 线程添加了RunLoop并运行，实际上是在执行do-while循环，相当于任务一直在执行，因此不会退出。

### 早期AFNetworking为什么需要线程保活

​     首先需要在子线程去start connection，请求发送后，所在的子线程需要保活以保证正常接收到 NSURLConnectionDelegate 回调方法。如果每来一个请求就开一条线程，并且保活线程，这样开销太大了。所以只需要保活一条固定的线程，在这个线程里发起请求、接收回调。

- 早期的AFNetworking(2.x)网络请求是基于`NSURLConnection`实现的；`NSURLConnection`被设计成异步发送，调用了`-start`方法后，`NSURLConnection`会新建一些线程用底层的`CFSocket`去发送和接收请求，在发送和接收的一些事件发生后通知原来线程的`RunLoop`去回调事件。也就是说`NSURLConnection`的代理回调，也是通过`RunLoop`触发的；

- 平常我们自己使用`NSURLConnection`实现网络请求时，`NSURLConnection`的创建与回调一般都是在主线程，主线程本来一直存在所有回调没有问题；

- AFN作为网络层框架，在`NSURLConnection`回调回来之后，对`Response`做了一些诸如序列化、错误处理的操作的，这些操作都放在子线程去做，处理后接着回到主线程，再通过AFN自己的代理回调给用户；

- AFN的接收`NSURLConnection`回调的这个线程，正常情况下在执行`[connection start]`发送网络请求后就立即退出了，后续的回调就调用不了；而线程保活就能确保该线程不退出，回调成功。


```objectivec
- (void)operationDidStart {
    [self.lock lock];
    if (![self isCancelled]) {
        self.connection = [[NSURLConnection alloc] initWithRequest:self.request delegate:self startImmediately:NO];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        for (NSString *runLoopMode in self.runLoopModes) {
            [self.connection scheduleInRunLoop:runLoop forMode:runLoopMode];
            [self.outputStream scheduleInRunLoop:runLoop forMode:runLoopMode];
        }
        [self.outputStream open];
        [self.connection start];
    }
    [self.lock unlock];
    dispatch_async(dispatch_get_main_queue(), ^{
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNetworkingOperationDidStartNotification object:self];
    });
}
```

首先，每一个请求对应一个AFHTTPRequestOperation实例对象（以下简称operation），每一个operation在初始化完成后都会被添加到一个NSOperationQueue中。
由这个NSOperationQueue来控制并发，系统会根据当前可用的核心数以及负载情况动态地调整最大的并发 operation 数量，我们也可以通过setMaxConcurrentoperationCount:方法来设置最大并发数。注意：并发数并不等于所开辟的线程数。具体开辟几条线程由系统决定。也就是说此处执行operation是并发的、多线程的。

### AFNetworking的线程保活

参考前面线程与RunLoop的关系，AFNetworking早期线程保活也采用了类似的处理方式：

```Objective-C
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];      // 获取当前RunLoop
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
```

创建常驻线程

```Objective-C
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}
```

`_networkRequestThread`就是创建的常驻线程，在线程里获取RunLoop并运行起来；所以这个线程不会被退出、销毁，除非RunLoop停止；这样就实现了常驻线程保活。

### AFNetworking 3.x不再需要线程保活

AFNetworking 3.x是基于NSUrlSession实现的，NSUrlSession内部自己维护了一个线程池，做Request线程的调度与管理；因此AFN3.x无需常驻线程，只是用的时候CFRunLoopRun();开启RunLoop，结束的时候CFRunLoopStop(CFRunLoopGetCurrent());`停止RunLoop即

苹果也是明白了这一痛点，从iOS9.0开始 deprecated 了NSURLConnection。 替代方案就是NSURLSession。当然NSURLSession还解决了很多其他的问题，这里不作赘述。

```objectivec
self.operationQueue = [[NSOperationQueue alloc] init];
self.operationQueue.maxConcurrentOperationCount = 1;
self.session = [NSURLSession sessionWithConfiguration:self.sessionConfiguration delegate:self delegateQueue:self.operationQueue];
```

为什么说NSURLSession解决了NSURLConnection的痛点，从上面的代码可以看出，NSURLSession发起的请求，不再需要在当前线程进行代理方法的回调！可以指定回调的delegateQueue，这样我们就不用为了等待代理回调方法而苦苦保活线程了。

同时还要注意一下，指定的用于接收回调的Queue的maxConcurrentOperationCount设为了1，这里目的是想要让并发的请求串行的进行回调。

为什么要串行回调？

```objectivec
- (AFURLSessionManagerTaskDelegate *)delegateForTask:(NSURLSessionTask *)task {
    NSParameterAssert(task);
    AFURLSessionManagerTaskDelegate *delegate = nil;
    [self.lock lock];
    //给所要访问的资源加锁，防止造成数据混乱
    delegate = self.mutableTaskDelegatesKeyedByTaskIdentifier[@(task.taskIdentifier)];
    [self.lock unlock];
    return delegate;
}
```

这边对 self.mutableTaskDelegatesKeyedByTaskIdentifier 的访问进行了加锁，目的是保证多线程环境下的数据安全。既然加了锁，就算maxConcurrentOperationCount不设为1，当某个请求正在回调时，下一个请求还是得等待一直到上个请求获取完所要的资源后解锁，所以这边并发回调也是没有意义的。相反多task回调导致的多线程并发，还会导致性能的浪费。

![img](https:////upload-images.jianshu.io/upload_images/1458949-dbf68e497c9add5b.png?imageMogr2/auto-orient/strip|imageView2/2/w/342/format/webp)

##### 补充1：AF3.x会给每个 NSURLSessionTask 绑定一个 AFURLSessionManagerTaskDelegate ，这个TaskDelegate相当于把NSURLSessionDelegate进行了一层过滤，最终只保留类似didCompleteWithError这样对上层调用者输出的回调。

```objectivec
- (void)URLSession:(__unused NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
    //此处代码进行了大量删减，只是为了让大家清楚的看到这个方法做的最重要的事
    dispatch_group_async(manager.completionGroup ?: url_session_manager_completion_group(), manager.completionQueue ?: dispatch_get_main_queue(), ^{
        if (self.completionHandler) {
            self.completionHandler(task.response, responseObject, error);
        }
    }
}
```

##### 补充2：面试官可能会问你：为什么AF3.0中需要设置

```objectivec
self.operationQueue.maxConcurrentOperationCount = 1;
```

而AF2.0却不需要？

这个问题不难，但是却可以帮助面试官判断面试者是否真的认真研读了AF的两个大版本的源码。
 解答：功能不一样：AF3.0的operationQueue是用来接收NSURLSessionDelegate回调的，鉴于一些多线程数据访问的安全性考虑，设置了maxConcurrentOperationCount = 1来达到串行回调的效果。
 而AF2.0的operationQueue是用来添加operation并进行并发请求的，所以不要设置为1。

```objectivec
- (AFHTTPRequestOperation *)POST:(NSString *)URLString
                      parameters:(id)parameters
                         success:(void (^)(AFHTTPRequestOperation *operation, id responseObject))success
                         failure:(void (^)(AFHTTPRequestOperation *operation, NSError *error))failure
{
    AFHTTPRequestOperation *operation = [self HTTPRequestOperationWithHTTPMethod:@"POST" URLString:URLString parameters:parameters success:success failure:failure];
    [self.operationQueue addOperation:operation];
    return operation;
}
```

##### 补充3：AF中常驻线程的实现（经典案例，不作赘述）

```objectivec
+ (NSThread *)networkRequestThread {
    static NSThread *_networkRequestThread = nil;
    static dispatch_once_t oncePredicate;
    dispatch_once(&oncePredicate, ^{
        _networkRequestThread = [[NSThread alloc] initWithTarget:self selector:@selector(networkRequestThreadEntryPoint:) object:nil];
        [_networkRequestThread start];
    });
    return _networkRequestThread;
}
```

首先用NSThread创建了一个线程，并且这个线程是个单例。

```objectivec
+ (void)networkRequestThreadEntryPoint:(id)__unused object {
    @autoreleasepool {
        [[NSThread currentThread] setName:@"AFNetworking"];
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
        [runLoop run];
    }
}
```

新建的子线程默认是没有添加Runloop的，因此给这个线程添加了一个runloop，并且加了一个NSMachPort，来防止这个新建的线程由于没有活动直接退出。