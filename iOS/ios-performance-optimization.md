# APP优化

### 卡顿优化 - CPU

- 尽量用轻量级的对象，比如用不到事件处理的地方，可以考虑使用CALayer取代UIView
- 不要频繁地调用UIView的相关属性，比如frame、bounds、transform等属性，尽量减少不必要的修改
- 尽量提前计算好布局，在有需要时一次性调整对应的属性，不要多次修改属性
- Autolayout会比直接设置frame消耗更多的CPU资源
- 图片的size最好刚好跟UIImageView的size保持一致
- 控制一下线程的最大并发数量
- 尽量把耗时的操作放到子线程
  - 文本处理（尺寸计算、绘制）
  - 图片处理（解码、绘制）
- 尽量避免短时间内大量图片的显示，尽可能将多张图片合成一张进行显示
- GPU能处理的最大纹理尺寸是4096x4096，一旦超过这个尺寸，就会占用CPU资源进行处理，所以纹理尽量不要超过这个尺寸
- 尽量减少视图数量和层次
- 减少透明的视图（alpha<1），不透明的就设置opaque为YES
- 尽量避免出现离屏渲染

### 离屏渲染

- 在OpenGL中，GPU有2种渲染方式
  - On-Screen Rendering：当前屏幕渲染，在当前用于显示的屏幕缓冲区进行渲染操作
  - Off-Screen Rendering：离屏渲染，在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作
- 离屏渲染消耗性能的原因
  - 需要创建新的缓冲区
  - 离屏渲染的整个过程，需要多次切换上下文环境，先是从当前屏幕（On-Screen）切换到离屏（Off-Screen）；等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上，又需要将上下文环境从离屏切换到当前屏幕
- 哪些操作会触发离屏渲染？
  - 光栅化，layer.shouldRasterize = YES
  - 遮罩，layer.mask
  - 圆角，同时设置layer.masksToBounds = YES、layer.cornerRadius大于0
    - 考虑通过CoreGraphics绘制裁剪圆角，或者叫美工提供圆角图片
  - 阴影，layer.shadowXXX
    - 如果设置了layer.shadowPath就不会产生离屏渲染

### 卡顿检测

- 平时所说的“卡顿”主要是因为在主线程执行了比较耗时的操作
- 可以添加Observer到主线程RunLoop中，通过监听RunLoop状态切换的耗时，以达到监控卡顿的目的

### 耗电优化

##### 耗电的主要来源

- CPU处理，Processing

- 网络，Networking

- 定位，Location

- 图像，Graphics

  

###### CPU处理

- 尽可能降低CPU、GPU功耗
- 少用定时器
- 优化I/O操作
  - 尽量不要频繁写入小数据，最好批量一次性写入
  - 读写大量重要数据时，考虑用dispatch_io，其提供了基于GCD的异步操作文件I/O的API。用dispatch_io系统会优化磁盘访问
  - 数据量比较大的，建议使用数据库（比如SQLite、CoreData）

###### 网络优化

- 减少、压缩网络数据
- 如果多次请求的结果是相同的，尽量使用缓存
- 使用断点续传，否则网络不稳定时可能多次传输相同的内容
- 网络不可用时，不要尝试执行网络请求
- 让用户可以取消长时间运行或者速度很慢的网络操作，设置合适的超时时间
- 批量传输，比如，下载视频流时，不要传输很小的数据包，直接下载整个文件或者一大块一大块地下载。如果下载广告，一次性多下载一些，然后再慢慢展示。如果下载电子邮件，一次下载多封，不要一封一封地下载

###### 定位优化

- 如果只是需要快速确定用户位置，最好用CLLocationManager的requestLocation方法。定位完成后，会自动让定位硬件断电
- 如果不是导航应用，尽量不要实时更新位置，定位完毕就关掉定位服务
- 尽量降低定位精度，比如尽量不要使用精度最高的kCLLocationAccuracyBest
- 需要后台定位时，尽量设置pausesLocationUpdatesAutomatically为YES，如果用户不太可能移动的时候系统会自动暂停位置更新
- 尽量不要使用startMonitoringSignificantLocationChanges，优先考虑startMonitoringForRegion:

###### 硬件检测优化

- 用户移动、摇晃、倾斜设备时，会产生动作(motion)事件，这些事件由加速度计、陀螺仪、磁力计等硬件检测。在不需要检测的场合，应该及时关闭这些硬件



### APP的启动

- APP的启动可以分为2种
  - 冷启动（Cold Launch）：从零开始启动APP
  - 热启动（Warm Launch）：APP已经在内存中，在后台存活着，再次点击图标启动APP
- APP启动时间的优化，主要是针对冷启动进行优化
- 通过添加环境变量可以打印出APP的启动时间分析（Edit scheme -> Run -> Arguments）
  - DYLD_PRINT_STATISTICS设置为1
  - 如果需要更详细的信息，那就将DYLD_PRINT_STATISTICS_DETAILS设置为

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f3a779bd195c429aa2e19ddfd5e1018f~tplv-k3u1fbpfcp-watermark.image)

- 更加详细的打印

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f81ae2adf1234ef08c963b0872bb2680~tplv-k3u1fbpfcp-watermark.image)
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c2fdb22176334466ac3eb792c3152c1d~tplv-k3u1fbpfcp-watermark.image)

- pre-main time : 调用main函数之前调用的时间

- dylib loading time ：动态库加载

- rebase/binding time ：

- objC setup time: 结构体准备时间

- initializer time ： 初始化耗时

- slowest intializers : 加载比较忙的库

  - libsystem.dylib

  - libMainThreadChecker.dylib

    

- APP的冷启动可以概括为3大阶段
  - dyld
  - runtime
  - main

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/919c55adba074e8abf61146f621383ed~tplv-k3u1fbpfcp-watermark.image)

### APP的启动 - dyld

- dyld（dynamic link editor），Apple的动态链接器，可以用来装载Mach-O文件（可执行文件、动态库等）
- 启动APP时，dyld所做的事情有
  - 装载APP的可执行文件Mach-o文件，同时会递归加载所有依赖的动态库
  - 当dyld把可执行文件、动态库都装载完毕后，会通知Runtime进行下一步的处理

### APP的启动 - runtime

- 启动APP时，runtime所做的事情有
  - 调用map_images进行可执行文件内容的解析和处理
  - 在load_images中调用call_load_methods，调用所有Class和Category的+load方法
  - 进行各种objc结构的初始化（注册Objc类 、初始化类对象等等）
  - 调用C++静态初始化器和__attribute__((constructor))修饰的函数
- 到此为止，可执行文件和动态库中所有的符号(Class，Protocol，Selector，IMP，…)都已经按格式成功加载到内存中，被runtime 所管理

### APP的启动 - main

- 总结一下

  - APP的启动由dyld主导，将可执行文件加载到内存，顺便加载所有依赖的动态库

  - 并由runtime负责加载成objc定义的结构

  - 所有初始化工作结束后，dyld就会调用main函数

  - 接下来就是UIApplicationMain函数，AppDelegate的application:didFinishLaunchingWithOptions:方

    

### APP的启动 - premain

#### 内存管理

内存是分页管理的，映射表不能以字节为单位，是 **以页为单位**。

- `macOS`以`4K`为一页
- `iOS`以`16K`一页

早期计算机没有虚拟地址，一旦加载都会 **全部加载到内存中** 。一旦物理内存不够了，那么应用就无法继续开启。应用在内存中的排序都是顺序排列的，这样进程只需要把自己的地址尾部往后偏移一点就能访问到别的进程中的内存地址，相当不安全。

`App` 启动后会认为已经获取到整个 `App` 运行所需的内存空间，但实际上并没有在物理内存上为他申请那么大的空间，只是生成了一张 **虚拟内存和物理内存关联的表** 。

 关于虚拟内存和物理内存？

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/76b69b3d07d3490d944c52dbd4c248ed~tplv-k3u1fbpfcp-watermark.image)

当 `App` 需要使用某一块虚拟内存的地址时，会通过这张表查询该虚拟地址是否已经在物理内存中申请了空间。

- 如果已经申请了则通过表的记录访问物理内存地址，
- 如果没有申请则申请一块物理内存空间并记录在表中（`Page Fault`）。

这个通过进程映射表映射到不同的物理内存空间的操作叫 **地址翻译** ，这个过程需要 `CPU` 和操作系统配合。

#### Page Fault

当数据未在物理内存会进行下列操作

- 系统阻塞该进程
- 将磁盘中对应`Page`的数据加载到内存
- 把虚拟内存指向物理内存

上述行为就就是`Page Fault`

#### Rebase && Bind

有两种主要的技术来保证应用的安全：ASLR和Code Sign。

ASLR的全称是Address space layout randomization，翻译过来就是“地址空间布局随机化”。App被启动的时候，程序会被影射到逻辑的地址空间，这个逻辑的地址空间有一个起始地址，而ASLR技术使得这个起始地址是随机的。如果是固定的，那么黑客很容易就可以由起始地址+偏移量找到函数的地址。

Code Sign相信大多数开发者都知晓，这里要提一点的是，在进行Code sign的时候，加密哈希不是针对于整个文件，而是针对于每一个Page的。这就保证了在dyld进行加载的时候，可以对每一个page进行独立的验证。

mach-o中有很多符号，有指向当前mach-o的，也有指向其他dylib的，比如`printf`。那么，在运行时，代码如何准确的找到`printf`的地址呢？

mach-o中采用了PIC技术，全称是Position Independ code。当你的程序要调用`printf`的时候，会先在`__DATA`段中建立一个指针指向printf，在通过这个指针实现间接调用。dyld这时候需要做一些fix-up工作，即帮助应用程序找到这些符号的实际地址。主要包括两部分

- Rebase 修正内部(指向当前mach-o文件)的指针指向。`ASLR+偏移值 = 运行时确定的内存地址`,造成这种现象的原因是因为系统运用了虚拟内存。
- Bind 修正外部指针指向

之所以需要Rebase，是因为刚刚提到的ASLR使得地址随机化，导致起始地址不固定，另外由于Code Sign，导致不能直接修改Image。Rebase的时候只需要增加对应的偏移量即可。待Rebase的数据都存放在`__LINKEDIT`中。 **Rebase解决了内部的符号引用问题，而外部的符号引用则是由Bind解决。**
Bind，就是给符号赋值的过程。例如NSLog方法，在编译时期生成的mach-o文件中，会创建一个符号！NSLog（目前指向一个随机的地址），然后在运行时（从磁盘加载到内存中，是一个镜像文件），会将真正的地址给符号（即在内存中将地址与符号进行绑定，是dyld做的，也称为动态库符号绑定）
在解决Bind的时候，是根据字符串匹配的方式查找符号表，所以这个过程相对于Rebase来说是略慢的。

#### 二进制重排

虚拟内存技术会产生缺页中断（`Page Fault`），这个过程是个耗时操作。
每页耗时也有很大差距，1微秒到0.8毫秒不等。
使用过程中对这点耗时感觉不明显，但是启动时加载大量数据，如果产生大量缺页中断（`Page Fault`），时间叠加后用户会有明显感知。
如果我们把所有启动时候的代码都放在一页或者两页，这样就很大程度减少了启动时的缺页中断（`Page Fault`）从而优化启动速度，这就是二进制重排。

##### `Page Fault`检测操作：

1. 使用Instrument中的system trace，将应用跑起来
2. 查看虚拟内存中的File Backed Page In中的count，代表`page fault`的次数，下图是159次（注意：iOS中一页16k）

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e8be434a4bcf4d8084026344f890dffd~tplv-k3u1fbpfcp-watermark.image)

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11b81512549442c294ff4cf1294389a1~tplv-k3u1fbpfcp-watermark.image)

##### 基于PGO优化启动

 PGO（*Profile Guided Optimization）是苹果官方提供的工具，具体使用方法是点击xcode工具栏 Product -> Perform Action -> Generate Optimization Profile，可以大幅度降低page In 次数。

##### xcode来进行二进制重排用配置的

 我们以`objc750`源码为例子，在xcode的Build Setting中查找`order file`，最终会在路径下生成`libobjc.order`的文件，里面存储的都是符号文件，见下图，苹果编译的动态库，就是按照`order`来进行排列的,也就是说二进制重排苹果一直是有使用到的

同理，我们自己也可以配置一个order文件路径，在这个 order 文件中，将你需要的符号按顺序写在里面，而当当工程 build 的时候 , Xcode 会读取这个文件 , 打的二进制包就会按照这个文件中的符号顺序进行生成对应的 mach-O

那么如何查看我们工程中的符号顺序呢？

> 在Build Setting中搜索Write Link Map File,将其设置为YES，意思是替我们生成一个符号文件，我们点击Products中的.App,然后show in finder，最终找到它。

头部告诉我们具体链接了哪些.o文件，而这个顺序是和`Build Phases`中的`Compile Sources`顺序保持一致。

上述文件中最左侧地址就是 实际代码地址而并非符号地址 , 因此我们二进制重排并非只是修改符号地址 , 而是利用符号顺序 , 重新排列整个代码在文件的偏移地址 , 将启动需要加载的方法地址放到前面内存页中 , 以此达到减少 page fault 的次数从而实现时间上的优化 , 一定要清楚这一点 . 你可以利用 MachOView 查看排列前后在 _text 段 ( 代码段 ) 中的源码顺序来帮助理解 .

在以上基础上我们来实战操作一下： 在demo中创建order文件，然后在`order file`中指定路径，我们在order文件中写入符号，编译后会发现此时顺序就是按照我们写入顺序来，而不是按照`Build Phases`中的`Compile Sources`顺序来的，因此剩下的任务就是拿到启动时候的方法

**Clang插桩** 用fishHook来hook方法 objc_msgSend来拿到其第二个参数SEL

抖音的方案存在局限和瓶颈，initialize hook不到，部分block hook不到，C++通过寄存器的间接函数调用静态扫描不出来，而使用Clang插桩的方式可以解决这个问题，做到全部hook。llvm内置了一个简单的代码覆盖率检测（`SanitizerCoverage`）。它在函数级、基本块级和边缘级插入对用户定义函数的调用

1. Other C Flags 来到 Apple Clang - Custom Compiler Flags 中 , 添加

> OC项目则添加-fsanitize-coverage=func,trace-pc-guard，如果是swift项目则添加 -sanitize-coverage=func`和`-sanitize=undefined

1. 按文档在代码中添加如下代码（我是直接加在demo中viewDidLoad中）

`start`和`stop`这个内存区间保存的就是工程所有符号的个数。

```objective-c
void __sanitizer_cov_trace_pc_guard_init(uint32_t *start,
                                         uint32_t *stop) {
  static uint64_t N;  // Counter for the guards.
  if (start == stop || *start) return;  // Initialize only once.
  printf("INIT: %p %p\n", start, stop);
  for (uint32_t *x = start; x < stop; x++)
    *x = ++N;  // Guards should start from 1.
}

void __sanitizer_cov_trace_pc_guard(uint32_t *guard) {
  if (!*guard) return;  // Duplicate the guard check.

  void *PC = __builtin_return_address(0);
  char PcDescr[1024];
  //__sanitizer_symbolize_pc(PC, "%p %F %L", PcDescr, sizeof(PcDescr));
  printf("guard: %p %x PC %s\n", guard, *guard, PcDescr);
}
```

### APP的启动优化

**一、改善启动时pre-main阶段**

1. **加载 Dylib**

  载入动态库，这个过程中，会去装载app使用的动态库，而每一个动态库有它自己的依赖关系，所以会消耗时间去查找和读取。对于Apple提供的的系统动态库，做了高度的优化。而对于开发者定义导入的动态库，则需要在花费更多的时间。Apple官方建议**尽量少的使用自定义的动态库，或者考虑合并多个动态库，其中一个建议是当大于6个的时候，则需要考虑合并它们。**

2. **Rebase/Binding**

  **减少App的Objective-C类,分类和Selector的个数**。这样做主要是为了加快程序的整个动态链接, 在进行动态库的重定位和绑定(Rebase/binding)过程中减少指针修正的使用，加快程序机器码的生成

3. **Objc setup**

    在这一步倒没什么优化可做的，Rebase/Bind阶段优化好了，这一步的耗时也会减少。

   注册Objc类 (class registration)

   把category的定义插入方法列表 (category registration)

   保证每一个selector唯一 (selctor uniquing)

4. **Initializers**

   到了这一阶段，dyld开始运行程序的初始化函数，调用每个Objc类和分类的+load方法，调用C/C++ 中的构造器函数(用attribute((constructor))修饰的函数)，和创建非基本类型的C++静态全局变量。Initializers阶段执行完后，dyld开始调用main()函数。

  在这一步，我们可以做的优化有：

  ***少在类的+load方法里做事情，尽量把这些事情推迟到+initiailize***

  ***减少构造器函数个数，在构造器函数里少做些事情***

  ***减少C++静态全局变量的个数***

  ***使用Swift代码编写工程，会避免很多C、C++和Objective-C才有的陷阱***
  - Swift没有初始化器
  - Swift不允许特定类型的未对齐数据结构，因为我们的cpu读取不同长度的数据结构内存时，需要切换与之对应的长度才行，当内存里面有很多长度不一的数据结构，那么读取起来，就要不停切换，这样十分消耗性能，所以苹果系统才会出现内存对齐这样的操作。而Swift就避免的这样的坑。这也是能提高性能，缩短启动时间的原因
  - Swift代码更加精简
  - Swift尽量使用struct


##### 二、main()阶段的优化**

1. 核心点：didFinishLaunchingWithOptions方法

   这一阶段的优化主要是**减少didFinishLaunchingWithOptions方法里的工作**，在didFinishLaunchingWithOptions方法里我们经常会进行：

   - 创建应用的window，指定其rootViewController，调用window的makeKeyAndVisible方法让其可见；
   - 由于业务需要，我们会初始化各个三方库；
   - 设置系统UI风格；
   - 检查是否需要显示引导页、是否需要登录、是否有新版本等；

   由于历史原因，这里的代码容易变得比较庞大，启动耗时难以控制。

2. 优化点：

   满足业务需要的前提下，didFinishLaunchingWithOptions在主线程里做的事情越少越好。在这一步，我们可以做的优化有：

   - 梳理各个二方/三方库，**把可以延迟加载的库做延迟加载处理**，比如放到首页控制器的viewDidAppear方法里
   - 梳理业务逻辑，**把可以延迟执行的逻辑做延迟执行处理**。比如检查新版本、注册推送通知等逻辑。
   - **避免复杂/多余的计算**。
   - **避免在首页控制器的viewDidLoad和viewWillAppear做太多事情**，这2个方法执行完，首页控制器才能显示，部分可以延迟创建的视图应做延迟创建/懒加载处理。
   - **首页控制器用纯代码方式来构建**。

​       

### 安装包瘦身

安装包（IPA）主要由可执行文件、资源组成

1. 资源（图片、音频、视频等）

   - 采取无损压缩
   - 去除没有用到的资源： [LSUnusedResource](https://github.com/tinymind/LSUnusedResources)

2. #### 可执行文件瘦身

   ##### 编译器优化

   - Strip Linked Product、Make Strings Read-Only、Symbols Hidden by Default设置为YES
   - 去掉异常支持，Enable C++ Exceptions、Enable Objective-C Exceptions设置为NO， Other C Flags添加-fno-exceptions

   ##### 找寻无用代码

   - 编写LLVM插件检测出重复代码、未被调用的代码

   - 利用AppCode[检测未使用代码](https://www.jetbrains.com/objc/）检测未使用的代码：菜单栏) -> Code -> Inspect Code

   - LinkMap

     1）通过LinkMap获取所有的代码类和方法信息，LinkMap是什么？怎么获取？见博客[LinkMap](https://www.jianshu.com/p/4bd6d1315104)

     2）通过Mach-O获取使用过的类和方法。

     3）取差集得到无用代码

     4）最后由人工进行确认无用代码后，进行删除。

     第2步是难点，关键在于如何检测有哪些类被使用过？

     原理概述：

     1. 其实原理比较简单，每个类在即将被使用之前，会进行一些惰性初始化工作，这个过程叫做 realize。类结构中有特定的 flag (一个 bit 位）标识当前类是否已经完成了 realize，一旦某个类被使用过，它的这个 flag 便已被置为了1

     2. 在合适的时机遍历所有类，通过这个 bit 位判断是否是 realized，遍历可以得知有哪些类是使用过的了，将使用过的类进行上报

     3. 上线一定时间后，汇总上报的数据便可以得到线上环境所有被使用过的类，用全部的类减掉使用过的类，便是无用类了

     ```objectivec
     #define RW_INITIALIZED (1<<29)
     bool isInitialized() {
        return getMeta()->data()->flags & RW_INITIALIZED;
     }
     ```

     上述是Objc的runtime源码，判断一个类是否初始化过的函数。

     isInitialized的结果会保存在元类的class_rw_t结构体的flags信息里，flags的1<<29位记录的就是这个类是否初始化的信息。

     

     `Mach-o`文件中`__DATA__objc_classrefs`段记录了引用类的地址，`__DATA__objc_classlist`段记录了所有类的地址，取差集可以得到未使用的类的地址，然后进行符号化，就可以得到未被引用的类信息。

     ```objective-c
     __objc_selrefs 里的方法一定是被调用了的。__objc_classrefs 里是被调用过的类，__objc_superrefs 是调用过 super 的类。通过 __objc_classrefs 和 __objc_superrefs，我们就可以找出使用过的类和子类。
     ```

     ```python
     def referenced_selectors(path):
         re_sel = re.compile("__TEXT:__objc_methname:(.+)") //获取所有方法
         refs = set()
         lines = os.popen("/usr/bin/otool -v -s __DATA __objc_selrefs %s" % path).readlines() # ios & mac //真正被使用的方法
         for line in lines:
             results = re_sel.findall(line)
             if results:
                 refs.add(results[0])
         return refs
     }
     ```

     - 生成LinkMap文件，可以查看可执行文件的具体组成

     ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae6532f670e1467a9a3d9f2d62de3964~tplv-k3u1fbpfcp-watermark.image)

     - 可借助第三方工具解析LinkMap文件： [LinkMap工具](https://github.com/huanxsd/LinkMap)

     

[启动优化二进制重排](https://juejin.cn/post/6844904165773328392)

[抖音研发实践：基于二进制文件重排的解决方案 APP启动速度提升超15%](https://mp.weixin.qq.com/s/Drmmx5JtjG3UtTFksL6Q8Q)

[包大小治理总结](https://www.pianshen.com/article/7418425261/)

[**浅析快手iOS启动优化方式——动态库懒加载**](https://mp.weixin.qq.com/s?__biz=MzkxOTI0MTA2OA==&mid=2247487524&idx=1&sn=6dfcc2d762ba1afbd3b0a5a178ec3cba&scene=21#wechat_redirect) 

