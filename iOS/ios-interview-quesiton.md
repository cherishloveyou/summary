#### 题目：

[**为什么Objective-C中有MetaClass这个设计？**](https://zhuanlan.zhihu.com/p/265409084)

[**Runtime原理探究**](https://www.jianshu.com/p/30de582dbeb7)

#### GCD实现多读单写

比如在内存中维护一份数据，有多处地方可能会同时操作这块数据，怎么能保证数据安全？这道题目总结得到要满足以下三点：

- 1.读写互斥
- 2.写写互斥
- 3.读读并发

```objective-c
@implementation KCPerson
- (instancetype)init{
    if (self = [super init]) {
       _concurrentQueue = dispatch_queue_create("com.xxx.syncQueue", DISPATCH_QUEUE_CONCURRENT);
       _dic = [NSMutableDictionary dictionary];
    }
    return self;
}

- (void)kc_setSafeObject:(id)object forKey:(NSString *)key{
    key = [key copy];
    dispatch_barrier_async(_concurrentQueue, ^{
       [_dic setObject:object key:key];
    });
}

- (id)kc_safeObjectForKey：:(NSString *)key{
    __block NSString *temp;
    dispatch_sync(_concurrentQueue, ^{
        temp =[_dic objectForKey：key];
    });
    return temp;
}
@end
```

- 首先我们要维系一个GCD 队列，最好不用全局队列，毕竟大家都知道全局队列遇到栅栏函数是有坑点的，这里就不分析了！
- 因为考虑性能 死锁 堵塞的因素不考虑串行队列，用的是自定义的并发队列！`_concurrentQueue = dispatch_queue_create("com.xxx.syncQueue", DISPATCH_QUEUE_CONCURRENT);`
- 首先我们来看看读操作:`kc_safeObjectForKey` 我们考虑到多线程影响是不能用异步函数的！说明：
  - 线程2 获取：`name` 线程3 获取 `age`
  - 如果因为异步并发，导致混乱 本来读的是`name` 结果读到了`age`
  - 我们允许多个任务同时进去! 但是读操作需要同步返回，所以我们选择:`同步函数` **（读读并发）**
- 我们再来看看写操作，在写操作的时候对key进行了copy, 关于此处的解释，插入一段来自参考文献的引用:

> 函数调用者可以自由传递一个NSMutableString的key，并且能够在函数返回后修改它。因此我们必须对传入的字符串使用`copy`操作以确保函数能够正确地工作。如果传入的字符串不是可变的（也就是正常的`NSString`类型），调用`copy`基本上是个空操作。

- 这里我们选择 `dispatch_barrier_async`, 为什么是栅栏函数而不是异步函数或者同步函数，下面分析：
  - 栅栏函数任务：之前所有的任务执行完毕，并且在它后面的任务开始之前，期间不会有其他的任务执行，这样比较好的促使 写操作一个接一个写 **（写写互斥）**，不会乱！
  - 为什么不是异步函数？应该很容易分析，毕竟会产生混乱！
  - 为什么不用同步函数？如果读写都操作了，那么用同步函数，就有可能存在：我写需要等待读操作回来才能执行，显然这里是不合理！

#### YYKit 中 YYThreadSafeArray 的实现

在 [YYKit/Utility](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fibireme%2FYYKit%2Ftree%2Fmaster%2FYYKit%2FUtility) 中实现了线程安全的可变数组/字典，其实现的思路是：
 ① 将 NSMutableArray 对象作为成员封装为一个新的类 YYThreadSafeArray
 ② 持有一个信号量对象作为数组操作的加锁控制

```objective-c
@implementation YYThreadSafeArray {
    NSMutableArray *_arr;  //Subclass a class cluster...
    dispatch_semaphore_t _lock;
}
```

③ 初始化时构造内部成员数组和信号量对象（使用宏定义实现）

```objective-c
// 通过宏定义实现带入外部代码实现初始化方法
#define INIT(...) self = super.init; \
if (!self) return nil; \
__VA_ARGS__; \
if (!_arr) return nil; \
_lock = dispatch_semaphore_create(1); \
return self;
- (instancetype)init {
    INIT(_arr = [[NSMutableArray alloc] init]);
}
```

④ 在进行修改和读取等操作时进行加锁（使用宏定义实现）

```objective-c
// 通过宏定义对代码块进行加锁操作
#define LOCK(...) dispatch_semaphore_wait(_lock, DISPATCH_TIME_FOREVER); \
__VA_ARGS__; \
dispatch_semaphore_signal(_lock);

// id obj = array[idx];
- (id)objectAtIndexedSubscript:(NSUInteger)idx {
    LOCK(id o = [_arr objectAtIndexedSubscript:idx]); return o;
}
// array[idx] = obj;
- (void)setObject:(id)obj atIndexedSubscript:(NSUInteger)idx {
    LOCK([_arr setObject:obj atIndexedSubscript:idx]);
}
```

> 关于信号量 `dispatch_semaphore_t` 是为了控制资源的访问频率使用，在 YYKit 的 `INIT` 宏定义实现中使用的信号量初始值为 **1**，在加锁操作前等待信号量，使用 `dispatch_semaphore_wait` 当信号量大于等于 1 时，减去 1 点信号值并开始执行后面的代码，此时信号值为 0，其他线程访问时没有信号值会一直等待，直到此任务完成后 `dispatch_semaphore_signal` 函数会将信号值加 1，其他线程的访问得以继续，从而实现信号量加锁的目的。

由于读写操作都使用了同一个信号量进行控制，可以得知此方案对可变数组的多线程操作是串行的，可以保证可变数组在多线程下访问的安全，即所有对数组的读写操作都将是依次逐个进行，潜在的问题是：限制了数组的多线程读取操作。

#### 可并行读取的线程安全数组

多线程写入和读取的加锁操作是必要的，如何在此基础上实现多线程并行读取操作？为此可以将数组的操作区分为写操作、读操作，需要满足以下要求：
 ① 在写入时，不能有其他读写操作
 ② 可以并行读取
 这些要求恰好可以使用 `Dispatch Concurrent Queue + dispatch_async_barrier` 加以实现，在同样的封装可变数组为成员变量的思路之后：
 ① 在初始化时，构造一个并行队列

```objective-c
@implementation ThreadSafeArray {
    NSMutableArray *_arr; 
    dispatch_queue_t _queue;
}

- (instancetype)init {
    ....
    _queue = dispatch_queue_create("unique.name", DISPATCH_QUEUE_CONCURRENT);
    ...
}
```

② 对写操作进行并发限制
 使用 dispatch_barrier_async/dispatch_barrier_sync 函数，确保两点：一是在执行此任务之前队列中其他任务已经完成，二是此任务完成之前队列中新增的任务不会执行，达到 barrier 的目标。

```objective-c
- (void)setObject:(id)obj atIndexedSubscript:(NSUInteger)idx {
    dispatch_barrier_async(_queue, ^{
        [_arr setObject:obj atIndexedSubscript:idx];
    });
}
```

③ 支持并发读取，使用 dispatch_sync 函数是将读取对象的操作加入到 queue 中，同步 dispatch 任务可以阻塞当前线程直到任务完成后成功获取到对象，而因为上述 barrier 机制的存在如果有写入操作则要等到写入操作完成后才能执行，单纯的读取操作可以在 queue 中并行，不会 barrier 队列。
 PS：使用 __block id o 修饰是为了在 block 内修改 block 外的局部变量。

```objective-c
- (id)objectAtIndexedSubscript:(NSUInteger)idx {
    __block id o;
    dispatch_sync(_queue, ^{
        o = [_arr objectAtIndexedSubscript:idx]
    });
    return o;
}
```

产生死锁的条件有四个：
　　l 互斥条件：所谓互斥就是进程在某一时间内独占资源。
　　l 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
　　l 不剥夺条件:进程已获得资源，在末使用完之前，不能强行剥夺。
　　l 循环等待条件:若干进程之间形成一种头尾相接的循环等待资源关系。
　　死锁通常是一个线程锁定了一个资源A，而又想去锁定资源B；在另一个线程中，锁定了资源B，而又想去锁定资源A以完成自身的操作，两个线程都想得到对方的资源，而不愿释放自己的资源，造成两个线程都在相互等待，造成了无法执行的情况。
　　避免死锁的一个通用的经验法则是:当几个线程都要访问共享资源A、B、C时，保证使每个线程都按照同样的顺序去访问它们，比如都先访问A，在访问B和C。
