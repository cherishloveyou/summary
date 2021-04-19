#### 题目：

```objectivec
//MNPerson
@interface MNPerson : NSObject
@property (nonatomic, copy)NSString *name;
- (void)print;
@end

@implementation MNPerson
- (void)print{
    NSLog(@"self.name = %@",self.name);
}
@end
---------------------------------------------------
@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    id cls = [MNPerson class];
    void *obj = &cls;
    [(__bridge id)obj print];
}
```

最终结果：

```objective-c
self.name = <ViewController: 0x7fe667608ae0>
```

**问题1：print 是实例方法，但是并没有哪里调用了 `[MNPerson alloc]init]` ?**

当前内存地址结构 - 与正常的`[person print]` 对。

![](https://raw.githubusercontent.com/cherishloveyou/summary/main/iOS/Assets/Question1/18569867-98cb9271115aeb5c.png)

- person变量的指针，执行 MNPerson 实例对象
- 实例对象的本身是个结构体，之前指向他，等价于执行结构体的第一个成员
- 结构体的第一个成员是isa，所以可以理解为，person->isa
- 所以两个print，其实内存结构一致
  - obj -> cls -> [MNPerson Class]
  - person -> isa -> [MNPerson Class]

> 调用print 方法，不需要关心有没有成员变量 `_name`，所以可以理解为，cls == isa

- 函数调用，是通过查找isa，其实本质，是查找结构体的前八个字节;
- 前八个字节正好是isa，所以这里可以理解为 cls == isa，这么理解的话，cls其实等于isa;
- 所以可以找得到 MNPerson 类，就可以找到MNPerson 类内部的方法，从而调用 `print` 函数

**问题2：为啥里面打印的是 `ViewController`**

这就需要了解到iOS的内存分配相关知识

因为`obj`，`cls`都是`viewDidLoad`方法（函数）里面的局部变量，我们知道函数的局部变量都是放在栈空间里面的。那么你了解函数的栈空间吗？我们来简单科普一下。

#### 函数的栈空间简介

栈空间的作用，是用来存放被调用函数其内部所定义的局部变量的。对于`arm64`架构来说，这么理解就够了，如果你恰好了解过`8086`汇编，那么可能知道，栈空间里面还会存放函数的参数，但是对于`arm64`来说，函数的参数通常会放到寄存器里面，所以我们就先简单的认为，函数的栈空间里面放的就是函数的局部变量。而且局部变量的存放顺序，是根据定义的先后顺序，从函数栈底开始，一个一个排列，最先定义的局部变量位于栈底（高地址），通过下图来描绘一下。

```cpp
void test(){
    int a = 10;
    int b = 20;
    int c = 30;
    NSLog(@"a = %p,b = %p,c = %p",&a,&b,&c);
}
---------------------------
a = 0x7ffee87e9fdc,
b = 0x7ffee87e9fd8,
c = 0x7ffee87e9fd4
```

![](https://raw.githubusercontent.com/cherishloveyou/summary/main/iOS/Assets/Question1/18569867-f54ebmxa7f1e36a2.png)

- 局部变量是在栈空间
- 上图可以发现，a先定义，a的地址比b高，得出结论：**栈的内存分配是从高地址到低地址**
- **栈的内存是连续的** (这点也很重要！！)
- 

> OC方法的本质，其实是函数调用, 底层就是调用 objc_msgSend() 函数发送消息。

```objectivec
- (void)viewDidLoad {
    [super viewDidLoad];
  
    NSString  *test = @"666";
    id cls = [MNPerson class];
    void *obj = &cls;
    [(__bridge id)obj print];
}
```

以上述代码为例，三个变量 - test、cls、obj，都是局部变量，所以都在栈空间

栈空间是从高地址到低地址分配，所以test是最高地址，而obj是最低地址

MNPerson底层结构

```objectivec
struct MNPerson_IMPL{
    Class isa;
    NSString *_name;
}

- (void)print{
    NSLog(@"self.name = %@",self->_name);
}
```

1. 要打印的 `_name` 成员变量，其实是通过`self ->` 去查找；
2. 这里的 self，就是函数调用者；
3. `[(__bridge id)obj print];` 即通过 obj 开始找；
4. 而找 `_name` ，是通过指针地址查找，找得`MNPerson_IMPL` 结构体
5. 因为这里的 `MNPerson_IMPL` 里面就两个变量，所以这里查找 `_name`，就是通过 `isa` 的地址，跳过8个字节，找到 `_name`

![](https://raw.githubusercontent.com/cherishloveyou/summary/main/iOS/Assets/Question1/18569867-d14a6bd1dcda7071.png)

而前面又说过，cls = isa，而_name 的地址 = isa往下偏移 8 个字节，所以上面的图可以转成

![](https://raw.githubusercontent.com/cherishloveyou/summary/main/iOS/Assets/Question1/18569867-bdedc38d7a25c738.png)



_name的本质，先找到 isa，然后跳过 isa 的八个字节，就找到 _name这个变量

所以上图输出

```objectivec
self.name = 666
```

最早没有 test变量的时候呢

```objectivec
- (void)viewDidLoad {
    [super viewDidLoad];

    id cls = [MNPerson class];
    void *obj = &cls;
    [(__bridge id)obj print];
}
```

#### [super viewDidLoad];做了什么

底层 - objc_msgSendSuper

```
objc_msgSendSuper({ self, [ViewController class] },@selector(ViewDidLoad)),
```

等价于：

```kotlin
struct __rw_objc_super arg = {
    (id)self,
    (id)class_getSuperclass(objc_getClass("ViewController"))
};  
objc_msgSendSuper(arg, @selector(viewDidLoad));
```

所以等于有个局部变量 - 结构体 temp，

结构体的地址 = 他的第一个成员，这里的第一个成员是self



![](https://raw.githubusercontent.com/cherishloveyou/summary/main/iOS/Assets/Question1/18569867-bb47154d23a130b2.png)

所以等价于 _name = self = 当前ViewController，所以最后输出

```jsx
self.name = <ViewController: 0x7fc6e5f14970>
```

![](https://raw.githubusercontent.com/cherishloveyou/summary/main/iOS/Assets/Question1/18569867-832ba979a35c5834.png)

#### 话外篇 super 的本质

![](https://raw.githubusercontent.com/cherishloveyou/summary/main/iOS/Assets/Question1/18569867-7fc0669e27e1d36a.png)

**其实super的本质，不是 objc_msgSendSuper({self,[super class],@selector(xxx)}) **

而是

```kotlin
objc_msgSendSuper2({self,
  [current class]//当前类
},@selector(xxx)})
```

函数内部逻辑，拿到第二个成员 - 当前类，通过superClass指针找到他的父类，从superClass开始搜索，最终结果是差不多的~

[**转载请注明出处！**](https://www.jianshu.com/p/d88a83e37a04)

[**转载请注明出处！**](https://www.jianshu.com/p/d628f1b9e51e)