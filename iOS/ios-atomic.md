# iOS atomic

IOS项目中nonatomic和atomic分析
//有两个属性，分别设置为nonatomic和atomic
一、 10000个异步任务，修改name属性的值

```objective-c
@interface ViewController : UIViewController
@property (nonatomic, strong) NSString *name;
@property (atomic, assign) int number;
@end
  
- (void)nonatomic {
  for (NSInteger i = 0; i < 10000; i++) {
      dispatch_async(dispatch_get_global_queue(0, 0), ^{
          self.name = [NSString stringWithFormat:@"name:%ld", i];
      });
  }
}
```

- 执行结果：崩溃，崩溃原因是在子线程Thread8上，对象释放了。

结果分析：
1、在MRC模式下，属性name的set方法如下

```objective-c
- (void)setName:(NSString *)name{
    if (_name != name) {
        [_name release];
        [name retain];
        _name = name;
    }
}
```


2、虽然在ARC模式下不用写其set方法，但是在底层还是会走到这里
3、因为是多线程，且没有加锁保护，所以在一个线程走到[_name release]后，可能在另一个线程又一次去释放，这时候造成崩溃。
4、把name属性的nonatomic改成atomic就不会崩溃了，因为atomic加锁了，是安全的。

二、接着上步说用atomic就安全了，再进一步分析
number属性使用atomic修饰的

```objective-c
- (void)atomic {
    self.number = 0;
    dispatch_apply(10000, dispatch_get_global_queue(0, 0), ^(size_t index) {
        self.number ++;
    });
    NSLog(@"_number:%d", self.number);
}
```

执行结果：执行结果并不是10000，而且每次运行结果都不一样，即运行结果不可预见。

结果分析：

_number++等价于
 int temp = _number+1;
 _number = temp;

虽然atomic保证了number属性线程安全了，但是并不能保证temp变量的线程安全，又因为是多线程的，所以有可能同时执行多次 int temp = _number+1;才执行一次 _number = temp;导致结果每次都不同，而且结果不可预知。

#### atomic VS nonatomic

> atomic是原子性的，nonatomic是非原子性的。
> atomic原子性并不能保证多线程安全，只是能保证数据的完整性
> 这个完整性体现在：使用者总能取到完整的值

但是如果一个线程在多次修改某个属性时，另一个线程去读取属性时，可能会取到未修改好的属性，下面我们将举例来证明：

源码：

```objective-c
@interface ViewController () 
@property (nonatomic, strong) NSArray *dataArray;
@end

@implementation ViewController

- (void)viewDidLoad {
  [super viewDidLoad];

  self.dataArray = @[@"1",@"2",@"3",@"4",@"5"];
  dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
      for (int i=0; i<100; i++) {
          self.dataArray = @[@"1",@"2"];
      }
      NSLog(@"--1111111=--%@",self.dataArray);
  });

  for (int i=0; i<10; i++) {
      dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
          NSLog(@"--222222=--%@",self.dataArray);
      });
  }
}
```

使用nonatomic修饰dataArray，会崩溃，但是并不是每次都崩，差不多10次左右就会崩一次。
我们用nonatomic，strong修饰的数组dataArray，在一个异步队列的任务中持续地去修改这个属性，在另一个异步队列的任务中持续地去读取这个属性，**由于dataArray是由strong来修饰的，那么在dataArray的setter方法中，其实是先保留新值，后释放旧值，再将指针指向新值，所以在后面这个异步队列的任务中，取到的dataArray可能是一个已经被释放的僵尸对象，所以会崩溃。**

当线程A进行写操作，这时其他线程的读或者写操作会因为该操作而等待。当A线程的写操作结束后，B线程进行写操作，然后当A线程需要读操作时，却获得了在B线程中的值，这就破坏了线程安全，如果有线程C在A线程读操作前release了该属性，那么还会导致程序崩溃。所以仅仅**使用atomic并不会使得线程安全，我们还要为线程添加lock来确保线程的安全。**

使用atomic修饰dataArray，不会崩溃在这段代码中。



那么还有一个问题：atomic是线程安全的吗？
线程安全：多线程操作共享数据不会出现想不到的结果就是线程安全的，否则，是线程不安全的。
下面让我们来做个小实验
对用atomic修饰的数组连续修改时，读取属性，输出结果为：

我们对在一个异步队列的任务中，对dataArray多次写入，并在另一个异步队列中多次读取dataArray，从输出结果中发现，得到的结果并不一样，所以使用atomic也并不能保证线程安全。

atomic 原理和作用
设置成员变量的@property属性时，默认为atomic，提供多线程安全。
在多线程环境下，原子操作是必要的，否则有可能引起错误的结果。加了atomic，setter函数会变成下面这样：

```objc
{lock}
    if (property != newValue) { 
        [property release]; 
         property = [newValue retain]; 
    }
{unlock}

```

**Atomic不能保证对象多线程的安全，它只是能保证你访问的时候给你返回一个完好无损的Value而已。atomic：系统生成的 getter/setter 会保证 get、set 操作的完整性，不受其他线程影响。getter 还是能得到一个完好无损的对象（可以保证数据的完整性），但这个对象在多线程的情况下是不能确定的，比如上面的例子。**

也就是说：如果有多个线程同时调用setter的话，不会出现某一个线程执行完setter全部语句之前，另一个线程开始执行setter情况，相当于函数头尾加了锁一样，每次只能有一个线程调用对象的setter方法，所以可以保证数据的完整性。

atomic所说的线程安全只是保证了getter和setter存取方法的线程安全，并不能保证整个对象是线程安全的。仅仅使用atomic并不会使得对象线程安全，我们还要为对象线程添加lock来确保线程的安全。

```objective-c
//nonatomic对象setter和getter方法的实现:
- (void)setCurrentImage:(UIImage *)currentImage {
  if (_currentImage != currentImage) {
      [_currentImage release];
      _currentImage = [currentImage retain];
  }
}

- (UIImage *)currentImage {
  return _currentImage;
}

//atomic对象setter和getter方法的实现:
- (void)setCurrentImage:(UIImage *)currentImage {
  @synchronized(self) {
      if (_currentImage != currentImage) {
          [_currentImage release];
          _currentImage = [currentImage retain];
      }
  }
}

- (UIImage *)currentImage {
  @synchronized(self) {
      return _currentImage;
  }
}
```


简而言之就是只保证setter和getter的操作完整性，不保证属性的线程安全，atomic修饰后，还是可以多线程同时修改这个值的，至于这个值最终是什么，天知道。
atomic修饰后， 不会出现多线程同时修改这个值的。至于这个值最终是什么，无法确定，是因为你不知道多线程的调用顺序，也就无法判断最终的值是什么。

 也就是要注意：atomic所说的线程安全只是保证了getter和setter存取方法的线程安全，并不能保证整个对象是线程安全的。如下列所示：比如：@property (atomic, strong) NSMutableArray *arr;  

如果一个线程循环的读数据，一个线程循环写数据，那么肯定会产生内存问题，因为这和setter、getter没有关系。如使用[self.arr objectAtIndex:index]就不是线程安全的。好的解决方案就是加锁。

atomic在set方法里加了锁，防止了多线程一直去写这个property，造成难以预计的数值。但这也只是读写的锁定。跟线程安全其实还是差一些。看下面。

```objective-c
@interface MONPerson : NSObject 
@property (copy) NSString * firstName; 
@property (copy) NSString * lastName; 

- (NSString *)fullName; 
  @end

Thread A:
p.firstName = @"Rob";
Thread B:
p.firstName = @"Robert";
Thread A:
label.string = p.firstName; // << uh, oh -- will be Robert

但是如果有个C也在写，D在读取，D会读到一些随机的值（ABC修改的值），这就不是线程安全的了。最好的方法是使用lock。

Thread A:
[p lock]; // << wait for it… … … …
// Thread B now cannot access 
pp.firstName = @"Rob";
NSString fullName = p.fullName;
[p unlock];
// Thread B can now access plabel.string = fullName;

Thread B:
[p lock]; // << wait for it… … … …
// Thread A now cannot access p…
[p unlock];
```

结论：**用atomic修饰后，这个属性的setter、getter方法是线程安全的，但是对于整个对象来说不一定是线程安全的。**



https://blog.csdn.net/shifang07/article/details/100576587
