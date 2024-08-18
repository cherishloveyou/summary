# dispatch_queue

Dispatch 的核心是队列，分为并行和串行两种，**主队列是典型的串行队列**。

**Queue（队列）：**队列分为串行和并行。串行队列按照A、B、C、D的顺序添加四个任务，这四个任务按照顺序执行，结束顺序也肯定是A、B、C、D，而并行队列同时执行这四个任务，完成的顺序因此也是随机的。

**异步执行(async)和同步执行(sync)：**使用dispatch_async调用一个block，这个block会被放到**指定的queue队列尾等待执行**，至于这个block是被并行还是串行执行，只和dispatch_async中的指定的queue有关，但是dispatch_async会马上返回。

使用dispatch_sync同样也是把block放到指定的queue上执行，但是会**等待这个block执行完毕后才返回**，这期间会阻塞当前运行调用dispatch_async或dispatch_sync代码的queue（通常为main_queue）直到sync函数返回。

```objective-c
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"async:1");
});
NSLog(@"async:2");
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"async:3");
});
NSLog(@"async:4");

async:2
async:4
async:1
async:3
```

```objectivec
dispatch_sync(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"sync:1");
});
NSLog(@"sync:2");
dispatch_sync(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
    NSLog(@"sync:3");
});
NSLog(@"sync:4");

sync:1
sync:2
sync:3
sync:4
```

```objective-c
NSLog(@"sync:1");
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSTimer *timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(test) userInfo:nil repeats:NO];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
    [[NSRunLoop currentRunLoop] run];
    NSLog(@"sync:2");
});
NSLog(@"sync:3");
- (void)test {
    NSLog(@"sync:4");
}
sync:1
sync:4
sync:3
sync:2
```

```objective-c
NSLog(@"sync:1");
dispatch_async(dispatch_get_main_queue(), ^{
    NSTimer *timer = [NSTimer timerWithTimeInterval:1 target:self selector:@selector(test) userInfo:nil repeats:NO];
    [[NSRunLoop currentRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
    [[NSRunLoop currentRunLoop] run];
    NSLog(@"sync:2");
});
NSLog(@"sync:3");
- (void)test {
    NSLog(@"sync:4");
}
sync:1
sync:4
sync:3
sync:2
```

```objectivec
// 前提条件：当前的 queue 为 main_queue
dispatch_sync(dispatch_get_main_queue(), ^{
    NSLog(@"mainQueue_sync:1");
});

上述代码会造成死锁。原因：前提条件是当前 queue 为 main_queue。main_queue 为串行队列，在当前 queue 上调用 sync 函数。需要执行的 block 被放到当前 queue 的队尾等待被执行，因为这是一个串行的 queue，调用 sync 函数会阻塞当前队列，等待 block 被执行->这个 block 一直不会被执行-> sync 函数一直不返回，所以当前 queue 就被阻塞了，造成了死锁。
一般串行队列中 sync 到自身上会产生死锁，sync 到其他队列上一般不会产生死锁，如在自定义 queue 中 sync main_queue，等到 main_queue 执行完毕再继续执行操作。
```

```objective-c
print(1)
serialQueue.async {
    print(2)
    serialQueue.sync {
        print(3)
    }
  print(4)
}
print(5)
  
 打印1、5、2，然后就死锁了。原因是列serialQueue.async的block1被异步追加到串行队列上后，开始执行，这个block1中又被同步追加了一个block2，此时serialQueue被阻塞，等待block2执行完毕，但是block1还未执行完毕，由于是串行队列，block只能按照追加的先后顺序一个一个执行：线程被阻塞->block1停止执行->block2等block1执行完毕->因此就造成了死锁。

通过dispatch_sync添加的任务，在哪个线程添加就会在哪个线程执行。因此向并发队列添加的任务，没有开启新线程，而是在主线程执行的
```
