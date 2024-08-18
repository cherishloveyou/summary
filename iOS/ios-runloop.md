# Runloop



### 什么是RunLoop

​      **RunLoop 是事件处理循环,在循环中用来处理程序运行过程中出现的各种事件。这些事件包括但不限于用户操作、定时器任务、内核消息、有顺序的处理各种Event。因为runLoop有状态，可以决定线程在什么时候处理什么事件，节省CPU资源。通常情况下，事件并不是永无休止的产生，也就没必要让线程永无休止的运行。runloop可以在无事件处理时进入休眠状态，避免无休止的do...while跑空圈。 一个线程对应一个RunLoop，程序运行是主线程的RunLoop默认启动，子线程的RunLoop按需启动（调用run方法）。runloop是线程的事件管理者，它会按照顺序管理线程要处理的事件，决定哪些事件在什么时候提交给主线程处理。**

```
什么是事件循环呢？
  事件循环（状态切换）
没有消息需要处理时，休眠以避免资源占用
  用户态——>内核态
有消息需要处理时，立刻被唤醒
  用户态<—— 内核态
   
什么是事件/消息进行管理呢？

RunLoop 通过 mach_msg()函数接收、发送消息来进行管理。
它的本质是调用函数 mach_msg_trap()，相当于是一个系统调用，会触发内核状态切换。
可以做到在有事做的时候做事，没事做的时候，会由用户态切换到内核态，避免资源浪费。

如何实现事件、消息的管理
mach_msg() 函数实际上是调用了一个 Mach 陷阱 (trap)，
即函数mach_msg_trap()，陷阱这个概念在 Mach 中等同于系统调用。
当你在用户态调用 mach_msg_trap() 时会触发陷阱机制，切换到内核态；
内核态中内核实现的 mach_msg() 函数会完成实际的工作，
```




### RunLoop基本作用

1. 保持程序持续运行，程序一启动就会开一个主线程，主线程一开起来就会跑一个主线程对应的RunLoop,RunLoop保证主线程不会被销毁，也就保证了程序的持续运行
2. 处理App中的各种事件（比如：触摸事件，定时器事件，Selector事件等）
3. 节省CPU资源，提高程序性能，程序运行起来时，当什么操作都没有做的时候，RunLoop就告诉CPU，现在没有事情做，我要去休息，这时CPU就会将其资源释放出来去做其他的事情，当有事情做的时候RunLoop就会立马起来去做事情

# RunLoop与线程关系

以下是获取主线程runloop和子线程runloop的函数。可以看出，这两个函数内部都调用了_CFRunLoopGet0()，CFRunLoopGet0()的入参是线程。

另外，获取子线程的runloop传入的是pthread_self()函数获取到的当前线程。所以这里可以看出，CFRunLoopGetCurrent函数必须要在线程内部调用，才能获取当前线程的RunLoop。也就是说子线程的RunLoop必须要在子线程内部获取。而主线程却没有这个限制，但是一般场景下也没有在子线程获取主线程runloop的必要。

- RunLoop和线程的一一对应的，对应的方式是以key-value的方式保存在一个全局字典中
- RunLoop 并不保证线程安全。我们只能在当前线程内部操作当前线程的 RunLoop 对象，而不能在当前线程内部去操作其他线程的 RunLoop 对象方法。
- 主线程的RunLoop会在初始化全局字典时创建
- 子线程的RunLoop会在第一次获取的时候创建，如果不获取的话就一直不会被创建
- RunLoop会在线程销毁时销毁

# CFRunLoopMode

mode作为runloop和source\timer\observer之间的桥梁。应用在启动时main runloop会注册5个mode。分别如下：

1. kCFRunLoopDefaultMode: App的默认 Mode，通常主线程是在这个 Mode 下运行的。

2. UITrackingRunLoopMode: 界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响。

3. UIInitializationRunLoopMode: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用。

4. GSEventReceiveRunLoopMode: 接受系统事件的内部 Mode，通常用不到。

5. kCFRunLoopCommonModes: 这是一个占位的 Mode，没有实际作用。

   

```
一个`Runloop`有一个`modes`
一个`modes` 是由多个`CFRunLoopMode`组成
一个`CFRunLoopMode`是由 `name/Source/Timer/Observer` 组成
`RunLoop` 只能运行一个 `Mode`,`RunLoop` 只会处理它当前 `Mode` 的事件
```

**每次调用 RunLoop  的主函数时，只能指定其中一个 Mode，这个Mode就是runloop的 CurrentMode。如果需要切换 Mode，只能退出 Loop再重新指定一个Mode 进入。这样做主要是为了分隔开不同组的Source/Timer/Observer，让其互不影响。**

mode中有一个特殊的mode叫做**commonMode**。commonMode并不是一个真正的mode，而是若干个被标记为commonMode的普通mode。所以commonMode本质上是一个集合，该集合存储的是mode的名字，也就是字符串，记录所有被标记为common的modeName。当我们向commonMode添加source\timer\observer时，本质上是遍历这个集合中的所有的mode，把item依次添加到每个被标记为common的mode中。

在程序启动时，主线程的runloop有两个预置的mode：kCFRunLoopDefaultMode 和 UITrackingRunLoopMode。默认情况下是会处于defaultMode，滑动scrollView列表时runloop会退出defaultMode转而进入trackingMode。所以，有时候我们加到defaultMode中的timer事件，在滑动列表时是不会执行的。不过，kCFRunLoopDefaultMode 和 UITrackingRunLoopMode这两个 Mode 都已经被添加到runloop的commonMode集合中。也就是说，主线程的这两个预置mode默认已经被标记为commonMode。想要我们的timer回调可以在滑动列表的时候依旧执行，只需要把timer这个item添加到commonMode。

## Source0和Source1区别

Source0：source0是App内部事件，由App自己管理。像UIEvent、CFSocket都是source0。source0并不能主动触发事件，当一个source0事件准备处理时，你需要先调用 CFRunLoopSourceSignal(source)，将这个 Source 标记为待处理，然后手动调用 CFRunLoopWakeUp(runloop) 来唤醒 RunLoop让其处理这个事件。框架已经帮我们做好了这些调用，比如网络请求的回调、滑动触摸的回调，我们不需要自己处理。

Source1：**由RunLoop和内核管理，Mach Port驱动**，来自系统内核或者其他进程或线程的事件，可以主动唤醒休眠中的RunLoop。(如CFMachPort、CFMessagePort)source1包含了一个 mach_port 和一个回调（函数指针），被用于通过内核和其他线程相互发送消息,这种 Source 能主动唤醒 RunLoop 的线程。

## UIButton点击事件是source0还是source1？

UIButton的点击事件到底是source0还是source1，这是很多人困惑的一点。断点打印堆栈看是从source0调出的，而有人说是source1。其实不难理解，我们上面说了source1是由runloop和内核管理，mach port驱动。所以button的点击事件首先是由source1 接收IOHIDEvent，然后再回调 IOHIDEventSystemClientQueueCallback() 内触发的source0，source0再触发的 _UIApplicationHandleEventQueue()。所以打印调用堆栈发现UIButton事件是source0触发的。我们可以在 IOHIDEventSystemClientQueueCallback() 处打一个 Symbolic Breakpoint 来验证这一点。

事实上，即便没有点击button，**只要触摸屏幕，就会产生一个CFRunLoopDoSource1 到IOHIDEventSystemClientQueueCallback() 的调用过程**。而点击按钮时，除了上述流程，会有一条新的调用从 GSEventRunModal -> CFRunLoopRunSpecific -> CFRunLoopDoSources0，所以看起来按钮点击事件还是直接触发的 source0事件。

# GCD和RunLoop的关系

当调用 dispatch_async(dispatch_get_main_queue(), block) 时，libDispatch  会向主线程的RunLoop发送消息，RunLoop会被唤醒，并从消息中取得这个 block，并在回调  **CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE**() 里执行这个  block。但这个逻辑仅限于 dispatch 到主线程，dispatch 到其他线程仍然是由 libDispatch 处理的。那么你肯定会问：为什么子线程没有这个和GCD交互的逻辑？原因有二：

- 主线程runloop是主线程的事件管理者。runloop负责何时让runloop处理何种事件。所有分发给主线程的任务必须统一交给主线程runloop排队处理。举例：UI操作只能在主线程，不在主线程操作UI会带来很多UI错乱问题以及UI更新延迟问题。
- 子线程不接受GCD的交互。因为子线程不一定会有runloop。

# AutoreleasePool和RunLoop的关系

App启动后，苹果在主线程 RunLoop 里注册了两个 Observer，其回调都是 _wrapRunLoopWithAutoreleasePoolHandler()。

第一个 Observer 监视的事件是 Entry(即将进入Loop)，其回调内会调用 _objc_autoreleasePoolPush() 创建自动释放池。其 order 是-2147483647，优先级最高，保证创建释放池发生在其他所有回调之前。

第二个 Observer 监视了两个事件： BeforeWaiting(准备进入休眠)  时调用_objc_autoreleasePoolPop() 和 _objc_autoreleasePoolPush()  释放旧的池并创建新池；Exit(即将退出Loop) 时调用 _objc_autoreleasePoolPop() 来释放自动释放池。这个  Observer 的 order 是 2147483647，优先级最低，保证其释放池子发生在其他所有回调之后。

在主线程执行的代码，通常是写在诸如事件回调、Timer回调内的。这些回调会被 RunLoop 创建好的 AutoreleasePool 环绕着，所以不会出现内存泄漏，开发者也不必显示创建 Pool 了。

# RunLoop应用

runloop在实际开发中有很多应用，通过源码可知苹果官方规定runloop一次只能运行在一个mode中，要想切换mode必须退出当前mode，以此来实现runloop不同mode的隔离。runloop的mode隔离实现了不同mode下任务的隔离。隔离了不同的mode就代表了隔离了不同mode下的source、timer、observer。这样做的好处是当前mode只会执行当前mode中的sourece、timer、observer，可以极大节省CPU资源。与所有任务都运行在一个mode中相比，这是一种很巧妙的设计。但苹果并没有将每个mode都严格的隔离，考虑到有些代码在不同的mode中都要执行的场景（比如，列表滚动时还要保证轮播图定时轮播），苹果又提供了一个名为commonMode的mode，这个mode不是一个真正的mode，而是多个mode的合集。兼顾性能的同时又给拓展留了活口，这种设计方式很值得学习。

## 苹果对runloop的使用

苹果在AutoreleasePool、手势识别、事件响应、UI更新、定时器、NSObject延时调用方法（performSelecter:afterDelay:) 、GCD、网络请求等方面都有使用RunLoop。

## AFNetworking对Runloop的使用

另外，众所周知，为了线程保活，AFNetworking内部也使用了runLoop：通过给子线程添加一个runloop来保证这个子线程不退出。这样，当需要这个子线程执行任务时，AFNetworking 通过调用 NSObject performSelector:onThread: 将任务抛给这个子线程的 RunLoop 即可。

## SDWebImage对Runloop的使用

SDWebImage中的动画播放类SDAnimatedImageView中也有runloop的影子。该类暴露了一个runloopMode属性，开发者可以指定动画播放的runloopMode，如果不指定则会使用内部默认的mode。这个runloopMode属性最终传递给了SDDisplayLink，SDDisplayLink在iOS平台上是对CADisplayLink的封装。最终通过调用CADisplyLink的实例方法- (void)addToRunLoop:(NSRunLoop *)runloop forMode:(NSRunLoopMode)mode;将的runloopMode设置给CADisplayLink。

其他有使用runLoop的地方还有卡顿监控、异步绘制等。总之，只要我们想要保活线程能够随时处理任务，这个线程必须要有runloop。


RunLoop 是 iOS 应用于线程中的一种循环机制。系统本身没有创建 RunLoop 的 API，不过可以通过 currentRunLoop 获取当前 RunLoop。主线程本身就存在一个 RunLoop，而且是运行转态，子线程的 RunLoop 需要手动开启，否知无法监听到输入源与定时源。子线程 RunLoop 随着所在子线程的事件源结束而关闭，随着所在子线程的结束而释放。

##### 获取/创建RunLoop对象

NSRunLoop 对象不是线程安全，如果在不同线程使用同一个 RunLoop 对象，可以用 CFRunLoopRef，保证线程安全。

```objective-c
[NSRunLoop currentRunLoop]; // 当前线程的runLoop
[NSRunLoop mainRunLoop]; // 主线程runLoop

CFRunLoopGetCurrent();
CFRunLoopGetMain();

[NSRunLoop currentRunLoop].getCFRunLoop; //NSRunLoop转CFRunLoopRef
```

##### 添加定时器及输入源 

```objective-c
[[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
[[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];

CFRunLoopAddTimer(runLoopRef, runLoopTimerRef, kCFRunLoopDefaultMode);
CFRunLoopAddSource(runLoopRef, runLoopSourceRef, kCFRunLoopDefaultMode);
```

##### 启动RunLoop

[[NSRunLoop currentRunLoop] run]; // 无条件且以默认的NSDefaultRunLoopMode启动
[[NSRunLoop currentRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:2.0]]; // 指定过期时间且以默认的NSDefaultRunLoopMode启动
[[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate dateWithTimeIntervalSinceNow:2.0]]; // 指定过期时间，指定启动方式

CFRunLoopRun(); // 子线程的runLoop需要启动
CFRunLoopRunInMode(kCFRunLoopDefaultMode, timeInterval, returnAfterSourceHandled);

##### 退出RunLoop

子线程的 RunLoop 会随着事件源的结束而 Exit，所以一般不会主动去停止 RunLoop。

/* 给RunLoop设置超时时间 */
[[NSRunLoop currentRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:2.0]]; //指定过期时间且以默认的NSDefaultRunLoopMode启动

/* 通知RunLoop停止  */
CFRunLoopStop([NSRunLoop currentRunLoop].getCFRunLoop); 

##### 主线程场景

可直接使用 timer，默认为 NSDefaultRunLoopMode。

NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(fireDemo) userInfo:nil repeats:YES];
UIScrollView 滑动时 RunLoop 切到了 UITrackingRunLoopMode 模式,默认为 NSDefaultRunLoopMode 的 Timer 不会调用，设置为 NSRunLoopCommonModes 就可以恢复正常。 

[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];

##### 子线程场景

开启子线程 RunLoop 时必须添加相应的 timer 或 port，否知运行无效。**子线程 RunLoop 启动时，runMode 之后的代码不会执行，直到 RunLoop 随着 Timer 结束而 Exit 后，才执行 runMode 之后的代码；timer 的 repeats 为 YES 时，不会执行 runMode 之后的代码**；子线程 RunLoop 没有 UITrackingRunLoopMode 模式。

一旦调用 [currentRunLoop run] 这个方法开启子线程的运行循环就不能停止，阻塞 run 之后代码执行，子线程也不会被释放。 

```objective-c
- (void)timerCount {
    NSLog(@"3");
}
NSLog(@"1");
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(timerCount) userInfo:nil repeats:NO];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
    [[NSRunLoop currentRunLoop] run];
    // timer结束后，触发breakPoint
    NSLog(@"2");
});
NSLog(@"4");
repeats==NO  
[[NSRunLoop currentRunLoop]run];    //print: 1=>4=>3=>2  卡住当前列队等待timer执行完
[[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];                    //print: 1=>4=>3=>2  卡住当前线程等待timer执行完
[[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate dateWithTimeIntervalSinceNow:0.5]]; //print: 1=>4=>2  runloop失效导致Timer不执行

repeats==YES 
[[NSRunLoop currentRunLoop]run];    //print: 1=>4=>3=>3=>3... 循环不处理run之后的事情
[[NSRunLoop currentRunLoop]runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];                    //print: 1=>4=>3=>3=>3... 循环不处理run之后的事情
[[NSRunLoop currentRunLoop]runMode:NSDefaultRunLoopMode beforeDate:[NSDate dateWithTimeIntervalSinceNow:0.5]]; //print: 1=>4=>2 runloop失效导致Timer不执行

```

```objective-c
- (void)timerCount {
    NSLog(@"3");
}
NSLog(@"1");
dispatch_async(dispatch_get_main_queue(), ^{
    NSTimer *timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(timerCount) userInfo:nil repeats:NO];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode]; //添加timer，使用默认runloop模式
    [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
    // timer结束后，不触发breakPoint
    NSLog(@"2");
});
NSLog(@"4");

repeats==NO  
[[NSRunLoop currentRunLoop]run];    //print: 1=>4=>3    main_queue不处理run之后的事情
[[NSRunLoop currentRunLoop]runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];                    //print: 1=>4=>2=>3 main_queue runloop不受影响
[[NSRunLoop currentRunLoop]runMode:NSDefaultRunLoopMode beforeDate:[NSDate dateWithTimeIntervalSinceNow:0.5]]; //print: 1=>4=>2=>3 main_queue runloop不受影响
repeats==YES 
[[NSRunLoop currentRunLoop]run];    //print: 1=>4=>3=>3=>3...
[[NSRunLoop currentRunLoop]runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];                    //print: 1=>4=>2=>3=>3...
[[NSRunLoop currentRunLoop]runMode:NSDefaultRunLoopMode beforeDate:[NSDate dateWithTimeIntervalSinceNow:0.5]]; //print: 1=>4=>2=>3=>3...
```

调用 runUntilDate 会阻塞之后的代码执行，直到指定的时间过后才恢复执行。

```objective-c
[[NSRunLoop currentRunLoop] runUntilDate:[NSDate dateWithTimeIntervalSinceNow:3.0]];
// do something
```

##### 创建常驻子线程

```objective-c
+ (NSThread *)networkThread {
  static dispatch_once_t onceToken;
  static NSThread *_networkThread;
  dispatch_once(&onceToken, ^{
      _networkThread = [[NSThread alloc] initWithTarget:[[self class] sharedInstance] selector:@selector(_runLoopThread) object:nil];
      [_networkThread setName:@"com.network.request"];
      [_networkThread setQualityOfService:[[NSThread mainThread] qualityOfService]];
      [_networkThread start];
  });
  return _networkThread;
}

- (void)_runLoopThread {
  // 直接使用run比较暴力，无法手动销毁线程
  [[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
  [runLoop run]; 

  /* 另一种形式
  while (!_stopRunning) {
      @autoreleasepool {
          [[NSRunLoop currentRunLoop] addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
          [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
      }
  }
  */
}    
```

##### 同步等待

在主线程中可以用 RunLoop 实现同步等待的操作，不会造成线程的阻塞，实际应用中存在代码中断执行的情况，等待异步的执行结果可以使用信号量 wait-signal。

```objective-c
__block BOOL isContinue = NO;
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(2 * NSEC_PER_SEC)), dispatch_get_main_queue(), ^{
    isContinue = YES;
 });

while (isContinue == NO) {
    [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
 }
// isContinue为YES后执行后续代码
```

[**Runloop**](https://juejin.cn/post/6960941386828873758)

[当面试官问Runloop时,想听到的答案](https://juejin.cn/post/7081932582576717831)

https://blog.csdn.net/z119901214/article/details/82260795
