# dyld加载应用启动原理详解

我们都知道APP的入口函数是main()，而在main()函数调用之前，APP的加载过程是怎样的呢？接下来我们一起来分析APP的加载流程。

## 一. 准备工作

由于load()比main()调用更早，因此我们创建一个工程，在控制器中写一个load()函数，并断点运行，如下图：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/89913aae6149456d9ee3008722cb9388~tplv-k3u1fbpfcp-zoom-1.png)

运行起来之后，可以清晰的看到比较详细的函数调用顺序，从`_dyld_start()`到`dyld:notifySingle()`，频率出现最多的就是这个dyld，那么dyld是什么？它在做什么？

简单来说dyld是一个动态链接器，用来加载所有的库和可执行文件。接下来我们将通过对dyld源码分析，去追踪dyld到底做了什么？

![_dyld_start](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/576c9a1685df42a99d9e1d9c38c6a256~tplv-k3u1fbpfcp-zoom-1.png)



## 二. dyld加载流程分析

### 1. 首先下载[dyld源码](https://opensource.apple.com/tarballs/dyld/)。

### 2. 打开dyld源码工程，根据上图`dyldbootstrap::start`为关键字搜索`dyldbootstrap`中调用的`start()`，如下图：

![dyldbootstrap::start](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8f83bc2f251741c293bbfeb57858da8b~tplv-k3u1fbpfcp-zoom-1.png)

### 3. 进入dyld的`start`函数

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9d345abc52de40d5b43fa36450411f14~tplv-k3u1fbpfcp-zoom-1.png)

其中`rebaseDyld()`分析如下：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7473615cba1f4ce791095e139f8be2e1~tplv-k3u1fbpfcp-zoom-1.png)

### 4. 进入dyld的`main`函数

> 注：因为`dyld::main()`函数代码比较多，以下会分段介绍，也会介绍相对来说比较重要的函数。

#### 4.1 内核检测

#### 4.2 获取main执行文件的cdHash缓存区

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11a5dbb0a8b34f7c911fe0c00345ac43~tplv-k3u1fbpfcp-zoom-1.png)

#### 4.3 获取CPU信息

```c
    // 获取CPU信息
    getHostInfo(mainExecutableMH, mainExecutableSlide);
```

#### 4.4 设置MachHeader和内存偏移量

```c
    // 设置MachHeader和内存偏移量Slide
    uintptr_t result = 0;
    sMainExecutableMachHeader = mainExecutableMH;
    sMainExecutableSlide = mainExecutableSlide;
```

#### 4.5 设置上下文

```c
    // 设置上下文，保存信息
    setContext(mainExecutableMH, argc, argv, envp, apple);
```

#### 4.6 配置进程限制

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f1d0a5d8a785461aac26b950b1ead664~tplv-k3u1fbpfcp-zoom-1.png)

#### 4.7 检测环境变量

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f16a820ef6a484188084f9e29b1ddd1~tplv-k3u1fbpfcp-zoom-1.png)

#### 4.8 打印环境配置信息

```c
    // 打印环境配置信息
    if ( sEnv.DYLD_PRINT_OPTS )
        printOptions(argv);
    if ( sEnv.DYLD_PRINT_ENV ) 
        printEnvironmentVariables(envp);
```

此处可以自己定义环境变量配置，回到刚才创建的新工程中，在`Edit Scheme` -> `Run` -> `Arguments` -> `Environment Variables` 添加两个参数 `DYLD_PRINT_OPTS` 和 `DYLD_PRINT_ENV`，并设置测试value值，如下：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d14cdf0af6c425b8ac7c45c9a1aa337~tplv-k3u1fbpfcp-zoom-1.png)

运行程序，可看到如下打印信息：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef3140246c9e4b8ea550492bcd97cb61~tplv-k3u1fbpfcp-zoom-1.png)

#### 4.9 加载共享缓存（如果没有共享缓存，iOS将无法运行）

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5e9db612473f49fbbbd89b341da02ca9~tplv-k3u1fbpfcp-zoom-1.png)

主要函数`mapSharedCache()`如下：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b8e29e067b374f6e810419193026a1d1~tplv-k3u1fbpfcp-zoom-1.png)

#### 4.10 `dyld`配置

**(1) `dyld3`(闭包模式)**

iOS11版本之后，引入dyld3闭包模式(ClosureMode)：加载速度更快，效率更高。

**开始执行闭包模式**

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb05550938004178a75fdfc0950c7382~tplv-k3u1fbpfcp-zoom-1.png)

**判断是否开启了闭包模式**

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf5dd3efaf374bafa1028f946f12f19a~tplv-k3u1fbpfcp-zoom-1.png)

**启动闭包模式加载**

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9715d73cba054b82aecb85351c772bfb~tplv-k3u1fbpfcp-zoom-1.png)

其中`launchWithClosure`加载闭包，会把后面说到的`dyld2`的大部分流程都封装到`launchWithClosure()`这个函数里面了，这里不再细说`launchWithClosure`，因为在接下来的`dyld2`(非闭包)中会详细解释整个dyld加载的流程，也就是`launchWithClosure`实现过程。

**(2) `dyld2`(非闭包模式)**

**开始执行闭包模式**

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/88439f3dd741469ebd7675fcf03cfc3b~tplv-k3u1fbpfcp-zoom-1.png)

**把dyld加入到UUID列表**

```c
    // 把dyld加入到UUID列表
    addDyldImageToUUIDList();
复制代码
```

**配置缓存代理**

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e6f111f8b4254713b2e15afb3c98d8dc~tplv-k3u1fbpfcp-zoom-1.png)

dyld4的针对于的`mach-o`解析器 ( iOS上可执行文件格式是Mach-O格式, 下方也有具体解释) 方面跟dyld3相同，但是引入了 JustInTime 的加载器来优化。

- `dyld3`:  相比`dyld2`新增预构建/闭包, 目的是将一些启动数据创建为闭包存到本地，下次启动将不再重新解析数据，而是直接读取闭包内容
- `dyld4`: 采用`pre-build + just-in-time` 预构建/闭包+实时解析的双解析模式, 将根据缓存有效与否选择合适的模式进行解析, 同时也处理了闭包失效时候需要重建闭包的性能问题。

#### 4.11 创建主程序的Image

开始创建主程序的Image，通过`instantiateFromLoadedImage()`，调用`instantiateMainExecutable()`，实例化具体的Image类，最后生成的对象，设置到`gLinkContext`中。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1e925fb1cd67464984b1dc7f45d2c8e8~tplv-k3u1fbpfcp-zoom-1.png)

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8269b828e95e462f969b9252ac11df35~tplv-k3u1fbpfcp-zoom-1.png)

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6b874351066b41c4a4f6c96417bcd6e6~tplv-k3u1fbpfcp-zoom-1.png)

#### 4.12 设置动态库的版本

```c
// 加载完共享缓存，设置动态库的版本
checkVersionedPaths();
```

#### 4.13 加载插入的动态库

```c
// 加载插入的动态库
if( sEnv.DYLD_INSERT_LIBRARIES != NULL ) {
    for (const char* const* lib = sEnv.DYLD_INSERT_LIBRARIES; *lib != NULL; ++lib) 
        loadInsertedDylib(*lib);
}
```

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a2e2c224b54f43d2b62810837318d37e~tplv-k3u1fbpfcp-zoom-1.png)

#### 4.14 链接主程序和动态库

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/84ad8584f3c744bab3895dfb88f4e193~tplv-k3u1fbpfcp-zoom-1.png)

- 其中的`link()`函数，函数体调用了`image->link()`，函数具体如下：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8c8b5fdbacda40b58e2a039ecf561b51~tplv-k3u1fbpfcp-zoom-1.png)

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e373a7d1ff044df595826a8a881b4a67~tplv-k3u1fbpfcp-zoom-1.png)

- 判断是否需要重新加载所有的Image

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1d13532039c24cbb8613380822cd0dc0~tplv-k3u1fbpfcp-zoom-1.png)

#### 4.15 绑定主程序和动态库

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/670f1ce1474a4e949b5304ddc660b1a7~tplv-k3u1fbpfcp-zoom-1.png)

#### 4.16 初始化主程序

根据Demo上的堆栈信息，如下：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ef588c9c48e543619f76c39227517d16~tplv-k3u1fbpfcp-zoom-1.png)

终于看到熟悉的函数了，那么dyld加载流程也快结束了。 根据堆栈信息，获取函数调用层级关系。

```c
    // 初始化主程序
    initializeMainExecutable(); 
复制代码
```

- 查找`runInitializers()`： `dyld::initializeMainExecutable()` -> `ImageLoader::runInitializers()`

![加载主程序](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/520594150f864cf2864451abf4dfee93~tplv-k3u1fbpfcp-zoom-1.png)

- 查找`processInitializers()`： `ImageLoader::runInitializers()` -> `ImageLoader::processInitializers()`

![runInitializers](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bddca3a027cc418c9ac0e9e4cee5a335~tplv-k3u1fbpfcp-zoom-1.png)

- 查找`recursiveInitialization()`： `ImageLoader::processInitializers()` -> `ImageLoader::recursiveInitialization()`

![processInitializers](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9b03e333e66b4a4c8e8d58ce02867b66~tplv-k3u1fbpfcp-zoom-1.png)

- 查找`notifySingle()`： `ImageLoader::recursiveInitialization()` -> `dyld::notifySingle()`

![recursiveInitialization](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ffab51d7e614e249d3aabbd3171114d~tplv-k3u1fbpfcp-zoom-1.png)

- 查找`load_images()`：在 `dyld::notifySingle()`中并没有找到`load_images()`，但是找到了`sNotifyObjCInit()`，该字段是objc函数回调。在 `dyld::notifySingle()`中执行了这个回调，那就需要追溯到谁去注册的这个回调了。
- 全局查找`sNotifyObjCInit()`赋值的地方。在`registerObjCNotifiers()`中赋值，如下：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/82ae4449d13745719697b4fba2fba174~tplv-k3u1fbpfcp-zoom-1.png)

- 全局查找`registerObjCNotifiers`，在`_dyld_objc_notify_register()`中调用，且第二个参数是我们需要的。如下：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6765dfdcb4f744bc91801febb8de1740~tplv-k3u1fbpfcp-zoom-1.png)

- 全局查找`_dyld_objc_notify_register()`，并没有在dyld源码库里找到，此时需要在源工程中，打符号断点`_dyld_objc_notify_register`，重新编译执行，可以看到是`_objc_init()`调用了。此时只能去查找[objc源码](https://opensource.apple.com/tarballs/objc4/)了。
- `objc`源码分析，在`objc-os.mm`文件中找到`_objc_init()`函数，其中注册了`_dyld_objc_notify_register`回调。

![objc-os.mm](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dac5864ca0dd4c08ba0334f0bd2f89b8~tplv-k3u1fbpfcp-zoom-1.png)

其中第二个参数就是`load_images()`，在`load_images()`中也找到了`call_load_methods()`。

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/00886707c9c042f7a0b8fba35b39e1a6~tplv-k3u1fbpfcp-zoom-1.png)

此时初始化程序还未执行完成，回到之前的 `ImageLoader::recursiveInitialization()`方法中。

- 执行`this->doInitialization()`函数

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1b9b5fe08a74b659a132da1ad1638fc~tplv-k3u1fbpfcp-zoom-1.png)

- 发送通知，初始化主程序完成。

#### 4.17 进入主程序

```c
//通知此进程将要进入程序main()
notifyMonitoringDyldMain();
```

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ac5de15e1f4d4f9d99fbef8292c61b8b~tplv-k3u1fbpfcp-zoom-1.png)

到此，`start()` -> `main()`，全部执行完毕。

## 三. 总结

- dyld(动态链接器)：是苹果操作系统一个重要组成部分，加载所有的库和可执行文件。

- dyld加载流程：

  - 从`_dyld_start()`开始 -> `dyldbootstrap::start()`

  - 进入dyld的`main()`

  - 检测内核，配置重定向信息：`rebase_dyld()`

  - 加载共享缓存

  - ```
    dyld2/dyld3(闭包模式)
    ```

    - 实例化主程序

    - 加载动态链接库 (主程序和动态库的image都会加载allImage里面：`loadAllImage`，主程序在第`0`位置)

    - 链接主程序、动态库、绑定符号(非懒加载、弱符号)等

    - ```
      关键的初始化方法 
      ```

      - `initializeMainExecutable()`

      - `ImageLoader::runInitializers()`

      - `ImageLoader::processInitializers()`

      - `ImageLoader::processInitializers()`

      - `ImageLoader::recursiveInitialization()`

      - `dyld::notifySingle()`

        - 此函数执行一个回调`_dyld_objc_notify_register()`
        - 通过断点调试：此回调是`_objc_init()`初始化时赋值的一个函数`load_images()`，里面执行了`call_load_methods()`函数，其作用是循环调用各个类的方法。

      - `doModInitFunctions()`函数：内部会调用全局C++对象的构造函数 `__attribute__((constructor))`的C函数

    - 返回主程序入口，执行`main`函数

## dyld历史

`C++`的特性：

- `C++`初始化器排序方式，他们在静态环境中工作良好，但是在动态环境中，就可能降低性能。因此大型`C++`代码库导致动态链接器需要完成大量的工作，从而速度会变慢。

为了解决这一系问题，苹果系统的开发者，也想出来很多办法来解决。方法的其中之一   -----    `预绑定技术`

系统使用预绑定技术为系统中的所有`dylib`和我们的工程，找到固定地址。那么动态加载器将会加载这些地址的所有内容。加载成功之后，将编译地址里面多有对应的二进制数据。以此来获取所有预计算地址，然后下次调用的时候，就把真正的数据放入到所对应的地址里面。

也就是说，当程序第一次加载时，系统会分配相应的地址，而这些地址里面存储的内容，就是脏数据，用来进行占位的。而再次进行调用的时候，就会把真正的数据填入到对应的地址里面。这样就避免重复的操作，大大的提高了程序的运行速度。

但是这样也存在缺陷，就是每次启动的时候，都会编译工程里面的所有的二进制数据。这样还是有很大的工作量。而且在安全上也有问题。

这样就有了`dyld 2`的推出。

## `dyld2`

### `dyld2`更好的支持`C++`

`dyld 2`是`macOS Tiger`的组成部分，是`dyld`的完全重写版本；他正确支持`C++`初始化器语义，系统扩展了`mach-O`格式，并且更新`dyld`。从而获得高效率的`C++`库支持；`dyld2`还具有完整的本机`dlopen`和`dlsymd`实现。同时还改进一些功能，提高平台的安全性。

### `dyld2`减少了预绑定内容

以前`dyld`的预绑定是处理系统中的所有`dylib`和我们的工程，而`dyld2`仅仅编辑系统库，这就减少预绑定的工作量。

### `dyld2`增加了基础结构和平台

`dyld 2`增加了大量的基础结构和平台，如`x86`、`x86_64`、`arm`、`arm64`和许多的衍生平台。同时还有`iOS`、`tvOS`、`watchOS`，都是需要匹配`dyld 2`的。

这样做的好处就是，真机只需要调用`arm64`架构的逻辑执行，而不需要加载其他架构里面的内容。能够针对不同架构来分别对待执行。

### `dyld2`增强安全性

首先是增加`代码签名`和`ASLR`，也就是地址空间配置随机加载，每次运行得到的地址，都是不同的，所以也不容易被`hook`。

### `dyld2`增加了`mach-O`文件头部中的项目

这个是重要的边界检查，可以避免恶意的二进制数据的加入。

### `dyld2`增强使用`Share Cache`

就是消除预绑定，转而使用`Share Cache`(共享缓存代码)。 `Share Cache`是一个单个文件，含有大多数系统`dylib`，因此可以进行优化。

`Share Cache`还调整所有文本段(`TEXT`)和所有数据段(`DATA`)，重写整个符号表，目的是减小内容大小，使得每个进程只需要挂载少量的区域。

`Share Cache`允许打包二进制数据段，从而节省大量的`RAM`，运行时可以节约`500M-1GB`内存空间。

`Share Cache`实际上是`dylib`预链接器；

`Share Cache`他还是预生产数据结构供`dyld`和`Objective-C`，在运行时使用。所以在程序启动时，不再做那么多大量的加载工作量。提高了系统的性能

### `dyld2`的启动流程

- 先分析工程的`mach-O`文件，弄清楚需要哪些库(`递归处理`)；
- 然后映射到所有mach-O文件，将之放入地址空间；
- 再执行符号表查询，将需要的内容复制到对应的地址指针上；
- 再进行绑定和基址重置，基础地址再与上一个随机地址，这一步是增加安全性；
- 最后，运行工程所有初始化器，准备执行main函数。

![7269E91A-B956-48EA-B125-38816B8046EC.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ff0a366d153c4e18b513dd62e0054744~tplv-k3u1fbpfcp-watermark.image)

## dyld3

`dyld 3`的一个最突出的变革，就是`动态链接器`，是现在`macOS system`的默认配置，将全面取代`dyld 2`。

- `dyld 3`提升了性能；
- `dyld 3`增强安全性；
- `dyld 3`更容易被测试。

### dyld3提升性能问题

为了提高程序启动和运行的速度，系统将大多数的`dyld`移出了进程，所以现在它只是普通的后台程序，可以使用标准的测试工具进行测试。也为后面进一步提高性能和运行速度打下了基础。

另外，也允许少量部分`dyld`驻留在进程中，从而减少程序的受攻击面积。代码少了，代码的运行速度就提高了，因此，也提高了启动速度。

在`dyld 3`中，将`mach-O`文件处理和符号表的查询，都写入磁盘。 `dyld 3`包含三个部分：

- 它是一个进程外`mach-O`分析器和编译器；
- 也是一个进程内引擎执行启动收尾处理；
- 也是一个启动收尾缓存服务；

大多数程序启动都会使用缓存，但始终不需要调用进程外`mach-O分析器`或`编译器`；启动收尾比`mach-O`更简单，他们是内存映射文件，不需要用复杂的方法进行分析，因此，提高了运行速度。

### dyld3增强安全性

在`dyld 2`中，增加了一些安全特性，但是很难跟随现实情形来增强安全性。

以前可以通过分析`mach-O`文件头和查找依赖关系，只需要使用撰改过的`mach-O`文件头进行攻击，修改相应的路径或者插入库到适当的位置，就能破坏原有的程序。

因此，现在是在后台的进程之外，完成了`mach-O`文件的一系列工作。

符号表的查找是十分耗时的操作，占用了大量的缓冲部分。因为在给定的库中，除非进行软件更新或者在磁盘上更改库，符号将始终位于库中的相同偏移位置。

dyld3不再需要分析mach-O头文件，或者查找符号表，这些都是十分耗时的操作，这样也就提高了运行速度。

### dyld3更容易被测试

在以前，比较依赖于动态链接器的底层功能，将它们的库插入进程，因而不能用于测试现有的`dyld`代码，所以对`dyld`的安全性和性能水平难以测试。

### dyld3还是一个启动收尾缓存服务

苹果系统直接将程序收尾直接加入到共享缓存中，使启动收尾映射到缓存中，所有`dylib`都使用这个缓存来启动，就更加快捷了。也是提高程序的启动速度。

对于第三方程序，在使用或更新的时候，就会生成对应的收尾处理。

- dyld3完全兼容dyld2

### dyld3对符号的解析

`dyld`必须加载所有符号，这需要占用大量资源，因此应该使用缓存，直接运行现有程序，会占用很多资源，将会花费很长的时间，为了解决这一问题，苹果系统就使用了一种机制----懒加载符号解析。

- 懒加载符号解析：默认情况下，库中的函数指针，如`printf`并不是指向`printf`，默认情况下，它是指向`dyld`中的一个函数，此函数返回一个指针指向`printf`，因此启动时，调用`printf`将会进入`dyld`，返回`printf`进行首次调用，然后第二次就直接调用`printf`。

`dyld 2`懒加载符号解析：

- 符号的查询太繁琐；
- 每个符号在第一次调用时都会被查找;
- 缺少符号会在第一次调用它们时导致崩溃;

`dyld 3`懒加载符号解析：

- 所有的符号查找都是缓存的，由于已经缓存并计算所有符号，因此在程序启动时，不会产生额外开销来绑定它们，所以速度非常快;
- 可以检查是否存在所有符号，预先解析所有符号，如果缺失将进行奔溃，而不是在使用程序时再奔溃;

### dyld3的运行流程

- 解析所有路径、环境；
- 先分析工程的`mach-O`二进制数据；
- 再执行符号表查询，将需要的内容复制到对应的地址指针上；
- 利用这些结果创建收尾处理，向磁盘写入收尾处理；
- 检查启动收尾处理是否正确；
- 然后映射到`dylib`之中
- 再跳转到main函数里面

![A37C16CE-A69E-47CA-A194-B4E49A6D680F.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2b061c183e6b42ac9a8c3f08b4a68a45~tplv-k3u1fbpfcp-watermark.image)






### dyld4的运行流程
- dyldStartup；

- "prepare"准备 方法`MainFunc appMain = prepare(state, dyldMA);`处理相关依赖绑定

- 看下`gProcessInfo`有`struct dyld_all_image_infos* gProcessInfo = &dyld_all_image_infos;`是一个存储dyld所有镜像信息的一个结构体

- 进行pre-build, 创建mainLoader（主装载器）, 如果熟悉`dyld3`的小伙伴知道, 旧版本是创建一个`ImageLoader`镜像装载器

-  创建just-in-time。`dyld3` 出于对启动速度的优化的目的, 增加了`预构建(闭包)`。App第一次启动或者App发生变化时会将部分启动数据创建为`闭包`存到本地，那么App下次启动将不再重新解析数据，而是直接读取闭包内容。当然前提是应用程序和系统应很少发生变化，但如果这两者经常变化等, 就会导闭包丢失或失效。所以`dyld4` 采用了 `pre-build + just-in-time` 的双解析模式，`预构建 pre-build` 对应的就是 `dyld3` 中的闭包，`just-in-time` 可以理解为`实时解析`。当然`just-in-time` 也是可以利用 `pre-build` 的缓存的，所以性能可控。有了`just-in-time`, 目前应用首次启动、系统版本更新、普通启动，`dyld4` 则可以根据缓存是否有效选择合适的模式进行解析。

- 装载内容，遍历所有dylibs

- 插入缓存

- 运行初始化方法 runAllInitializersForMain，notifyObjCInit 函数执，行完初始化之后会执行`notifyObjCInit`, 告诉objc 去运行所有 `+load` 方法, 而此时系统`main`还没有执行, 这也就是为什么`+ load`方法执行在`main`前面的原因。由`load_images` → `call_load_methods`→ `call_class_loads`内部也可以看出会 循环调用所有`+load` 方法，直到不再有。

- link动态库和主程序

- 加载主程序入口

  ![5954916-f66c63fdc4de5424](/Users/moon/Documents/summary/iOS/Assets/Dyld/5954916-f66c63fdc4de5424.webp)

  

[iOS底层探究-----dyld程序启动加载流程](https://juejin.cn/post/6983693445969739807)





