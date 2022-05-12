# iOS 事件传递和处理

## 什么是事件？

这里讲的事件是**用户交互的抽象**，像IOHIDEvent和UIEvent都是不同处理阶段的封装。

IOHIDEvent是iOS系统对事件的封装，感兴趣可以看源码[IOHIDEvent.h](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Fsource%2FIOHIDFamily%2FIOHIDFamily-308%2FIOHIDFamily%2FIOHIDEvent.h.auto.html)和[IOHIDEvent.cpp](https://links.jianshu.com/go?to=https%3A%2F%2Fopensource.apple.com%2Fsource%2FIOHIDFamily%2FIOHIDFamily-421.6%2FIOHIDFamily%2FIOHIDEvent.cpp.auto.html)（HID是**Human Interface Device**的缩写）。UIEvent是UIKit封装的描述用户操作类型的对象，可能有touch事件、motion事件、remote-control事件、press事件等。不同事件在响应链中处理方式不同，这里我们主要分析touch事件的传递和处理。

###  事件分类

对于 iOS 设备用户来说，他们操作设备的方式主要有三种：触摸屏幕、晃动设备、通过遥控设施控制设备。对应的事件类型有以下三种：

1. 触屏事件（Touch Event）
2. 运动事件（Motion Event）
3. 远端控制事件（Remote-Control Event）



### 响应者链

当发生事件响应时，必须知道由谁来响应事件。在 iOS 中，由响应者链来对事件进行响应。

所有事件响应的类都是 UIResponder 的子类，响应者链是一个由不同对象组成的层次结构，其中的每个对象将依次获得响应事件消息的机会。当发生事件时，事件首先被发送给第一响应者，第一响应者往往是事件发生的视图，也就是用户触摸屏幕的地方。事件将沿着响应者链一直向下传递，直到被接受并做出处理。一般来说，第一响应者是个视图对象或者其子类对象，当其被触摸后事件被交由它处理，如果它不处理，事件就会被传递给它的视图控制器对象 ViewController（如果存在），然后是它的父视图（superview）对象（如果存在），以此类推，直到顶层视图。接下来会沿着顶层视图（top view）到窗口（UIWindow 对象）再到程序（UIApplication 对象）。如果整个过程都没有响应这个事件，该事件就被丢弃。一般情况下，在响应者链中只要由对象处理事件，事件就停止传递。

一个典型的事件响应路线如下：

```
First Responser --> The Window --> The Application --> nil（丢弃）
```

我们可以通过 `[responder nextResponder]` 找到当前 responder 的下一个 responder，持续这个过程到最后会找到 UIApplication 对象。

通常情况下，我们在 First Responder （一般也就是用户当前触控的 View ）这里就会响应请求，进入下面的事件分发机制。



## 用户点击手机屏幕的过程

**App外：用户点击->硬件响应->参数量化->数据转发->App接收。**

在用户触摸屏幕之后，屏幕硬件会接受用户的操作，并采集关键的参数传递给IOKit，而IOKit将这些数据打包并传给[SpringBoard.app](https://links.jianshu.com/go?to=https%3A%2F%2Fiphonedevwiki.net%2Findex.php%2FSpringBoard.app)，继而转发给前台App。

**App内：子线程接收事件->主线程封装事件->UIWindow启动hitTest确定目标视图->UIApplication开始发送事件->touch事件开始回调。**

App启动时便会启动一个com.apple.uikit.eventfetch-thread子线程，负责接收SpringBoard.app转发过来的数据（通过runloop监听source1，查看堆栈中有__CFRunLoopDoSource1），数据会被封装成IOHIDEvent对象，然后转发给主线程；

![img](https:////upload-images.jianshu.io/upload_images/1049769-b5a78268ef6503b7?imageMogr2/auto-orient/strip|imageView2/2/w/554)

主线程同样在启动时监听source0，接收eventfetch-thread线程发送的IOHIDEvent数据，再封装成UIEvent，根据UIEvent的类型判断是否需要启动hitTest。motion事件不需要hitTest，touch事件也有部分不需要hitTest，比如说touch结束触发的事件。

![img](https:////upload-images.jianshu.io/upload_images/1049769-1debaee5a14524e0?imageMogr2/auto-orient/strip|imageView2/2/w/780)

确定目标视图之后，UIApplication便会发送事件，将UITouch和UIEvent发送给目标视图，触发其touches系列的方法。

## UIKit寻找目标视图的过程

寻找的过程主要依赖两个UIView的方法：-hitTest:withEvent方法和-pointInsdie:withEvent方法。

```objectivec
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event
```

hitTest方法返回point和event对应的视图；

```objectivec
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event
```

pointInside方法返回point和event是否在自己当前视图上；

这两个方法UIView都提供了默认实现，hitTest方法默认会调用所有子视图的hitTest方法，如果有一个返回。

UIKit会从UIWindow开始寻找目标视图，先调用UIWindow的hitTest方法询问是否有响应的视图，hitTest方法首先会先调用UIWindow的pointInside方法询问是否在点击范围内。

a.如果pointInside方法返回NO，则证明UIWindow无法响应该事件，hitTest方法会马上返回nil；
b.如果pointInside方法返回YES，则证明UIWindow可以响应该事件，hitTest方法会接着调用UIWindow子视图的hitTest方法。

- b1.如果子视图hitTest方法如果有返回视图，则UIWindow的hitTest方法会返回该视图；
- b2.如果所有子视图hitTest方法都没有返回视图，则UIWindow的hitTest方法会返回自己。

UIWindow是UIView的子类，UIView的hitTest方法实现和上述过程一致。

**思考：**
UIView在调用子视图hitTest时，是先调用哪些子视图？

> 从subview数组的末尾开始调用hitTest，subview数组下标越小，视图层级越低。

## UIKit确定目标视图后的过程

当UIKit确定目标视图之后，就会创建UITouch，UITouch的window属性和view属性就是上面过程中的UIWindow和目标视图。

接着UIApplication就会调用sendEvent:方法，接着UIWindow在sendEvent:方法中会调用sendTouchesForEvent:方法，如下图：

![img](https:////upload-images.jianshu.io/upload_images/1049769-b9636578268775d7?imageMogr2/auto-orient/strip|imageView2/2/w/1088)

UIWindow的sendTouchesForEvent:方法调用的是我们熟悉的touches四大方法：
`-touchesBegan:withEvent:`
`-touchesMoved:withEvent:`
`-touchesEnded:withEvent:`
`-touchesCancelled:withEvent:`
从上一步寻找到的目标视图开始，目标视图会首先被调用touches方法，接着是目标视图的父视图，再是父视图的父视图，如果某个视图是ViewController的.view属性，还会调用ViewController的方法，直到UIWindow、UIApplication、UIApplicationDelegate（我们创建的AppDelegate）。

下面是官方文档给出的回调顺序：（Responder chains in an app）

![img](https:////upload-images.jianshu.io/upload_images/1049769-61645ab99ae81555?imageMogr2/auto-orient/strip|imageView2/2/w/1200)



# 手势处理发生在哪一步

手势（UIGestureRecognizer）是iPhone的重要交互方式，[手势识别](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.apple.com%2Fdocumentation%2Fuikit%2Ftouches_presses_and_gestures%2Fimplementing_a_custom_gesture_recognizer%2Fabout_the_gesture_recognizer_state_machine) 介绍了手势是如何识别，甚至可以添加自定义手势。

UIGestureRecognizer同样有touches系列方法：

![img](https:////upload-images.jianshu.io/upload_images/1049769-63052f662e13045d?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

手势处理的发生时机我们可以通过手势的touchesBegan:withEvent:方法来看，当我们断点在手势的touchesBegan方法时，我们看到堆栈：

![img](https:////upload-images.jianshu.io/upload_images/1049769-e592a659ab073f29?imageMogr2/auto-orient/strip|imageView2/2/w/1054)

注意到堆栈中的UIApplication的sendEvent:方法，**sendEvent是发生在UIKit寻找目标视图过程之后**。从另外一种角度来思考，**touchesBegan方法中会用到UITouch，而UITouch中的view属性是目标视图**，所以手势的处理应该也放在UIKit寻找目标视图之后。

当手势的touchesBegan:withEvent:处理完成之后，便会触发目标视图的touchesBegan方法。

但是**当手势识别成功之后，默认会cancel后续touch操作**，从目标视图开始的响应链都会收到touchesCancelled方法，而不是正常的touchesEnded方法，堆栈如下：

![img](https:////upload-images.jianshu.io/upload_images/1049769-dfb5faf3de598de6?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

这个行为也可以通过设置下面的cancelsTouchesInView=NO来避免触发touchesCancelled方法。

![img](https:////upload-images.jianshu.io/upload_images/1049769-aab4af4328b02e08?imageMogr2/auto-orient/strip|imageView2/2/w/1200)

注意到不管是手势处理开始的touchesBegan方法，还是手势识别成功后触发touchesCancelled方法，堆栈中都有一个UIGestureEnvironment类。这是一个UIKit的私有类，在网上搜到[相关代码](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FJaviSoto%2FiOS10-Runtime-Headers%2Fblob%2Fmaster%2FFrameworks%2FUIKit.framework%2FUIGestureEnvironment.h)介绍：



```objectivec
@interface UIGestureEnvironment : NSObject {
    NSMutableArray * _delayedPresses;
    NSMutableArray * _delayedPressesToSend;
    NSMutableArray * _delayedTouches;
    NSMutableArray * _delayedTouchesToSend;
    UIGestureGraph * _dependencyGraph;
    NSMutableArray * _dirtyGestureRecognizers;
    bool  _dirtyGestureRecognizersUnsorted;
    struct __CFRunLoopObserver { } * _gestureEnvironmentUpdateObserver;
    NSMutableSet * _gestureRecognizersNeedingRemoval;
    NSMutableSet * _gestureRecognizersNeedingReset;
    NSMutableSet * _gestureRecognizersNeedingUpdate;
    NSMapTable * _nodesByGestureRecognizer;
    bool  _updateExclusivity;
}

- (void)addGestureRecognizer:(id)arg1;
- (void)addRequirementForGestureRecognizer:(id)arg1 requiringGestureRecognizerToFail:(id)arg2;
- (bool)gestureRecognizer:(id)arg1 requiresGestureRecognizerToFail:(id)arg2;
- (id)init;
- (void)removeGestureRecognizer:(id)arg1;
...
```

从头文件的方法声明，我们可以大概知道这是一个手势管理类，手势的添加、移除、响应都在内部完成。

**思考:**

1、UIButton的点击回调是怎么实现的？
2、如果给UIButton添加Tap手势，点击UIButton的时候是触发UIButton的Tap手势，还是触发UIButton的点击回调？

# 总结

所以综上三步，我们可以知道整个流程大概是：

1. 寻找目标视图：UIApplication->UIWindow->ViewController->View->targetView
2. 手势识别：UIGestureEnvironment-> UIGestureRecognizer
3. 响应链回调：targetView->Viewd->ViewController->UIWindow->UIApplication

iOS的用户交互相关非常复杂。由于时间有限，这里仅仅从事件的传递和处理出发，来建立一个基础的认知。



## 参考文献

手势识别 [https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/implementing_a_custom_gesture_recognizer/about_the_gesture_recognizer_state_machine](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.apple.com%2Fdocumentation%2Fuikit%2Ftouches_presses_and_gestures%2Fimplementing_a_custom_gesture_recognizer%2Fabout_the_gesture_recognizer_state_machine)

响应链介绍 [https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/using_responders_and_the_responder_chain_to_handle_events?from=from_parent_mindnote](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.apple.com%2Fdocumentation%2Fuikit%2Ftouches_presses_and_gestures%2Fusing_responders_and_the_responder_chain_to_handle_events%3Ffrom%3Dfrom_parent_mindnote)

## 思考题

1、UIButton的点击回调是怎么实现的？

> UIButton是UIControl的子类，通过追踪touch事件的变化得到一些UIControl定义的事件（UIControlEvents）；UIButton的点击操作是通过UIControlEvents的事件变化回调来触发，本质依赖的是响应链回调过程中的touches系列方法。

2、如果给UIButton添加Tap手势，点击UIButton的时候是触发UIButton的Tap手势，还是触发UIButton的点击回调？

> 上文分析了手势的识别是发生在响应链回调之前，也就是tap手势是发生在touches系列方法回调之前，那么Tap手势应该是在UIButton的touches方法之前。如果UIButton监听的是常用的UIControlEventTouchUpInside事件，则不会回调；如果监听的是UIControlEventTouchCancel事件，则在触发完Tap手势之后，还会收到回调。



https://juejin.cn/post/6844903905080410125
