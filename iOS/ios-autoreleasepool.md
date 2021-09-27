# AutoreleasePool

AutoreleasePool是OC中的一种内存管理机制，它持有释放池里的对象的所有权，在自动释放池销毁时，统一给所有对象发送一次`release`消息。通过这个机制，可以**延迟对象的释放**。

在OC中 自生成并持有对象的方式只有 *alloc/new/copy/mutableCopy* 四种 ，其他方式均为非自生成并持有对象 

```objectivec
{
    //自生成并持有对象
    NSMutableArray *array1 = [[NSMutableArray alloc]init];
    //非自生成并持有对象, 因为array2 持有的是通过 +()array 方法返回的对象
    NSMutableArray *array2 = [NSMutableArray array];
}
```

接着autoreleasePool说，正常情况下，在超出作用域时对象会被自动释放掉，如下代码

```objectivec
{
    NSObject *obj = [[NSObject alloc]init];
    //自己生成并持有对象，引用计数 + 1 （retain）
    //retainCount = 1
}
//超出作用域 obj 引用计数 -1 (release)
//此时retainCount = 0 所以 obj 销毁
```

retain 和 release 成对出现，不需要另加一个自动释放池来进行管理，一样能完成内存的自动回收。这不就扯了吗？本来就好好的东西，干嘛非得加个自动释放池呢？看下面的一个例子

```objectivec
- (NSObject *)getObj {
    NSObject *obj = [[NSObject alloc]init];
    //自己生成并持有对象，引用计数 + 1 （retain）
    //retainCount = 1
    return obj;
    //return 导致提前出作用域 引用计数 -1 (release)
    //retainCount = 0 obj 被释放
}
```

当我们需要调用这个函数赋值时 如下

```objectivec
{
    NSObject *obj_1 = [self getObj];
}
```

问题来了，正如刚才所说，正常情况下 出作用域时对象会被自动释放掉，于是就造成了 obj_1 在想取得持有对象时 发现对象被释放掉了，这显然是不合理的。这就像是你满心欢喜在天猫买了个冰棒，拿到快递时发现冰棒竟然化没了，你说闹心不闹心。

虽然道理是这个道理，但在实际工作时并没有这种情况发生，这是怎么回事呢？这其实就是autoreleasePool的功劳了，编译器会在return 之前提前把对象retain 并 注册到自动释放池 大体过程类似下面的代码（这里只是用于演示过程）

```objectivec
- (NSObject *)getObj {
    NSObject *obj = [[NSObject alloc]init];
    //自己生成并持有对象，引用计数 + 1 （retain）
    //retainCount = 1
    
    //下面这一步由编译器自动完成
    NSObject *autoreleaseObj = obj;
    [autoreleaseObj retain];
    //obj 引用计数 + 1
    //retainCount = 2
    
    [autoreleaseObj autorelease];
    //把autoreleaseObj注册到自动释放池
    
    return autoreleaseObj;
    //return 导致提前出作用域 引用计数 -1 (release)
    //retainCount = 1 obj不会被释放
    //此时obj 被autoreleasePool持有
}
```

那这个pool在哪里呢？对于oc来说，整个程序都是运行在一个pool中的，可以看一下main函数的实现

```objectivec
int main(int argc, char * argv[]) {
    //自动释放池
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

此时由于return后对象被autoreleasePool持有，因此不会被提前释放掉，obj_1 自然也就可以拿到并持有该对象引用计数+1。这样，当出作用域时obj_1被释放，引用计数-1，当autoreleasePool退出时 对象引用计数 -1 至此被注册到autoreleasePool的对象的引用计数 = 0 被释放掉。由此完成了内存的自动管理。

问题是解决了，但会造成内存消耗增加，为什么会增加内存消耗？由于对象会被注册到自动释放池，而且在自动释放池结束前对象一直被持有，因此当大量的对象被注册到自动释放池时就会造成内存激增，看下面一段代码

```objectivec
- (void)viewDidLoad {
    [super viewDidLoad];

    for (int i = 0; i < 10000; i ++) {
        for (int j = 0; j < 100; j ++) {
            NSMutableArray *array = [self getArray];
            NSLog(@"%@",array);
        }
    }
 }

- (NSMutableArray *)getArray {
    NSMutableArray *arr = [[NSMutableArray alloc]init];
    [arr addObject:@"hh"];
    return arr;
}
```

事实上，只要是调用非自己生成并持有的对象，该对象就会被注册到自动释放池，比如下面的方式

```objectivec
NSMutableArray *arr = [NSMutableArray arrayWithObjects:@"hh", nil];
//由于调用的是非自己生成并持有对象，所以会被注册到自动释放池，因此在循环调用时一样会造成内存激增。需要注意
```

接着内存暴增说，既然自动释放池会造成内存暴增，那肯定要找方法来解决，我们从自动释放池的释放过程入手来解决这个问题，首先说一下自动释放池的嵌套。

关于嵌套，这里补充一点，当多层自动释放池嵌套时，内层自动释放池会屏蔽掉外层自动释放池对内层自动释放池中的对象retain，很绕，说的直白一点，就是内层自动释放池中的对象只会注册到内层自动释放池中。如下

```objectivec
//外层池子
@autoreleasepool {
    //内层池子
    @autoreleasepool {
        NSMutableString *poolStr = [NSMutableString stringWithString:@"hh"];
        //非自己生成并持有对象，poolStr被注册到内层池子
    }
    //内层池子结束，poolStr释放
}
```

结合上面自动释放池内存激增的原理，既然内层池子释放时，注册到内层池子的对象也会被释放，因此对于内存激增的问题，我们可以采用自动释放池的嵌套来解决，对于上面的代码我们稍加修改，如下

```objectivec
 - (void)viewDidLoad {
    [super viewDidLoad];

    for (int i = 0; i < 10000; i ++) {
        //内层池子
        @autoreleasepool {
            for (int j = 0; j < 100; j ++) {
                NSMutableArray *array = [self getArray];
                NSLog(@"%@",array);
            }
        }
    }
 }

- (NSMutableArray *)getArray {
    NSMutableArray *arr = [[NSMutableArray alloc]init];
    [arr addObject:@"hh"];
    return arr;
}
```

此时，由于内层池子中的一百次循环完毕后，内层池子便会结束，因此注册到内层池子的对象便会被即时释放，因此内存不会继续增加。

额外补充：其实当开辟一条新的线程时，同时也会创建一个pool用于新线程的内存管理，但是POSIX由于不在ARC的内存管理范围之内，因此通过pthread创建的新线程需要自己创建autoreleasePool，这一点可以通过打印autoreleasePool得知（打印方法，自行百度），由于class cluster 和 tagged Pointer优化 会造成部分对象的表现异常，比如NSString 和 NSNumber在内容较少时并不会生成真正的对象，因此也不会被加入到自动释放池.



#### AutoreleasePool的创建方式

通常使用`@autoreleasepool {}`代码块来手动创建一个自动释放池

```c
@autoreleasepool {
    //这里创建自动释放的对象，创建的对象会被加入到AutoreleasePool对象里
    ... ...
}
```

这个代码块等价于

```c
{
    //创建一个AutoreleasePool对象
    __AtAutoreleasePool *atautoreleasepoolobj = objc_autoreleasePoolPush(); 
    
    //这里创建自动释放的对象，创建的对象会被加入到AutoreleasePool对象里
    ... ...    
   //给所有自动释放的对象发送一次release消息，并销毁AutoreleasePool对象
   objc_autoreleasePoolPop(atautoreleasepoolobj)
}

`{}`表示AutoreleasePool对象的作用域
```

代码块的实现逻辑如下：

- 先通过调用`objc_autoreleasePoolPush`函数来创建一个`AutoreleasePool`对象。
- 然后给在代码块里创建的每个自动释放的对象发送一个`autorelease`消息，将这些自动释放的对象加入到`AutoreleasePool`对象里。
- 最后在`AutoreleasePool`对象将要销毁时，通过调用`objc_autoreleasePoolPop`函数给池中每个自动释放的对象发送一次`release`消息，再销毁`AutoreleasePool`对象。

> 注意区分`AutoreleasePool对象`和`自动释放的对象`，`AutoreleasePool对象`指的是实例化的一个自动释放池（本质也是对象），而 `自动释放的对象`是指被加入到这个池中的对象。
>  `AutoreleasePool`的原理可阅读后面的底层分析一文。

#### AutoreleasePool在Runloop中的创建和销毁

通常情况下，在平时开发中不需要手动创建自动释放池，因为`Runloop`会自动创建和销毁`AutoreleasePool`对象。

![img](https:////upload-images.jianshu.io/upload_images/2912639-3d344d82169f267a.png?imageMogr2/auto-orient/strip|imageView2/2/w/562/format/webp)



如上图所示，`AutoreleasePool`在`Runloop`中的创建和销毁的过程如下:

- App启动后，系统在主线程`RunLoop`里注册了两个`Observer`，其回调都是`_wrapRunLoopWithAutoreleasePoolHandler()`。
- 第一个`Observer`监视一个事件：
  - `Entry（即将进入Loop）`：调用`objc_autoreleasePoolPush`来创建自动释放池。
- 第二个`Observer`监视了两个事件：
  - `Before waiting（准备进入休眠）`：先调用`objc_autoreleasePoolPop`销毁旧的自动释放池，再调用`objc_autoreleasePoolPush`创建一个新的自动释放池。
  - `Exit（即将退出Loop）`：调用`objc_autoreleasePoolPop`销毁自动释放池。
- 第一个`observe`的`order`是`-2147483647`，优先级最高，保证创建释放池发生在其他所有回调之前。
   第二个`Observer`的`order`是`2147483647`，优先级最低，保证销毁自动释放池发生在其他所有回调之后。

也就是说，在一个`RunLoop`事件开始的时候会自动创建一个`AutoreleasePool`，在事件结束时再自动销毁。上面举例的`imageNamed`方法内部创建的对象也是加入到主线程`RunLoop`创建的`AutoreleasePool`中实现延迟释放的。因此，通常在开发中不需要开发者自己创建`AutoreleasePool`。

#### 手动创建AutoreleasePool的场景

虽然`Runloop`会自动创建和销毁自动释放池，但在有些情况下还是需要手动创建`AutoreleasePool`。[苹果官方文档](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.apple.com%2Flibrary%2Farchive%2Fdocumentation%2FCocoa%2FConceptual%2FMemoryMgmt%2FArticles%2FmmAutoreleasePools.html)建议在下面这三种情况下可能需要开发者创建自动释放池：

- **编写不基于UI框架的程序，例如命令行工具。**

  这一点的原因不是特别清楚，猜测是不基于UI框架的程序，可能不响应用户事件，导致不自动创建和销毁自动释放池。

- **编写一个创建大量临时对象的循环。**

  在循环内使用自动释放池块可以在下一次迭代之前释放这些对象，有助于减少应用程序的最大内存占用，即`降低内存峰值`。

- **编写非Cocoa程序时创建子线程。**

  Cocoa程序中的每个线程都维护自己的自动释放池块堆栈。而编写一个非Cocoa程序，比如`Foundation-only program`，这时如果创建了子线程，若不手动创建自动释放池，自动释放的对象将会堆积得不到释放，导致内存泄漏。

这里就第二个场景举例，来说明在循环内使用`AutoreleasePool`对于降低内存峰值的作用。

```objectivec
//情况一：循环内不使用AutoreleasePool
for (int i = 0; i<1000000; i++) {

    NSString *string = [NSString stringWithFormat:@"%@", @"0123456789"];
    NSLog(@" ==== %p", string);
}

//情况二：循环内使用AutoreleasePool
for (int i = 0; i<1000000; i++) {
    @autoreleasepool {
        NSString *string = [NSString stringWithFormat:@"%@", @"0123456789"];
        NSLog(@" ==== %p", string);
    }
}
```

分别运行上面两种情况可以看到，在循环过程中，第一种情况的内存占用一直在增加，第二种情况的内存不会增加。这是因为：

- `情况一`：循环过程中，创建的`NSString`对象一直在堆积，只有在循环结束才一起释放，所以内存一直在增加。
- `情况二`：每一次迭代中都会创建并销毁一个`AutoreleasePool`，而每一次创建的`NSString`对象都会加入到`AutoreleasePool`中，所以在每次`AutoreleasePool`销毁时，`NSString`对象就会被释放，这样内存就不会增加。

这个场景中`AutoreleasePool`是通过立即释放对象来降低内存峰值，而前面又说自动释放池用来延迟对象的释放，这两者其实不矛盾，本质是一样的，都是在自动释放池销毁时调用`objc_autoreleasePoolPop`来释放池中的对象。只不过调用的时机不同，这里的`@autoreleasepool {}`是在超出自己的作用域时就调用函数来销毁，而前面的是在`Runloop`休眠或退出时才调用函数来销毁，所以调用的时机不同，才会实现`立即或者延迟`释放的目的。

> `@autoreleasepool {}`的作用域指的就是前面提到的`{}`，是`AutoreleasePool对象`的作用域。

#### 哪些对象可以被添加到自动释放池？

在`MRC`模式下，只要给对象发送`autorelease`消息，这个对象就会被添加到自动释放池。但在`ARC`模式下，是由编译器自动给对象发送`autorelease`消息，且不会给所有的对象都发送，只会给被编译器识别为`自动释放的对象`发送。一般来说，**使用类方法（工厂方法）实例化的对象才是自动释放的对象，才能被添加到自动释放池**，而使用`new、alloc、copy`关键字生成的对象和`retain`了的对象，不会被添加到自动释放池中。

- 以`UIImage`对象为例

```objectivec
for (int i = 0; i<1000000; i++) {
    @autoreleasepool {
        //1.自动释放的对象，需要被添加到自动释放池中
        UIImage *image = [UIImage imageNamed:@"test.png"];
        //2.非自动释放的对象，不能被添加到自动释放池中
        UIImage *image = [[UIImage alloc] init];
        NSLog(@" ==== image = %p", image);
    }
}
```

分别运行上面两种情况，第一种情况内存不会增加，第二种情况内存会增加。第二种情况虽然在@autoreleasepool {}中创建对象，但由于不是自动释放的对象，所以还是不能被添加到AutoReleasePool中，只能在循环结束一起释放。因此，在ARC模式下，只有自动释放的对象才能被添加到AutoReleasePool中，非自动释放的对象在超出作用域时会被立即释放。

> 需要注意的是，自动释放的对象如果没有被添加到AutoReleasePool中，就会产生内存泄露。

#### 总结

总得来说，关于AutoreleasePool的基本概念可以归纳以下几点：

- AutoreleasePool（自动释放池）持有释放池里的对象的所有权，在自动释放池销毁时，统一给所有对象发送一次release消息。
- 在一个RunLoop事件开始的时候会自动创建一个AutoreleasePool，在事件结束时再自动销毁，这样可以延迟对象的release。
- 也可以使用@autoreleasepool {}来手动创建自动释放池，在循环中使用可以立即释放对象，降低内存峰值。
- 在MRC模式下，只要给对象发送autorelease消息，这个对象就会被添加到自动释放池。
- 在ARC模式下，一般来说，使用类方法（工厂方法）实例化的对象才是自动释放的对象，才能被添加到自动释放池。自动释放的对象如果没有被添加到AutoReleasePool中，就会产生内存泄露。
- 在ARC中，自动释放的对象由编译器自主识别并发送autorelease消息，添加到AutoReleasePool中。



[AutoreleasePool的原理和实现](https://www.jianshu.com/p/1b66c4d47cd7)

[Autoreleasepool实现原理](https://www.jianshu.com/p/2ddd9040b272)

[Autorelease Pool的实现原理总结](https://blog.csdn.net/lizitao/article/details/56485100)

