# iOS逆向 -- 反调试 和 反反调试



### 一 ptrace 作用

ptrace系统调从名字上看是用于进程跟踪的，它提供了父进程可以观察和控制其子进程执行的能力，并允许父进程检查和替换子进程的内核镜像(包 括寄存器)的值。其基本原理是: 当使用了ptrace跟踪后，所有发送给被跟踪的子进程的信号(除了SIGKILL)，都会被转发给父进程，而子进程则会被阻塞，这时子进程的状态就会被 系统标注为TASK_TRACED。而父进程收到信号后，就可以对停止下来的子进程进行检查和修改，然后让子进程继续运行 ，因而可以实现断点调试和系统调用的跟踪

使用ptrace，你可以在用户层 【拦截和修改】系统调用(sys call)

注意：被跟踪的程序在进入或者退出某次系统调用的时候都会触发一个SIGTRAP信号，而被父进程捕获

Ptrace其原型为：

# include <sys/ptrace.h>

long ptrace(enum __ptrace_request request, pid_t pid, void *addr, void *data);

ptrace有四个参数:

1). enum __ptrace_request request：指示了ptrace要执行的命令。

2). pid_t pid: 指示ptrace要跟踪的进程。

3). void *addr: 指示要监控的内存地址。

4). void *data: 存放读取出的或者要写入的数据。

ptrace是如此的强大，以至于有很多大家所常用的工具都基于ptrace来实现，如strace和gdb

反调试举例说明：

比如： ptrace的命令PT_DENY_ATTACH 是苹果增加的一个 ptrace 选项，用于阻止 GDB 等调试器依附到某进程，用法如下：

```cpp
ptrace(PT_DENY_ATTACH, 0, 0, 0);

void anti_gdb_debug() {
    void *handle = dlopen(0, RTLD_GLOBAL | RTLD_NOW);
    ptrace_ptr_t ptrace_ptr = dlsym(handle, "ptrace");
    ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);
    dlclose(handle);
}
```

总结一下：ptrace被广泛用于反调试,因为一个进程只能被ptrace一次,如果事先调用了ptrace方法,那就可以防止别人调试我们的程序.

```undefined
也就是说谁先调用ptrace 谁说了算，如果我们直接在app 中写ptrace ， 那么调试的时候肯定就无法调试，因为被应用内的ptrace 抢占了
```

`反反调试： 如果别人的的app进行了ptrace防护，那么你怎么让他的ptrace不起作用，进而调试其他的app。由于ptrace是系统函数，那么我们可以用fishhook来hook住ptrace函数，然后让他的app调用我们自己的ptrace函数，即写动态库，如果多个动态库hook 了ptrace ,我们可以调整 Link Binary Libraries的顺序加载，假设人家应用自己写的hook ptrace动态库肯定会在自己前面，最后的方式我们可以通过修改macho的二进制让他的ptrace失效【不去执行ptrace】，然后进行调试.`
最后：在给一种反调试的方案，这种也只能无法断点调试ptrace 函数

我不想暴露自己的ptrace等系统方法，不想被符号断点断住，可以采用汇编进行调用ptrace

![img](https://upload-images.jianshu.io/upload_images/1974361-74285fd7ec13dbb0.png?imageMogr2/auto-orient/strip|imageView2/2/w/848/format/webp)

这样人家就很难通过断点的方式去调试

### 调试器建立调试关系的两种方式:

用gdb调试程序[调试程序也在一个进程里]，可以直接gdb ./test,也可以gdb (test的进程号)。这对应着使用ptrace建立跟踪关系的两种方式:

- fork:利用fork+execve执行被测试的程序，子进程在执行execve之前调用ptrace(PTRACE_TRACEME)，建立了与父进程(debugger 调试程序进程)的跟踪关系。
- attach: debugger可以调用ptrace(PTRACE_ATTACH，pid,…)，建立自己与进程号为pid的进程间的跟踪关系。即利用PTRACE_ATTACH，使自己变成被调试程序的父进程(用ps可以看到)。用attach建立起来的跟踪关系，可以调用ptrace(PTRACE_DETACH，pid,…)来解除。注意attach进程时的权限问题，如一个非root权限的进程是不能attach到一个root进程上的。

第一种方式的例子：

ptrace提供了对子进程进行单步的功能，ptrace(PTRACE_SINGLESTEP, …) 会使内核在子进程的每一条指令执行前先将其阻塞，然后将控制权交给父进程

而父进程此时会使用 wait函数等待阻塞信号，然后判断status变量来检查子进程是被ptrace暂停掉还是已经运行结束并退出，如果状态是ptrace暂停的，则可以获取子进程的寄存器器状态，

ptrace(PTRACE_GETREGS,child, NULL, ®s)，获取当前指令等，在让ptrace控制单步执行
ptrace(PTRACE_SINGLESTEP, child,NULL, NULL);

每一步都去唤醒子进程继续执行，并告诉内核在执行一条指令后就将其阻塞

最后让子进程恢复

PTRACE_SYSCALL：继续，但在下一个系统调用入口或出口处停止。

### 二： sysctl 作用

sysctl命令被用于在内核运行时动态地修改内核的运行参数

函数原型

int sysctl (int *name, int nlen, void *oldval, size_t *oldlenp, void *newval, size_t newlen);

Name /* 整形数组，每个数组元素代表系统参数存取路径上的一个文件或目录名，例如/proc/sys/kernel用CTL_KERN表示*/

oldval /* 当读取系统参数时，用于存取系统参数值，也就是/proc/sys/下的某个文件内容*/

Newval /* 当写系统参数时，记录所要写入的新值*/

反调试举例：

当一个进程被调试的时候，该进程会有一个标记来标记自己正在被调试，所以可以通过sysctl去查看当前进程的信息，看有没有这个标记位即可检查当前调试状态。

![img](https://upload-images.jianshu.io/upload_images/1974361-1fbeb7273c367761.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

检测到调试器就退出，或者制造崩溃，或者隐藏工程啥的，当然也可以定时去查看有没有这个标记

### 三： syscall 作用

为从实现从用户态切换到内核态，系统提供了一个系统调用函数syscall ，所有的系统调用都可以通过syscall 去实现

比如： syscall (26,31,0,0) 来调用系统函数ptrace，ptrace的系统调用函数号是26

syscall是通过软中断来实现从用户态到内核态，也可以通过汇编svc调用来实现。

![img](https://upload-images.jianshu.io/upload_images/1974361-70dc08547fe9cf31.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

比如：arm 32位 #80 就是软中断值，r12 存放系统函数编号，arm 64位 #128 是系统中断码，x0存放系统函数编号

### 四： 重签名防护

想自己的app不被重签名，可以在代码中检测签名信息

查看证书的application-identifier 查看embedded.mobileprovision信息security cms -D -i embedded.mobileprovision 找到<key>application-identifier</key>的value的第一部分就是

在执行代码的时候检查签名是否和我们已知的签名对比

```objectivec
void checkCodesign(NSString *id){
  // 描述文件路径
  NSString *embeddedPath = [[NSBundle mainBundle] pathForResource:@"embedded" ofType:@"mobileprovision"];
  // 读取application-identifier 注意描述文件的编码要使用:NSASCIIStringEncoding
  NSString *embeddedProvisioning = [NSString stringWithContentsOfFile:embeddedPath encoding:NSASCIIStringEncoding error:nil];
  NSArray *embeddedProvisioningLines = [embeddedProvisioning componentsSeparatedByCharactersInSet:[NSCharacterSet newlineCharacterSet]];

  for (int i = 0; i < embeddedProvisioningLines.count; i++) {
      if ([embeddedProvisioningLines[i] rangeOfString:@"application-identifier"].location != NSNotFound) {

      NSInteger fromPosition = [embeddedProvisioningLines[i+1] rangeOfString:@"<string>"].location+8;
      NSInteger toPosition = [embeddedProvisioningLines[i+1] rangeOfString:@"</string>"].location;

      NSRange range;
      range.location = fromPosition;
      range.length = toPosition - fromPosition;

      NSString *fullIdentifier = [embeddedProvisioningLines[i+1] substringWithRange:range];
      NSArray *identifierComponents = [fullIdentifier componentsSeparatedByString:@"."];
      NSString *appIdentifier = [identifierComponents firstObject];

      // 对比签名ID
      if (![appIdentifier isEqual:id]) {
          //exit
          asm(
           "mov X0,#0\n"
           "mov w16,#1\n"
           "svc #0x80"
           );
      }
      break;
     }
  }
}
```

### 五：反反调试

这里主要针对ptrace、sysctl、syscall来反反调试，做法就很简单了，hook函数

比如：

1： hook ptrace 函数， 遇到request = 31, 就知道程序进行了反调试，所以我们可以将request 改掉

![img](https://upload-images.jianshu.io/upload_images/1974361-22bdca15741c42b1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1048/format/webp)

2：hook sysctl , 看其是否在检查进程被追踪的这个标记TASK_TRACED，我们将其返回信息info_ptr -> kp_proc.p_flag 改掉,让其检查的结果是没有设置追踪标识

![img](https://upload-images.jianshu.io/upload_images/1974361-7dcad82f9ad35a4c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

3 hook syscall , 防止ptrace 是通过syscall 的方式去调用的，

![img](https://upload-images.jianshu.io/upload_images/1974361-c764bca26b667617.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

4：hook dlsym 防止通过这种方式去调用ptrace 函数

![img](https://upload-images.jianshu.io/upload_images/1974361-e8c658b3107e655a.png?imageMogr2/auto-orient/strip|imageView2/2/w/920/format/webp)

5 初始化函数

![img](https://upload-images.jianshu.io/upload_images/1974361-14748491490919a0.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

或者使用fishhook

![img](https://upload-images.jianshu.io/upload_images/1974361-0cc9597b1a2402e9.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

lldb 反反调试的

通过lldb下断点，然后修改参数，或者直接返回也可以达到反反调试的效果

为了方便直接使用facebook的chisel来增加脚本。

![img](https://upload-images.jianshu.io/upload_images/1974361-201fff63163ccf16.png?imageMogr2/auto-orient/strip|imageView2/2/w/1036/format/webp)



![img](https://upload-images.jianshu.io/upload_images/1974361-8961eb261d4c3dde.png?imageMogr2/auto-orient/strip|imageView2/2/w/996/format/webp)



![img](https://upload-images.jianshu.io/upload_images/1974361-e1a5cb69b2c62d30.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

当遇到情况为$x0 == 31 时发生回掉，将x0 或者r0 的值改成0

![img](https://upload-images.jianshu.io/upload_images/1974361-d5882f4353234291.png?imageMogr2/auto-orient/strip|imageView2/2/w/816/format/webp)

### 六： App的防护

1. 首先加强现有的密码检测机制的监测力度,除了密码长度,数字、字符甚至特殊符号的混杂程度,本文着重对易受攻击的键盘上特定组合码进行检测;
2. 其次完善iOS内存保护机制,利用Objective-C对象实现内存安全擦除,保证及时对文件数据的每个字节都做到全覆盖,防止对象被跟踪后信息遭到泄露;

3. 再次在维护程序在运行时的安全性上提出了一种贯穿程序被调试的三个阶段的反调试机制:从程序开始被调试、继续被跟踪到最终被恶意修改均进行跟踪测试,最终阻止被恶意修改的目标继续的执行


针对上述安全隐患，我们的iOS应用安全防护框架需实现的任务大致如下：

- 防护
  - ObjC类名方法名等重命名为难以理解的字符
  - 加密静态字符串运行时解密
  - 混淆代码使其难于反汇编
  - 本地存储文件防篡改
- 检测
  - 调试状态检测 ： 反调试 ptrace . sysctl
  - 越狱环境检测
  - ObjC的Swizzle检测
  - 任意函数的hook检测
  - 指定区域或数据段的校验和检测
- 自修复
  - 自修复被篡改的数据和代码段

此外，还需要多层的防护，通过高层保护低层的方式来保证整个防护机制不失效。 参考IBM移动终端安全防护框架解决方案:

1：越狱检测的方法：

```objectivec
1》使用NSFileManager判断设备是否安装了如下越狱常用工具
   /Applications/Cydia.app
   /Library/MobileSubstrate/MobileSubstrate.dylib
   /bin/bash
   /usr/sbin/sshd
   /etc/apt

  这种方式不要写成Bool方式去检查 ，容易被攻击者hook
  注意： 攻击者可能会改变这些工具的安装路径，躲过你的判断。

2》可以尝试打开cydia应用注册的URL scheme，后面应该是你知道某个应用URL scheme
    if([[UIApplication sharedApplication] canOpenURL:[NSURL URLWithString:@"[cydia://package/com.example.package](cydia://package/com.example.package)"]]){
        NSLog(@"Device is jailbroken");
    }
   但是不是所有的工具都会注册URL scheme，而且攻击者可以修改任何应用的URL scheme。

3》你可以尝试读取下应用列表，看看有无权限获取：
  攻击者可能会hook NSFileManager 的方法，让你的想法不能如愿
```

![img](https://upload-images.jianshu.io/upload_images/1974361-316cc3dcff074640.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

```objectivec
     4》你可以回避 NSFileManager，使用stat系列函数检测Cydia等工具：
```

![image.png](https://upload-images.jianshu.io/upload_images/1974361-cfbf09cba56d9648.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

攻击者可能会利用 [Fishhook原理](http://blog.csdn.net/yiyaaixuexi/article/details/19094765) hook了stat

```bash
   5》你可以看看stat是不是出自系统库，有没有被攻击者换掉
```

![img](https://upload-images.jianshu.io/upload_images/1974361-07437122a6140bf6.png?imageMogr2/auto-orient/strip|imageView2/2/w/1064/format/webp)

使用dladdr方法可以获得一个函数所在的模块.从而判断该函数是否被替换掉

如果结果不是 /usr/lib/system/libsystem_kernel.dylib 的话，那就100%被攻击了。

如果 libsystem_kernel.dylib 都是被攻击者替换掉的…

也可以判断ios 的方法在什么库中，通过该方法验证指定类的方法是否都来自指定模块

![img](https://upload-images.jianshu.io/upload_images/1974361-01d940ae36fab1f8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

建议使用inline方式编译,像这样以内联函数的形式编译,攻击者必须修改每一处调用该函数的的地方

检查所有方法判断是否来之某个模块

```undefined
    6》检索一下自己的应用程序是否被链接了异常动态库，列出所有已链接的动态库：
```

![img](https://upload-images.jianshu.io/upload_images/1974361-c03324ee1d42de4e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

通常情况下，会包含越狱机的输出结果会包含字符串： Library/MobileSubstrate/MobileSubstrate.dylib

攻击者可能会给MobileSubstrate改名，但是原理都是通过DYLD_INSERT_LIBRARIES注入动态库

```undefined
       7》可以通过检测当前程序运行的环境变量：
```

![img](https://upload-images.jianshu.io/upload_images/1974361-8c1395e6717020ff.png?imageMogr2/auto-orient/strip|imageView2/2/w/866/format/webp)

image.png

未越狱设备返回结果是null，越狱设备就各有各的精彩了，尤其是老一点的iOS版本越狱环境

上述越狱检查总结如下：

- 不要用NSFileManager，这是最容易被hook掉的。
- 检测方法中所用到的函数尽可能用底层的C，如文件检测用stat函数(iPod7.0，越狱机检测越狱常见的会安装的文件只能检测到此步骤，下面的检测不出来)
- 再进一步，就是检测stat是否出自系统库
- 再进一步，就是检测链接动态库(尽量不要，appStore可能审核不过)
- 再进一步，检测程序运行的环境变量

即使这样还是不能完全检查

比如： 用户可能安装越狱检测绕过插件（xCon），对于越狱检测，很大程度上都还是针对某些目录下某个文件名字是否换了或者文件被替换了等等去检测；

```objectivec
- (BOOL)mgjpf_isJailbroken {
   //以下检测的过程是越往下，越狱越高级
   ///Applications/Cydia.app, /privte/var/stash

    BOOL jailbroken = NO;
    NSString *cydiaPath = @"/Applications/Cydia.app";
    NSString *aptPath = @"/private/var/lib/apt/";

    if ([[NSFileManager defaultManager] fileExistsAtPath:cydiaPath]) {
        jailbroken = YES;
    }

    if ([[NSFileManager defaultManager] fileExistsAtPath:aptPath]) {
        jailbroken = YES;
    }

    //可能存在hook了NSFileManager方法，此处用底层C stat去检测

    struct stat stat_info;

    if (0 == stat("/Library/MobileSubstrate/MobileSubstrate.dylib", &stat_info)) {
        jailbroken = YES;
    }

    if (0 == stat("/Applications/Cydia.app", &stat_info)) {
        jailbroken = YES;
    }

    if (0 == stat("/var/lib/cydia/", &stat_info)) {
        jailbroken = YES;
    }

    if (0 == stat("/var/cache/apt", &stat_info)) {
        jailbroken = YES;
    }

//    /Library/MobileSubstrate/MobileSubstrate.dylib 最重要的越狱文件，几乎所有的越狱机都会安装MobileSubstrate
//    /Applications/Cydia.app/ /var/lib/cydia/绝大多数越狱机都会安装
//    /var/cache/apt /var/lib/apt /etc/apt
//    /bin/bash /bin/sh
//    /usr/sbin/sshd /usr/libexec/ssh-keysign /etc/ssh/sshd_config

    //可能存在stat也被hook了，可以看stat是不是出自系统库，有没有被攻击者换掉
    //这种情况出现的可能性很小
    int ret;
    Dl_info dylib_info;
    int (*func_stat)(const char *,struct stat *) = stat;

    if ((ret = dladdr(func_stat, &dylib_info))) {
        NSLog(@"lib:%s",dylib_info.dli_fname);      //如果不是系统库，肯定被攻击了
        if (strcmp(dylib_info.dli_fname, "/usr/lib/system/libsystem_kernel.dylib")) {   //不相等，肯定被攻击了，相等为0
            jailbroken = YES;
        }
    }

    //还可以检测链接动态库，看下是否被链接了异常动态库，但是此方法存在appStore审核不通过的情况，这里不作罗列
    //通常，越狱机的输出结果会包含字符串： Library/MobileSubstrate/MobileSubstrate.dylib——之所以用检测链接动态库的方法，是可能存在前面的方法被hook的情况。这个字符串，前面的stat已经做了
    //如果攻击者给MobileSubstrate改名，但是原理都是通过DYLD_INSERT_LIBRARIES注入动态库
    //那么可以，检测当前程序运行的环境变量

    char *env = getenv("DYLD_INSERT_LIBRARIES");
    if (env != NULL) {
        jailbroken = YES;
    }
    return jailbroken;
}
```

## 2. 全工程加固。

1. 支持混淆。
2. 脏代码插入。

基于LLVM+Obfuscation的修改方案。
源码地址：
https://github.com/miniwing/ollvm-12.x.git 
分支：
ollvm-12.x

使用方法:

1. 下载代码:
   git clone https://github.com/miniwing/ollvm-12.x.git xxx
2. cd xxx
3. mkdir build
4. cd build
5. cmake -DXXXX1 -DXXX2 -DLLVM_CREATE_XCODE_TOOLCHAIN=ON ../build
   上面的XXX模块根据自己的需要进行配置。
   LLVM_CREATE_XCODE_TOOLCHAIN=ON 是配置生成xCode toolchain.
6. make -jn （根据自己的电脑配置编译线程数）。
7. sudo make install-xcode-toolchain
8. 移动生成的toolchain 到 xcode的对应目录。

编译环境:

cmake版本

```javascript
cmake -version
cmake version 3.21.1
```

clang版本

```javascript
Apple clang version 12.0.5 (clang-1205.0.22.11)
Target: x86_64-apple-darwin20.6.0
Thread model: posix
```

编译之后clang版本

```javascript
clang version 12.0.1 (Obfuscation LLVM --Powered by MINIWING @Harry)
Target: x86_64-apple-darwin20.6.0
Thread model: posix
```

风险：

1. 混淆力度对上架审核有风险。
2. toolchain中clang版本xcode 自带的clang对应。大版本对应相关比如12.1, 12.2 属于12.x
3. android的toolcnain要对ndk中的 toolchain同样需要对应。

[ios逆向](https://www.jianshu.com/nb/31867173)

[Obfuscator-llvm源码分析](https://zhuanlan.zhihu.com/p/39479793)



