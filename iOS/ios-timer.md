## iOS Timer

NSTimer

GCD定时器

dispatch_after

(void)performSelector:(SEL)aSelector withObject:(nullable id)anArgument afterDelay:(NSTimeInterval)delay;

#### NSTimer

NSTimer的初始化方式有几下几种。我们注意到分为invocation和selector两种调用方式，其实这两种区别不大，一般我们用selector方式较为方便。

```objectivec
+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)ti invocation:(NSInvocation *)invocation repeats:(BOOL)yesOrNo;
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti invocation:(NSInvocation *)invocation repeats:(BOOL)yesOrNo;

+ (NSTimer *)timerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(nullable id)userInfo repeats:(BOOL)yesOrNo;
+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)ti target:(id)aTarget selector:(SEL)aSelector userInfo:(nullable id)userInfo repeats:(BOOL)yesOrNo;
```

##### invocation方式

```objectivec
- (void)viewDidLoad {
    [super viewDidLoad];

    NSMethodSignature  *signature = [[self class] instanceMethodSignatureForSelector:@selector(Timered:)];
    NSInvocation* invocation = [NSInvocation invocationWithMethodSignature:signature];
    invocation.target = self;
    invocation.selector = @selector(Timered:);
    NSTimer* timer = [NSTimer scheduledTimerWithTimeInterval:1 invocation:invocation repeats:YES];
}

- (void)Timered:(NSTimer*)timer {
    NSLog(@"timer called");
}
```

##### selector方式：

```objectivec
- (void)viewDidLoad {
    [super viewDidLoad];

    NSTimer* timer = [NSTimer scheduledTimerWithTimeInterval:1 target:self selector:@selector(Timered:) userInfo:nil repeats:YES];
}

- (void)Timered:(NSTimer*)timer {
    NSLog(@"timer called");
}
```

##### scheduledTimerWith和timerWith和区别

那每种方式的调用接口又分为scheduledTimerWith和timerWith是为什么呢？这是因为NSTimer是加到runloop中执行的。看scheduledTimerWith的函数说明，创建并安排到runloop的default mode中。

> Creates a timer and schedules it on the current run loop in the default mode.

如果我们调用的是timerWith接口，就需要自己加入runloop。

> You must add the new timer to a run loop, using [addTimer:forMode:](https://links.jianshu.com/go?to=apple-reference-documentation%3A%2F%2FhcocJkO-uk).

```objectivec
NSTimer *timer  =  [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(Timered) userInfo:nil repeats:YES];

[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSDefaultRunLoopMode];
```

##### 坑一：子线程启动定时器问题：

我们都知道iOS是通过runloop作为消息循环机制，主线程默认启动了runloop，可是子线程没有默认的runloop，因此，我们在子线程启动定时器是不生效的。

解决的方式也简单，在子线程启动一下runloop就可以了。

```objectivec
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        NSTimer* timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(Timered:) userInfo:nil repeats:YES];
        [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
        [[NSRunLoop currentRunLoop] run];
    });
```

##### 坑二：runloop的mode问题：

我们注意到schedule方式启动的timer是add到runloop的NSDefaultRunLoopMode中，这就会出现其他mode时timer得不到调度的问题。最常见的问题就是在UITrackingRunLoopMode，即UIScrollView滑动过程中定时器失效。

解决方式就是把timer add到runloop的NSRunLoopCommonModes。UITrackingRunLoopMode和kCFRunLoopDefaultMode都被标记为了common模式，所以只需要将timer的模式设置为NSRunLoopCommonModes，就可以在默认模式和追踪模式都能够运行。

##### 坑三：循环引用问题：

前两个都是小坑，因为对于大部分简单场景，是不会踩到的。但是循环引用问题，是每个使用者都会遇到的。

究其原因，就是NSTimer的target被强引用了，而通常target就是所在的控制器，他又强引用的timer，造成了循环引用。下面是target参数的说明：

> target： The object to which to send the message specified by `aSelector` when the timer fires. The timer maintains a strong reference to this object until it (the timer) is invalidated.

**注意：不是所有的NSTimer都会造成循环引用。就像不是所有的block都会造成循环引用一样。以下两种timer不会有循环引用：**

- 非repeat类型的。非repeat类型的timer不会强引用target，因此不会出现循环引用。
- block类型的，新api。iOS 10之后才支持，因此对于还要支持老版本的app来说，这个API暂时无法使用。当然，block内部的循环引用也要避免。

**不是解决了循环引用，target就可以释放了，别忘了在持有timer的类dealloc的时候执行invalidate。**

何解决循环引用：

##### 方式一：强引用的target变成了NSTimer的类对象

##### 方式二：NSProxy的方式

##### 方式三：封装timer，弱引用target



## GCD定时器

GCD中的Dispatch Source其中的一种类型是DISPATCH_SOURCE_TYPE_TIMER，可以实现定时器的功能。注意的是需要把timer声明为属性，否则，由于这种timer并不是添加到runloop中的，直接就被释放了。

GCD定时器的好处是，他并不是加入runloop执行的，因此子线程也可以使用。也不会引起循环引用的问题。

```objective-c
WS(weakSelf);
timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_main_queue());
dispatch_source_set_timer(timer, DISPATCH_TIME_NOW, 5 * NSEC_PER_SEC, 1 * NSEC_PER_SEC);
dispatch_source_set_event_handler(timer, ^{
    [weakSelf commentAnimation];
});
dispatch_resume(timer);
```

## dispatch_after

dispatch_after内部也是使用的Dispatch Source。因此也避免了NSTimer的很多坑。

```cpp
static inline void
_dispatch_after(dispatch_time_t when, dispatch_queue_t queue,
        void *ctxt, void *handler, bool block)
{
    dispatch_timer_source_refs_t dt;
    dispatch_source_t ds;
    uint64_t leeway, delta;

    if (when == DISPATCH_TIME_FOREVER) {
#if DISPATCH_DEBUG
        DISPATCH_CLIENT_CRASH(0, "dispatch_after called with 'when' == infinity");
#endif
        return;
    }

    delta = _dispatch_timeout(when);
    if (delta == 0) {
        if (block) {
            return dispatch_async(queue, handler);
        }
        return dispatch_async_f(queue, ctxt, handler);
    }
    leeway = delta / 10; // <rdar://problem/13447496>

    if (leeway < NSEC_PER_MSEC) leeway = NSEC_PER_MSEC;
    if (leeway > 60 * NSEC_PER_SEC) leeway = 60 * NSEC_PER_SEC;

    // this function can and should be optimized to not use a dispatch source
    ds = dispatch_source_create(&_dispatch_source_type_after, 0, 0, queue);
    dt = ds->ds_timer_refs;

    dispatch_continuation_t dc = _dispatch_continuation_alloc();
    if (block) {
        _dispatch_continuation_init(dc, ds, handler, 0, 0, 0);
    } else {
        _dispatch_continuation_init_f(dc, ds, ctxt, handler, 0, 0, 0);
    }
    // reference `ds` so that it doesn't show up as a leak
    dc->dc_data = ds;
    _dispatch_trace_continuation_push(ds->_as_dq, dc);
    os_atomic_store2o(dt, ds_handler[DS_EVENT_HANDLER], dc, relaxed);

    if ((int64_t)when < 0) {
        // wall clock
        when = (dispatch_time_t)-((int64_t)when);
    } else {
        // absolute clock
        dt->du_fflags |= DISPATCH_TIMER_CLOCK_MACH;
        leeway = _dispatch_time_nano2mach(leeway);
    }
    dt->dt_timer.target = when;
    dt->dt_timer.interval = UINT64_MAX;
    dt->dt_timer.deadline = when + leeway;
    dispatch_activate(ds);
}
```

1. 计算delta = _dispatch_timeout(when)；如果时间到直接dispatch_async执行。
2. dispatch_source_create创建source，并赋值给dispatch_continuation_t，dispatch_continuation_t会控制执行异步操作，将后续的时间上的操作都赋值给source的ds_timer_refs，并激活这个source。

##### performSelector：after

这种方式通常是用于在延时后去处理一些操作，其内部也是基于将timer加到runloop中实现的。因此也存在NSTimer的关于子线程runloop的问题。

这种调用方式的好处是可以取消。

```objectivec
- (void)cancelPreviousPerformRequestsWithTarget:(id)aTarget selector:(SEL)aSelector object:(nullable id)anArgument;
```

##### 延时一次操作的选择：

几种方式都是定时器，都可以实现延时操作。综合相比：如果只是单独一次的延时操作，NSTimer和GCD的定时器都显得有些笨重。performSelector方式比较合适，但是又收到了子线程runloop的限制。因此，dispatch_after是最优的选择。

##### 延时的取消操作：

以上几种方式都可以实现取消操作。

- NSTimer可以通过invalidate来停止定时器。
- GCD的定时器可以调用dispatch_suspend来挂起。
- performSelector：after可以通过cancelPreviousPerformRequestsWithTarget取消。
- dispatch_after可以通过dispatch_block_cancel来取消。

```objective-c
self.delayStartBlock = dispatch_block_create(0, ^{
    NSLog(@"把我取消");
});

dispatch_after(dispatch_time(DISPATCH_TIME_NOW, (int64_t)(5 * NSEC_PER_SEC)), dispatch_get_main_queue(),self.delayStartBlock);

dispatch_block_cancel(self.delayStartBlock);
```