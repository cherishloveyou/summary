# Method swizzling

**`Method Swizzling`本质上就是对方法的`IMP`和`SEL`进行交换**

在Objective-C的运行时中，一个类是用一个名为objc_class的结构体表示的，从结构体中可以发现一个objc_method_list指针，它保存着当前类的所有方法列表。同时，objc_method_list也是一个结构体,结构体里发现了一个objc_method字段.

```objective-c
// objc_method 结构体
typedef struct objc_method *Method;

struct objc_method {
    SEL _Nonnull method_name;                    // 方法名
    char * _Nullable method_types;               // 方法类型
    IMP _Nonnull method_imp;                     // 方法实现
};
```

使用Method Swizzling交换方法，其实就是修改了objc_method结构体中的method_imp，也就是说改变了method_name和method_imp之间的映射关系。

##  Method Swizzling 方案 A

> 在该类的分类中添加 Method Swizzling 交换方法，用普通方式

这种方式在开发中应用最多的。

```objc
@implementation UIViewController (Swizzling)

// 交换 原方法 和 替换方法 的方法实现
+ (void)load {
    
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 当前类
        Class class = [self class];
        
        // 原方法名 和 替换方法名
        SEL originalSelector = @selector(originalFunction);
        SEL swizzledSelector = @selector(swizzledFunction);
        
        // 原方法结构体 和 替换方法结构体
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        
        /* 如果当前类没有 原方法的 IMP，说明在从父类继承过来的方法实现，
         * 需要在当前类中添加一个 originalSelector 方法，
         * 但是用 替换方法 swizzledMethod 去实现它 
         */
        BOOL didAddMethod = class_addMethod(class,
                                            originalSelector,
                                            method_getImplementation(swizzledMethod),
                                            method_getTypeEncoding(swizzledMethod));
        
        if (didAddMethod) {
            // 原方法的 IMP 添加成功后，修改 替换方法的 IMP 为 原始方法的 IMP
            class_replaceMethod(class,
                                swizzledSelector,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            // 添加失败（说明已包含原方法的 IMP），调用交换两个方法的实现
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

// 原始方法
- (void)originalFunction {
    NSLog(@"originalFunction");
}

// 替换方法
- (void)swizzledFunction {
    NSLog(@"swizzledFunction");
}

@end
```

##  Method Swizzling 方案 B

> 在该类的分类中添加 Method Swizzling 交换方法，但是使用函数指针的方式。

方案 B 和方案 A 的最大不同之处在于使用了函数指针的方式，使用函数指针最大的好处是可以有效避免命名错误。

```objc
#import "UIViewController+PointerSwizzling.h"
#import <objc/runtime.h>

typedef IMP *IMPPointer;

// 交换方法函数
static void MethodSwizzle(id self, SEL _cmd, id arg1);
// 原始方法函数指针
static void (*MethodOriginal)(id self, SEL _cmd, id arg1);

// 交换方法函数
static void MethodSwizzle(id self, SEL _cmd, id arg1) {
    
    // 在这里添加 交换方法的相关代码
    NSLog(@"swizzledFunc");
    
    MethodOriginal(self, _cmd, arg1);
}

BOOL class_swizzleMethodAndStore(Class class, SEL original, IMP replacement, IMPPointer store) {
    IMP imp = NULL;
    Method method = class_getInstanceMethod(class, original);
    if (method) {
        const char *type = method_getTypeEncoding(method);
        imp = class_replaceMethod(class, original, replacement, type);
        if (!imp) {
            imp = method_getImplementation(method);
        }
    }
    if (imp && store) { *store = imp; }
    return (imp != NULL);
}

@implementation UIViewController (PointerSwizzling)

+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [self swizzle:@selector(originalFunc) with:(IMP)MethodSwizzle store:(IMP *)&MethodOriginal];
    });
}

+ (BOOL)swizzle:(SEL)original with:(IMP)replacement store:(IMPPointer)store {
    return class_swizzleMethodAndStore(self, original, replacement, store);
}

// 原始方法
- (void)originalFunc {
    NSLog(@"originalFunc");
}

@end
```

##  Method Swizzling 方案 C

> 在其他类中添加 Method Swizzling 交换方法

这种情况一般用的不多，最出名的就是 AFNetworking 中的_AFURLSessionTaskSwizzling 私有类。_AFURLSessionTaskSwizzling 主要解决了 iOS7 和 iOS8 系统上 NSURLSession 差别的处理。让不同系统版本 NSURLSession 版本基本一致。



```objc
static inline void af_swizzleSelector(Class theClass, SEL originalSelector, SEL swizzledSelector) {
    Method originalMethod = class_getInstanceMethod(theClass, originalSelector);
    Method swizzledMethod = class_getInstanceMethod(theClass, swizzledSelector);
    method_exchangeImplementations(originalMethod, swizzledMethod);
}

static inline BOOL af_addMethod(Class theClass, SEL selector, Method method) {
    return class_addMethod(theClass, selector,  method_getImplementation(method),  method_getTypeEncoding(method));
}

@interface _AFURLSessionTaskSwizzling : NSObject

@end

@implementation _AFURLSessionTaskSwizzling

+ (void)load {
    if (NSClassFromString(@"NSURLSessionTask")) {
        
        NSURLSessionConfiguration *configuration = [NSURLSessionConfiguration ephemeralSessionConfiguration];
        NSURLSession * session = [NSURLSession sessionWithConfiguration:configuration];
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wnonnull"
        NSURLSessionDataTask *localDataTask = [session dataTaskWithURL:nil];
#pragma clang diagnostic pop
        IMP originalAFResumeIMP = method_getImplementation(class_getInstanceMethod([self class], @selector(af_resume)));
        Class currentClass = [localDataTask class];
        
        while (class_getInstanceMethod(currentClass, @selector(resume))) {
            Class superClass = [currentClass superclass];
            IMP classResumeIMP = method_getImplementation(class_getInstanceMethod(currentClass, @selector(resume)));
            IMP superclassResumeIMP = method_getImplementation(class_getInstanceMethod(superClass, @selector(resume)));
            if (classResumeIMP != superclassResumeIMP &&
                originalAFResumeIMP != classResumeIMP) {
                [self swizzleResumeAndSuspendMethodForClass:currentClass];
            }
            currentClass = [currentClass superclass];
        }
        
        [localDataTask cancel];
        [session finishTasksAndInvalidate];
    }
}

+ (void)swizzleResumeAndSuspendMethodForClass:(Class)theClass {
    Method afResumeMethod = class_getInstanceMethod(self, @selector(af_resume));
    Method afSuspendMethod = class_getInstanceMethod(self, @selector(af_suspend));

    if (af_addMethod(theClass, @selector(af_resume), afResumeMethod)) {
        af_swizzleSelector(theClass, @selector(resume), @selector(af_resume));
    }

    if (af_addMethod(theClass, @selector(af_suspend), afSuspendMethod)) {
        af_swizzleSelector(theClass, @selector(suspend), @selector(af_suspend));
    }
}

- (void)af_resume {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_resume];
    
    if (state != NSURLSessionTaskStateRunning) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidResumeNotification object:self];
    }
}

- (void)af_suspend {
    NSAssert([self respondsToSelector:@selector(state)], @"Does not respond to state");
    NSURLSessionTaskState state = [self state];
    [self af_suspend];
    
    if (state != NSURLSessionTaskStateSuspended) {
        [[NSNotificationCenter defaultCenter] postNotificationName:AFNSURLSessionTaskDidSuspendNotification object:self];
    }
}
```

## 2.5 Method Swizzling 方案 D

> 优秀的第三方框架：[JRSwizzle](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Frentzsch%2Fjrswizzle) 和 [RSSwizzle](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Frabovik%2FRSSwizzle)

JRSwizzle 和 RSSwizzle 都是优秀的封装 Method Swizzling 的第三方框架。

1. **JRSwizzle** 尝试解决在不同平台和系统版本上的 Method Swizzling 与类继承关系的冲突。对各平台低版本系统兼容性较强。JRSwizzle 核心是用到了 `method_exchangeImplementations` 方法。在健壮性上先做了 `class_addMethod` 操作。
2. **RSSwizzle** 主要用到了 `class_replaceMethod` 方法，避免了子类的替换影响了父类。而且对交换方法过程加了锁，增强了线程安全。它用很复杂的方式解决了 **[What are the dangers of method swizzling in Objective-C？](https://links.jianshu.com/go?to=https%3A%2F%2Fstackoverflow.com%2Fquestions%2F5339276%2Fwhat-are-the-dangers-of-method-swizzling-in-objective-c)** 中提到的问题。是一种更安全优雅的 Method Swizzling 解决方案。

------



# Method Swizzling 使用注意

##### 坑点1 ：method-swizzling 多次调用的混乱问题

1. 应该只在 `+load` 中执行 Method Swizzling。会先加载所有的类，这时会调用每个类的 `+load` 方法
2. Method Swizzling 在 `+load` 中执行时，不要调用 `[super load];`。如果在 `+ (void)load`方法中调用 `[super load]` 方法，就会导致父类的 `Method Swizzling` 被重复执行两次
3. Method Swizzling 应该总是在 `dispatch_once` 中执行。
4. 使用 Method Swizzling 后要记得调用原生方法的实现。

load方法会被系统主动调用多次，**解决方案**: 可以通过`单例`，使方法交换只执行一次.

```objective-c
+ (void)load{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        //TODO:这里进行你的方法交换
    });
}
```

因为 `+initialize` 方法的调用时机是在 第一次向该类发送第一个消息的时候才会被调用。如果该类只是引用，没有调用，则不会执行 `+initialize` 方法。
 Method Swizzling 影响的是全局状态，`+load` 方法能保证在加载类的时候就进行交换，保证交换结果。而使用 `+initialize` 方法则不能保证这一点，有可能在使用的时候起不到交换方法的作用。



`+ load` 方法的调用规则为：

1. 先调用主类，按照编译顺序，顺序地根据继承关系由父类向子类调用；

2. 再调用分类，按照编译顺序，依次调用；

3. `+ load` 方法除非主动调用，否则只会调用一次。

   

   这样的调用规则导致了 `+ load` 方法调用顺序并不一定确定。一个顺序可能是：`父类 -> 子类 -> 父类类别 -> 子类类别`，也可能是 `父类 -> 子类 -> 子类类别 -> 父类类别`。所以  Method Swizzling 的顺序不能保证，那么就不能保证  Method Swizzling 后方法的调用顺序是正确的。

   所以被用于 Method Swizzling 的方法必须是当前类自身的方法，如果把继承父类来的 IMP 复制到自身上面可能会存在问题。如果 `+ load` 方法调用顺序为：`父类 -> 子类 -> 父类类别 -> 子类类别`，那么造成的影响就是调用子类的替换方法并不能正确调起父类分类的替换方法。

   

##### 坑点2：子类无声明无实现，父类有声明有实现.

会导致崩溃，崩溃的根本原因是找不到方法的实现imp，**解决方案**: 在方法交换时，先判断交换的方法是否有实现imp，未实现则先实现。

##### 坑点3：父类有声明无实现，子类无声明无实现

递归崩溃，因为原始的方法没有imp



https://www.jianshu.com/p/1ab7e611107c

https://www.jianshu.com/p/5deb7bc97b7f
