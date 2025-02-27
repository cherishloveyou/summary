# dispatch_queue

Dispatch 的核心是队列，分为并行和串行两种，**主队列是典型的串行队列**。

**Queue（队列）：**队列分为串行和并行。串行队列按照A、B、C、D的顺序添加四个任务，这四个任务按照顺序执行，结束顺序也肯定是A、B、C、D，而并行队列同时执行这四个任务，完成的顺序因此也是随机的。

**异步执行(async)和同步执行(sync)：**使用dispatch_async调用一个block，这个block会被放到**指定的queue队列尾等待执行**，至于这个**block是被并行还是串行执行，只和dispatch_async中的指定的queue有关，但是dispatch_async会马上返回。**

使用dispatch_sync同样也是把block放到指定的queue上执行，但是会**等待这个block执行完毕后才返回**，这期间会阻塞当前运行调用dispatch_async或dispatch_sync代码的queue（通常为main_queue）直到sync函数返回。

**dispatch_sync  ：调用用  dispatch_sync的线程会等dispatch_sync的对内容执行完再继续执行。**

**dispatch_async ：调用dispatch_async的线程不会的等dispatch_async的内容，自己继续执行。**

**sync/async的区别在于 调用diapatch的线程是否等待dispatch执行完。**



Serial Dispatch Queue（串行队列）：等待正在执行中的处理结束，再执行下一条处理。

Concurrent Dispatch Queue（并发队列）：不等待现在执行中的处理是否结束，继续执行下面的处理。**只有在异步执行中，才能体现并发性**

- 同步执行。不开启新的线程

- 异步执行。开启新的线程

  

**『**主线程』中，『不同队列』+『不同任务』简单组合的区别：

|     区别      |           并发队列           |             串行队列              |            主队列            |
| :-----------: | :--------------------------: | :-------------------------------: | :--------------------------: |
| 同步（sync）  | 没有开启新线程，串行执行任务 |   没有开启新线程，串行执行任务    |        死锁卡住不执行        |
| 异步（async） |  有开启新线程，并发执行任务  | 有开启新线程（1条），串行执行任务 | 没有开启新线程，串行执行任务 |



**『不同队列』+『不同任务』** 组合，以及 **『队列中嵌套队列』** 使用的区别：

|     区别      | 『异步执行+并发队列』嵌套『同一个并发队列』 | 『同步执行+并发队列』嵌套『同一个并发队列』 | 『异步执行+串行队列』嵌套『同一个串行队列』 | 『同步执行+串行队列』嵌套『同一个串行队列』 |
| :-----------: | :-----------------------------------------: | :-----------------------------------------: | :-----------------------------------------: | :-----------------------------------------: |
| 同步（sync）  |       没有开启新的线程，串行执行任务        |        没有开启新线程，串行执行任务         |               死锁卡住不执行                |               死锁卡住不执行                |
| 异步（async） |         有开启新线程，并发执行任务          |         有开启新线程，并发执行任务          |     有开启新线程（1 条），串行执行任务      |     有开启新线程（1 条），串行执行任务      |



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
sync:3
sync:4
sync:2
  
//RunLoop 会阻塞当前线程
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

//RunLoop 会阻塞当前线程 如果是main线，则阻塞死锁了 
```

```objectivec
// 前提条件：当前的 queue 为 main_queue
dispatch_sync(dispatch_get_main_queue(), ^{
    NSLog(@"mainQueue_sync:1");
});

//上述代码会造成死锁。原因：前提条件是当前 queue 为 main_queue。main_queue 为串行队列，在当前 queue 上调用 sync 函数。需要执行的 block 被放到当前 queue 的队尾等待被执行，因为这是一个串行的 queue，调用 sync函数会阻塞当前队列，等待block被执行->这个block一直不会被执行-> sync函数一直不返回，所以当前 queue 就被阻塞了，造成了死锁。
//一般串行队列中 sync 到自身上会产生死锁，sync 到其他队列上一般不会产生死锁，如在自定义 queue 中 sync main_queue，等到 main_queue 执行完毕再继续执行操作。
```

```objective-c
NSLog(@"sync:1");
dispatch_queue_t queue = dispatch_queue_create("test", DISPATCH_QUEUE_SERIAL);
dispatch_async(queue, ^{
    NSLog(@"sync:2");
    dispatch_sync(queue, ^{
      NSLog(@"sync:3");
   });
    NSLog(@"sync:4");
});
NSLog(@"sync:5");
  
 //打印1、5、2，然后就死锁了。原因是列serialQueue.async的block1被异步追加到串行队列上后，开始执行，这个block1中又被同步追加了一个block2，此时serialQueue被阻塞，等待block2执行完毕，但是block1还未执行完毕，由于是串行队列，block只能按照追加的先后顺序一个一个执行：线程被阻塞->block1停止执行->block2等block1执行完毕->因此就造成了死锁。

//通过dispatch_sync添加的任务，在哪个线程添加就会在哪个线程执行。因此向并发队列添加的任务，没有开启新线程，而是在主线程执行的
```

```objective-c
NSLog(@"sync:1");    
dispatch_queue_t queue = dispatch_queue_create("test", DISPATCH_QUEUE_SERIAL);
dispatch_async(queue, ^{ 
  NSLog(@"sync:2");
});    
NSLog(@"sync:3");    
dispatch_sync(queue, ^{ 
  NSLog(@"sync:4");
});    
NSLog(@"sync:5"); 

sync:1  
sync:3  
sync:2  
sync:4  
sync:5 
  
//sync:1 和 sync:3 是同步执行的，因此会立即打印。
//sync:2 是异步提交的任务，会在队列调度时执行。
//sync:4 是同步提交的任务，必须等待 sync:2 执行完成后才能执行。
//sync:5 是同步执行的代码，会在sync:4 执行完成后立即打印。
```



#### dispatch_barrier_async

- 这个函数传入的并发队列必须是自己通过dispatch_queue_create创建的
- 如果传入的是一个串行或是一个全局并发队列，那这个函数变等同于dispatch_async的效果

```objective-c
dispatch_queue_t queue = dispatch_queue_create("rw", DISPATCH_QUEUE_CONCURRENT);
dispatch_async(queue, ^{
    //读
    [self read];
});
dispatch_barrier_async(queue, ^{
    [self write];  // 写
});
```





#### （1）串行队列+同步执行

```objectivec
- (void)KSserialQueueSync {
    NSLog(@"test start");
    
    dispatch_queue_t serialQueue = dispatch_queue_create("com.ks.serialQueue", NULL);
    
    dispatch_sync(serialQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block1 %@", [NSThread currentThread]);
        }
    });
    dispatch_sync(serialQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block2 %@", [NSThread currentThread]);
        }
    });
    dispatch_sync(serialQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block3 %@", [NSThread currentThread]);
        }
    });
    NSLog(@"test over");
}
```

```
因为是同步执行，所以不创建新的线程，在主线程中执行。
因为是串行队列，所以队列的任务一个接一个地执行。
因为所有任务都在test start和test over之间执行，所以说明任务一加入队列就立马执行。

[915:40526] test start
[915:40526] block1 <NSThread: 0x610000069e80>{number = 1, name = main}
[915:40526] block1 <NSThread: 0x610000069e80>{number = 1, name = main}
[915:40526] block2 <NSThread: 0x610000069e80>{number = 1, name = main}
[915:40526] block2 <NSThread: 0x610000069e80>{number = 1, name = main}
[915:40526] block3 <NSThread: 0x610000069e80>{number = 1, name = main}
[915:40526] block3 <NSThread: 0x610000069e80>{number = 1, name = main}
[915:40526] test over
```



#### （2）串行队列+异步执行

```objectivec
- (void)KSserialQueueAsync {
    NSLog(@"test start");
    
    dispatch_queue_t serialQueue = dispatch_queue_create("com.ks.serialQueue", NULL);
    dispatch_async(serialQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block1 %@", [NSThread currentThread]);
        }
    });
    dispatch_async(serialQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block2 %@", [NSThread currentThread]);
        }
    });
    dispatch_async(serialQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block3 %@", [NSThread currentThread]);
        }
    });
    
    NSLog(@"test over");
}

```

```
因为是异步执行，所以创建了新的线程。
因为是串行队列，所以队列中的任务一个接一个执行。
因为所有队列任务执行在test start和test over之后，说明任务不是添加到队列之后立马执行，而是当所有任务添加到队列之后再执行。
任务是按顺序执行的（串行队列 每次只有一个任务被执行，任务一个接一个按顺序执行）。

[941:46623] test start
[941:46623] test over
[941:46686] block1 <NSThread: 0x61800007a100>{number = 2, name = (null)}
[941:46686] block1 <NSThread: 0x61800007a100>{number = 2, name = (null)}
[941:46686] block2 <NSThread: 0x61800007a100>{number = 2, name = (null)}
[941:46686] block2 <NSThread: 0x61800007a100>{number = 2, name = (null)}
[941:46686] block3 <NSThread: 0x61800007a100>{number = 2, name = (null)}
[941:46686] block3 <NSThread: 0x61800007a100>{number = 2, name = (null)}
```



#### （3）并发队列+同步执行

```objectivec
- (void)KSconcurrentQueueSync {
    NSLog(@"test start");
    
    dispatch_queue_t concurrentQueue = dispatch_queue_create("com.ks.concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_sync(concurrentQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block1 %@", [NSThread currentThread]);
        }
    });
    
    dispatch_sync(concurrentQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block2 %@", [NSThread currentThread]);
        }
    });
    
    dispatch_sync(concurrentQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block3 %@", [NSThread currentThread]);
        }
    });
    
    NSLog(@"test over");
}
```



```csharp
输出结果分析：
- 因为是同步执行，不创建新的线程，在主线程中执行。
- 虽然是并发队列，但因为是同步执行，不能创建新线程，只有当前线程这一个线程，没有体现出并发性，任务还是一个接一个执行。
- 因为所有任务都在`test start`和`test over`之间执行，所以说明任务一加入队列就立马执行。
  
[1001:58076] test start
[1001:58076] block1 <NSThread: 0x60000006da80>{number = 1, name = main}
[1001:58076] block1 <NSThread: 0x60000006da80>{number = 1, name = main}
[1001:58076] block2 <NSThread: 0x60000006da80>{number = 1, name = main}
[1001:58076] block2 <NSThread: 0x60000006da80>{number = 1, name = main}
[1001:58076] block3 <NSThread: 0x60000006da80>{number = 1, name = main}
[1001:58076] block3 <NSThread: 0x60000006da80>{number = 1, name = main}
[1001:58076] test over
```



#### （4）并发队列+异步执行

```objectivec
- (void)KSconcurrentQueueAsync {
    NSLog(@"test start");
    
    dispatch_queue_t concurrentQueue = dispatch_queue_create("com.ks.concurrentQueue", DISPATCH_QUEUE_CONCURRENT);
    
    dispatch_async(concurrentQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block1 %@", [NSThread currentThread]);
        }
    });
    
    dispatch_async(concurrentQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block2 %@", [NSThread currentThread]);
        }
    });
    
    dispatch_async(concurrentQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block3 %@", [NSThread currentThread]);
        }
    });
    
    NSLog(@"test over");
}
```

```csharp
输出结果分析：

- 因为异步执行，所以创建了新的线程。
- 因为并发队列，异步执行时体现其并发性，任务之间交替着同时执行。
- 因为所有队列任务执行在`test start`和`test over`之后，说明任务不是添加到队列之后立马执行，而是当所有任务添加到队列之后再执行。
[1042:64557] test start
[1042:64557] test over
[1042:64615] block3 <NSThread: 0x610000067c00>{number = 4, name = (null)}
[1042:64618] block2 <NSThread: 0x608000067e80>{number = 3, name = (null)}
[1042:64640] block1 <NSThread: 0x610000067d00>{number = 2, name = (null)}
[1042:64615] block3 <NSThread: 0x610000067c00>{number = 4, name = (null)}
[1042:64618] block2 <NSThread: 0x608000067e80>{number = 3, name = (null)}
[1042:64640] block1 <NSThread: 0x610000067d00>{number = 2, name = (null)}
```

#### （5）主队列+同步执行

```objectivec
- (void)KSmainQueueSync {
    NSLog(@"test start");
    
    dispatch_queue_t mainQueue = dispatch_get_main_queue();
    
    dispatch_sync(mainQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block1 %@", [NSThread currentThread]);
        }
    });
    
    dispatch_sync(mainQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block2 %@", [NSThread currentThread]);
        }
    });
    
    dispatch_sync(mainQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block3 %@", [NSThread currentThread]);
        }
    });
    
    NSLog(@"test over");
}
```



```bash
[1084:70872] test start
```

输出结果分析：

- 只输出了一条语句，之后的语句都没有执行。
- 主队列是主线程的一条队列。
- 发生死锁：我们知道同步执行，就是要立马执行（参见串行队列同步执行和并发队列同步执行的结果分析第三条）。但是现在主队列无法立马执行，因为当前主线程正在执行的任务是`KSmainQueueSync`这个方法，需要等待这个方法执行完；但是`KSmainQueueSync`这个方法又要等待第一个第二个第三个任务执行完。相互等待而造成死锁。

**只要在另一条线程上调用就好了，利用串行队列+异步执行来创建一条新的线程**

```objective-c
dispatch_queue_t queue = dispatch_queue_create("com.ks.serialQueue", NULL);
dispatch_async(queue, ^{
    [self KSmainQueueSync];
});
```

```csharp
输出结果分析：主队列是串行队列的一种，所以之前对串行队列的分析这里也适用
- 因为是同步执行，所以不创建新的线程，在主线程中执行。
- 因为是串行队列，所以队列的任务一个接一个地执行。
- 因为所有任务都在`test start`和`test over`之间执行，所以说明任务一加入队列就立马执行。
[1097:77638] test start
[1097:77593] block1 <NSThread: 0x6080000699c0>{number = 1, name = main}
[1097:77593] block1 <NSThread: 0x6080000699c0>{number = 1, name = main}
[1097:77593] block2 <NSThread: 0x6080000699c0>{number = 1, name = main}
[1097:77593] block2 <NSThread: 0x6080000699c0>{number = 1, name = main}
[1097:77593] block3 <NSThread: 0x6080000699c0>{number = 1, name = main}
[1097:77593] block3 <NSThread: 0x6080000699c0>{number = 1, name = main}
[1097:77638] test over
```



#### （6）主队列+异步执行

```objectivec
- (void)KSmainQueueAsync {
    NSLog(@"test start");
    dispatch_queue_t mainQueue = dispatch_get_main_queue();
    dispatch_async(mainQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block1 %@", [NSThread currentThread]);
        }
    });
    dispatch_async(mainQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block2 %@", [NSThread currentThread]);
        }
    });
    dispatch_async(mainQueue, ^{
        for (int i = 0; i < 2; i++) {
            NSLog(@"block3 %@", [NSThread currentThread]);
        }
    });
    NSLog(@"test over");
}
```



```csharp
输出结果分析：
- 虽然是异步执行，可以开启新的线程，但因为是主队列，它只会在主线程中执行。（这点与普通串行队列有区别）
- 因为主队列是特殊的串行队列，所以队列的任务一个接一个地执行。
- 因为所有队列任务执行在`test start`和`test over`之后，说明任务不是添加到队列之后立马执行，而是当所有任务添加到队列之后再执行。
  
[1130:84261] test start
[1130:84261] test over
[1130:84261] block1 <NSThread: 0x60800006de40>{number = 1, name = main}
[1130:84261] block1 <NSThread: 0x60800006de40>{number = 1, name = main}
[1130:84261] block2 <NSThread: 0x60800006de40>{number = 1, name = main}
[1130:84261] block2 <NSThread: 0x60800006de40>{number = 1, name = main}
[1130:84261] block3 <NSThread: 0x60800006de40>{number = 1, name = main}
[1130:84261] block3 <NSThread: 0x60800006de40>{number = 1, name = main}
```


 归纳注意事项：
 （1）创建一个队列与创建一个线程是不同的两件事。
 （2）在一个线程内可能会有多个队列，混杂有串行队列和并行队列。
 （3）是否创建新线程，取决于队列是同步执行还是异步执行。
 （4）死锁问题。

