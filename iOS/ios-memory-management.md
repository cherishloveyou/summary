## iOS内层管理

在iOS中内存分为五大区域：`栈区、堆区、全局区、常量区、代码区`



内存中的对象主要有两类，一类是值类型，比如int、float、struct等基本数据类型，另一类是引用类型，也就是继承自NSObject类的所有的OC对象。前一种值类型不需要我们管理，后一种引用类型是需要我们管理内存的，一旦管理不好，就会产生非常糟糕的后果。
**为什么值类型不需要管理，而引用类型需要管理呢？那是因为他们分配内存方式不一样。**
值类型会被放入栈中，他们依次紧密排列，在内存中占有一块连续的内存空间，遵循先进后出的原则。引用类型会被放到堆中，当给对象分配内存空间时，会随机的从内存当中开辟空间，对象与对象之间可能会留有不确定大小的空白空间，因此会产生很多内存碎片，需要我们管理。



- 任何继承了NSObject的对象
- 对其它非对象类型无效
- 简单来说：
  - 只有OC对象需要进行内存管理
  - 非OC对象类型比如基本数据类型不需要进行内存管理

Objective-C的对象在内存中是以堆的方式分配空间的,并且堆内存是由你释放的，就是release
OC对象存放于堆里面(堆内存要程序员手动回收)
非OC对象一般放在栈里面(栈内存会被系统自动回收)

### Objective-C 中的内存分配

在 Objective-C 中，对象通常是使用 `alloc` 方法在堆上创建的。 `[NSObject alloc]` 方法会在对堆上分配一块内存，按照`NSObject`的内部结构填充这块儿内存区域。

一旦对象创建完成，就不可能再移动它了。因为很可能有很多指针都指向这个对象，这些指针并没有被追踪。因此没有办法在移动对象的位置之后更新全部的这些指针。

### MRC 与 ARC

Objective-C中提供了两种内存管理机制：MRC（MannulReference Counting）和 ARC(Automatic Reference Counting)，分别提供对内存的手动和自动管理，来满足不同的需求。现在苹果推荐使用 ARC 来进行内存管理。

### MRC

#### 对象操作的四个类别

| 对象操作       |        OC中对应的方法        | 对应的 retainCount 变化 |
| -------------- | :--------------------------: | :---------------------: |
| 生成并持有对象 | alloc/new/copy/mutableCopy等 |           +1            |
| 持有对象       |            retain            |           +1            |
| 释放对象       |           release            |           -1            |
| 废弃对象       |           dealloc            |            -            |

#### 四个法则

- 自己生成的对象，自己持有。

- 非自己生成的对象，自己也能持有。

- 不在需要自己持有对象的时候，释放。

- 非自己持有的对象无需释放。

  

  如下是四个黄金法则对应的代码示例：

  ```objectivec
  //自己生成并持有该对象
  id obj0 = [[NSObeject alloc] init];
  id obj1 = [NSObeject new];
  
  //持有非自己生成的对象
  id obj = [NSArray array]; // 非自己生成的对象，且该对象存在，但自己不持有
  [obj retain]; // 自己持有对象
  
  //不在需要自己持有的对象的时候，释放
  id obj = [[NSObeject alloc] init]; // 此时持有对象
  [obj release]; // 释放对象
  
  //指向对象的指针仍就被保留在obj这个变量中
  //但对象已经释放，不可访问
  //非自己持有的对象无法释放
  id obj = [NSArray array]; // 非自己生成的对象，且该对象存在，但自己不持有
  [obj release]; // ~~~此时将运行时crash 或编译器报error~~~ 非 ARC 下，调用该方法会导致编译器报 issues。此操作的行为是未定义的，可能会导致运行时 crash 或者其它未知行为
  ```

  其中 `非自己生成的对象，且该对象存在，但自己不持有` 这个特性是使用`autorelease`来实现的，示例代码如下：

  ```objectivec
  - (id)getAObjNotRetain {
      id obj = [[NSObject alloc] init]; // 自己持有对象
      [obj autorelease]; // 取得的对象存在，但自己不持有该对象
      return obj;
  }
  ```

  `autorelease` 使得对象在超出生命周期后能正确的被释放(通过调用release方法)。在调用 `release` 后，对象会被立即释放，而调用 `autorelease` 后，对象不会被立即释放，而是注册到 `autoreleasepool` 中，经过一段时间后 `pool`结束，此时调用release方法，对象被释放。

  在MRC的内存管理模式下，与对变量的管理相关的方法有：retain, release 和 autorelease。retain 和 release 方法操作的是引用记数，当引用记数为零时，便自动释放内存。并且可以用 NSAutoreleasePool 对象，对加入自动释放池（autorelease 调用）的变量进行管理，当 drain 时回收内存。

### ARC

ARC 是苹果引入的一种自动内存管理机制，会根据引用计数自动监视对象的生存周期，实现方式是在编译时期自动在已有代码中插入合适的内存管理代码以及在 Runtime 做一些优化。

#### 变量标识符

在ARC中与内存管理有关的变量标识符，有下面几种：

- `__strong`
- `__weak`
- `__unsafe_unretained`
- `__autoreleasing`

`__strong` 是默认使用的标识符。只有还有一个强指针指向某个对象，这个对象就会一直存活。

`__weak` 声明这个引用不会保持被引用对象的存活，如果对象没有强引用了，弱引用会被置为 nil

`__unsafe_unretained` 声明这个引用不会保持被引用对象的存活，如果对象没有强引用了，它不会被置为 nil。如果它引用的对象被回收掉了，该指针就变成了野指针。

`__autoreleasing` 用于标示使用引用传值的参数（id *），在函数返回时会被自动释放掉。

变量标识符的用法如下：

### __strong

`__strong` 表示强引用，对应定义 `property` 时用到的 `strong`。当对象没有任何一个强引用指向它时，它才会被释放。如果在声明引用时不加修饰符，那么引用将默认是强引用。当需要释放强引用指向的对象时，需要保证所有指向对象强引用置为 nil。`__strong` 修饰符是 id 类型和对象类型默认的所有权修饰符。



# 内存管理方案技术

- `Tagged Pointer`：(标记指针)，用来处理小对象NSNumber，NSDate、NSString

- `Nonpointer_isa`：非指针类型。简单来说就是64位的二进制数据，不是存的指针，其他位用来存储OC对象一些其他信息

- `sideTables`：当引用计数存储到一定值时，并不会再存储到`Nonpointer_isa`的位域的`extra_rc`，而是会存储到`SideTables`散列表中。在散列表中主要有两个表，分别是`引用计数表、弱引用表`，`同一时间，真机中散列表最多只能有8张`

  

- 什么是tagged Pointer？

> 在64位的地址中，我们并没有真正所有的使用这些位，我们只是在一个真正的对象指针中使用了中间的一些位，由于对齐的要求地位总是0，对象必须总是位于指针大小倍数的一个地址中，由于地址空间有限，所以高位总是位0，我们实际上不会用到所有的64位，由于高位和低位总是位0，所以我们可以把这些位置设置位1，这样就不是一个真正的指针。我们可以把这些位赋予其他的一些含义。我们把这样的地址称为`tagged Pointer`

- Tagged Pointer专⻔⽤来存储⼩的对象，例如NSNumber和NSDate
- Tagged Pointer指针的值不再是地址了，⽽是真正的值。所以，实际上它不再是⼀个对象了，它只是⼀个披着对象⽪的普通变量⽽已。所以，它的内存并不存储在堆中，也不需要malloc和free
- 在内存读取上有着3倍的效率，创建时⽐以前快106倍。

### Tagged Pointer数据混淆

在dyld加载类的时候，_read_images->initializeTaggedPointerObfuscator

```objective-c
static void
initializeTaggedPointerObfuscator(void)
{
    if (!DisableTaggedPointerObfuscation) {
        // Pull random data into the variable, then shift away all non-payload bits.
        arc4random_buf(&objc_debug_taggedpointer_obfuscator,
                       sizeof(objc_debug_taggedpointer_obfuscator));
        objc_debug_taggedpointer_obfuscator &= ~_OBJC_TAG_MASK;

#if OBJC_SPLIT_TAGGED_POINTERS
        // The obfuscator doesn't apply to any of the extended tag mask or the no-obfuscation bit.
        objc_debug_taggedpointer_obfuscator &= ~(_OBJC_TAG_EXT_MASK | _OBJC_TAG_NO_OBFUSCATION_MASK);

        // Shuffle the first seven entries of the tag permutator.
        int max = 7;
        for (int i = max - 1; i >= 0; i--) {
            int target = arc4random_uniform(i + 1);
            swap(objc_debug_tag60_permutations[i],
                 objc_debug_tag60_permutations[target]);
        }
#endif
    } else {
        // Set the obfuscator to zero for apps linked against older SDKs,
        // in case they're relying on the tagged pointer representation.
        objc_debug_taggedpointer_obfuscator = 0;
    }
}
```

我们可以看到每次程序启动的时候都会生成一个随机数`objc_debug_taggedpointer_obfuscator`，让后通过这个数进行编码和解码

### 编码_objc_encodeTaggedPointer

```objective-c
static inline void * _Nonnull
_objc_encodeTaggedPointer(uintptr_t ptr)
{
    uintptr_t value = (objc_debug_taggedpointer_obfuscator ^ ptr);
#if OBJC_SPLIT_TAGGED_POINTERS
    if ((value & _OBJC_TAG_NO_OBFUSCATION_MASK) == _OBJC_TAG_NO_OBFUSCATION_MASK)
        return (void *)ptr;
    uintptr_t basicTag = (value >> _OBJC_TAG_INDEX_SHIFT) & _OBJC_TAG_INDEX_MASK;
    uintptr_t permutedTag = _objc_basicTagToObfuscatedTag(basicTag);
    value &= ~(_OBJC_TAG_INDEX_MASK << _OBJC_TAG_INDEX_SHIFT);
    value |= permutedTag << _OBJC_TAG_INDEX_SHIFT;
#endif
    return (void *)value;
}
```

编码实际上做了异或的处理，在真机上做了一些不一样的处理。我们已x86_64的为例。

### 解码_objc_decodeTaggedPointer

```objective-c
static inline uintptr_t
_objc_decodeTaggedPointer_noPermute(const void * _Nullable ptr)
{
    uintptr_t value = (uintptr_t)ptr;
#if OBJC_SPLIT_TAGGED_POINTERS
    if ((value & _OBJC_TAG_NO_OBFUSCATION_MASK) == _OBJC_TAG_NO_OBFUSCATION_MASK)
        return value;
#endif
    return value ^ objc_debug_taggedpointer_obfuscator;
}
```

编码解码举例，假如`objc_debug_taggedpointer_obfuscator = 1010 1010` 原始的`ptr= 0101 1110`，

编码： `0101 111` ^ `1010 1010` = `0000 1010`

解码： `1010 1010` ^ `1010 1010` =  `0101 1110`



## 环境变量设置

我们可以通过设置xCode环境变量设置，`OBJC_DISABLE_TAG_OBFUSCATION = YES` 关闭混淆

### Tagged Pointer 数据结构

在`objc`源码找到下面的代码

```objective-c
static inline bool _objc_isTaggedPointer(const void * _Nullable ptr)
{
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}
```

通过是否是`tagged poniter`方法，_OBJC_TAG_MASK这个字段。在mac os下 ![截屏2021-09-20 下午2.40.15.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a41a85a666e3444bb462a7d1bf61f205~tplv-k3u1fbpfcp-watermark.awebp?) `_OBJC_TAG_MASK = 1`，也就是最低位为1，在iOS下`define _OBJC_TAG_MASK (1UL<<63)`，经过调试发现在不同的真机版本下面，tagged pointer结构会有所不同，下面的结果是我在iOS，14.5下面lldb调试，以二进制打印地址

![截屏2021-09-20 下午3.18.07.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e77dcc0a04145719907011b44fe7f95~tplv-k3u1fbpfcp-watermark.awebp?) lldb调试，以二进制打印地址 ([p6-juejin.byteimg.com/tos-cn-i-](https://link.juejin.cn?target=https%3A%2F%2Fp6-juejin.byteimg.com%2Ftos-cn-i-)

- 0b表示二进制，最高位位1表示是tagged pointer类型。`0-1`表示类型:`010 = 2`表示`NSString`，`011= 3`,表示`NSNumber`,在objc源码里可以验证这一点

![截屏2021-09-20 下午3.01.48.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/df75af2372664baa981a734d64a4b041~tplv-k3u1fbpfcp-watermark.awebp?)

- `NSString`：`3-6`位，表示字符串的长度。比如0010 = 2，字符串长度为2
- `NSNumber`：`3-6`位，表示表示存储的是int 、float、double类型

### 数据结构总结

用一张图总结真机`14.5`下的存储结构。

![截屏2021-09-20 下午3.42.54.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c3eade0bc7a4573ac34cd31c9fd7b93~tplv-k3u1fbpfcp-watermark.awebp?)

当然这个图只是我在14.5下面的存储，不同的版本位置可能不一样。有兴趣的可以去研究一下。

### NSString 内存管理

那么是不是`NSString`一定就是`Tagged Pointer`类型呢。下面我们做一个测试。

- 通过initWithString，stringWithString 和stringWithFormat，initWithFormat分别初始化string
- 初始化的字符串的长度不同，`<9的情况`，`和>9的情况`

代码如下：

![NString.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f8906fc44254dc9aef803ad9aecf681~tplv-k3u1fbpfcp-watermark.awebp?)

打印结果：

![打印结果.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4b3dd4e79cda4ee19d4da83c6ea37290~tplv-k3u1fbpfcp-watermark.awebp?) 从上面的打印结果发现NSString的类型有3种。

- `__NSCFConstantString`：字符串常量，是一种`编译时常量`，retainCount值很大，对其操作，`不会引起引用计数变化，存储在字符串常量区` 。直接通过@""、initWithString、StringWithString 初始化

-` NSTaggedPointerString`：就是我们上面提到的`tagged pointer`类型。这个是苹果的优化，但是字符串的长度不能太长。具体是NSString类型，还是Tagged Pointer类型可以参考下面有一张表。

![截屏2021-09-20 下午4.25.02.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b88f81c1f96345278e89787ceadd676f~tplv-k3u1fbpfcp-watermark.awebp?)

- `__NSCFString`：这个是对象类型，会分配内存空间、增加引用计数，存储在堆上

### 总结

1. `tagged pointer`实际上是苹果系统堆内存的优化，可用于（NSString 、NSNumber、NSDate）。它的数据直接存储在指针中
2. 由于`tagged pointer`不是存储在堆上，它的存取更方便，效率更高
3. 减少了不必要的浪费、节省内存空间。



# nonpointer_isa(非指针类型)

isa分为`pointer_isa`(指针类型)和非指针类型(`Nonpointer_isa`),简单的理解就是，如果`isa`是指针类型，那么就是一个纯的地址，没有做其他处理。如果是一个非指针类型，那么`isa`就是64位的地址，不止包含地址，还有其他的一些字段。





# retain原理

进入源码`objc_retain -> retain -> rootRetain`

- 【第一步】判断是否是`Nonpointer_isa`
- 【第二步】操作引用计数
  - 如果不是`Nonpointer_isa`，则直接操作`SideTables`散列表
  - 判断`是否正在释放`，如果正在释放就执行dealloc
  - 执行`ectra_rc+1`，引用计数+1，并给一个引用计数的`状态标识carry`来判断`ectra_ra`是否满了
  - 如果`carry`标记状态表示`ectra_rc引用计数满了`，此时开始操作`散列表`，即将`ectra_rc`存储的一半拿出存到`散列表`，

# release原理

通过`setProperty -> reallySetProperty -> objc_release -> release -> rootRelease -> rootRelease`进入rootRelease源码，其中与retain相反

- 判断是否是`Nonpointer isa`，如果不是，操作`散列表-1`

- 如果是，对`extra_rc`中的引用计数-1，并将此时的extra_rc状态存储到`carry`

- 如果`carry == 0`，执行`underflow`

- 执行

  ```
  underflow
  ```

  - 判断散列表是否存储了一半引用计数
  - 如果是，从散列表中取出一半引用计数，进行-1，然后存储到extra_rc中
  - 如果不是，直接`dealloc`



# dealloc 原理

进入源码`dealloc -> _objc_rootDealloc -> rootDealloc`

- 判断`是否是小对象`，如果是直接返回

- 判断

  ```
  是否有isa. nonpointer、cxx、关联对象、弱引用表、引用计数表
  ```

  - 如果没有，直接free是否内存
  - 如果有，执行`object_dispose`





[iOS 内存管理](https://www.jianshu.com/p/fc487c114402)

[iOS内存管理-Tagged Pointer技术](https://juejin.cn/post/7009932334505918495)
