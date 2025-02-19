# **Flutter性能优化实践 **

Flutter作为一款高性能、高质量的移动开发框架，在开发过程中仍然需要进行一些性能优化，以确保应用的流畅性和响应速度。以下是一些关键的Flutter性能优化策略：

#### 1、减少Widget重建

**使用const构造函数**：对于不会改变的Widget，使用const构造函数创建常量Widget，可以避免不必要的重建。
**合理使用Key**：在ListView或GridView等可滚动列表中，为**列表项指定Key**可以帮助Flutter识别哪些项发生了变化，从而只重建发生变化的项。
**减少刷新范围**：我们使用setState方法就可以轻松刷新页面，但是要尽力控制刷新范围，例如：倒计时功能，我们封装一个widget
**减少刷新次数：** 验证码输入6位数字，底部登陆按钮可用，我们不需要每次有文们输入的时候就刷新，给一个bool值判断
**避免在build方法中进行复杂操作**：build方法应该只负责构建UI，避免在其中进行复杂的计算或数据处理。
**错峰加载**：错峰加载的目的是为了避免因同一时间的大量构建，而产生卡顿现象

在使用PageView.builder这个Widget时，我发现在左右滑动切换页面时会有卡顿的现象，onPageChanged就会回调结果，触发了页面的刷新代码。我们可以添加一个通知，滚动结束的时候才加载新的页面

#### 2、避免不必要的UI重绘

- **使用shouldRepaint方法**：在CustomPainter中，通过覆写shouldRepaint方法来判断是否需要重绘，避免不必要的绘制操作。 

#### 3、优化图片加载

- **使用缓存技术**：对于需要重复加载的图片，使用缓存技术可以减少网络请求和加载时间。Flutter提供了ImageCache等机制来支持图片缓存。
- **图片通过阿里云处理**，下载小图

#### 4、使用异步操作

**使用Future和Stream**：在Flutter中，Future和Stream是处理异步操作的主要方式，它们可以帮助开发者更好地管理异步流程。
耗时计算：避免将一些耗时计算放在UI线程，我们可以把耗时计算放到Isolate去执行

#### flutter长列表加载大量网络图片，程序直接崩溃

1、图片通过阿里云处理，下载小图

2、图片尽可能使用缓存，这个可以使用CachedNetworkImage插件实现

4、运行过程中不变的组件使用const修饰（例如固定的icon)，这样可以复用组件，渲染效率更高。

5、要么手动删缓存，要么把缓存大小弄小一点，就不会内存溢出了`

```ini
代码imageCache.maximumSize = 10
```

6、设置ScrollAwareImageProvider：判断快速滑动, 下载和解码会停止

### Flutter 常用插件

1. 网络请求 [dio](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Fdio)
2. json解析 json_serializable
3. 持久化操作本地存储 [shared_preferences](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Fshared_preferences)
4. 路由管理 [fluro](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Ffluro)
5. 网络加载图片并缓存本地 [cached_network_image](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Fcached_network_image)
6. 屏幕适配 [flutter_screenutil](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Fflutter_screenutil)
7. 极光推送 [jpush_flutter](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Fjpush_flutter)
8. 全局状态管理 [provider](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Fprovider)
9. 轮播图 [flutter_swiper](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Fflutter_swiper)
10. 上拉刷新，下拉加载[pull_to_refresh](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Fpull_to_refresh)、[flutter_easyrefresh](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Fflutter_easyrefresh)
11. 高德地图SDK插件[amap_flutter_map](https://link.juejin.cn?target=https%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%3A%2F%2Fpub.flutter-io.cn%2Fpackages%2Famap_flutter_map)
12. 是一个能够快速便捷的为原生应用提供 Flutter 混合的集成方案 FlutterBoost
13. fluttertoast (toast)

### Flutter状态管理setState&Provider&Bloc区别

#### setState：

- 适用于简单的小规模应用或组件内部的状态管理。更新是局部的，只会触发当前组件及其子组件的重建。
- 当组件的状态更新不会涉及到**跨组件的通信时**，使用 `setState` 是最简单直接的方式。
- 刷新方式：`setState` 是通过直接修改**组件的状态**来触发 UI 的重建。

#### Provider：

- 适用于中大型应用或需要**跨组件共享状态**的情况。
- **通过监听机制实现状态的更新**，当状态变化时，所有依赖于该状态的组件都会收到通知并进行更新。`Provider` 是通过创建一个**共享的状态源**，并**监听该状态源的变化**来实现状态的共享和更新。
- 它是基于 `InheritedWidget` 实现的跨组件传值
- 提供了更多的灵活性和可扩展性，适合处理更复杂的状态管理需求

#### Bloc

- Bloc基于响应式编程原则，使用流（Stream）和流订阅（StreamSubscription）来管理状态。这使得Bloc能够自动响应状态的变化，并通知UI进行更新，从而提高了应用的响应性和用户体验。

#### Provider原理

Provider的作用主要有两个

- 1、跨组件传值：利用 Flutter 的 `InheritedWidget` 机制来实现跨组件
- 2、状态管理：利用`ChangeNotifier`实现状态刷新

- 3、对于实现了通知机制的状态对象（如 `ChangeNotifier`），你需要调用其通知方法（如 `notifyListeners()`）来告诉 Flutter 状态已经改变。`Provider`会监听这些变化，并**自动触发依赖于该状态的 widget 的重建。**

#### Provider对InheritedWidget有哪些优点

- 1、使用简单。模型类继承ChangeNotifier，没有更多的布局widget，只需要通过context.read/context.watch操作或者监听模型类即可；
- 2、**懒加载**：Provider支持懒加载机制，这意味着只有在真正需要某个资源时，它才会被加载和初始化。这有助于提升应用的性能，减少不必要的资源消耗和加载时间。
- 3、**优化监听架构的时间复杂度**：在监听状态变化时，Provider的监听架构时间复杂度以指数级增长（如ChangeNotifier，其复杂度为O(N)）。这意味着在大型应用中，Provider可以更有效地管理状态变化，减少不必要的计算和渲染。

#### 为什么使用Provider而不是Bloc（见仁见智）

**1、易用性和学习曲线**

- **Provider的易用性：Provider是一个轻量级的状态管理库，它提供了简单而直观的API**，使得状态管理变得相对容易。开发者可以轻松地共享和更新应用中的状态
- **Bloc的学习曲线相对较陡峭。它采用了响应式编程的思想，** **并将业务逻辑与UI分离，增加了理解和掌握的难度**。但随着应用规模的增加，管理多个Bloc和它们之间的交互可能会变得复杂。这需要开发者具备良好的组织能力和架构设计能力。

**2、性能考虑**

Provider的性能：Provider基于Flutter框架内置的InheritedWidget机制，能够在保持应用性能的同时提供简洁高效的状态管理功能。它不会给应用引入过多的复杂性和性能开销。

Bloc的性能：虽然Bloc在性能上通常也能满足需求，但由于其复杂性和额外的抽象层，一些同学不能十分理解具体逻辑情况下可能会导致刷新问题

跟页面绑定太严重，如果以后想要切换库难度比较大

**3、社区支持和文档**

- Provider的社区支持：Provider是Flutter官方推荐的状态管理解决方案之一，拥有庞大的用户群体和积极的开发团队

**4、Bloc还要依赖各种其他三方库**

- rxdart
- flutter_bloc
- provider
- equal

### 请简单介绍下Flutter框架，以及它的优缺点

**优点**

**1、跨平台性**：Flutter能够使用同一份代码部署到iOS、Android、Web和桌面等多个平台，极大地降低了开发成本和时间。
**2、性能强大**：Flutter拥有高效的渲染引擎和优化的底层实现，能够提供流畅的用户体验和接近原生应用的性能。
**3、丰富的组件库**：Flutter提供了丰富的Material Design和Cupertino（iOS风格）组件，同时支持自定义组件样式和行为。
**4、热重载功能**：在开发过程中，Flutter支持热重载功能，可以在不重启应用的情况下即时看到代码更改的效果，提高开发效率。

**缺点**

**1、适配问题**：随着操作系统和Flutter框架的不断更新，开发者可能需要修改大量的代码以适应新的版本和特性
2、三方库有限，需要自己造轮子；
**3、代码可读性较差**：Flutter的代码可读性相对较差，特别是对于不熟悉Dart语言和Flutter框架的开发者来说，可能需要花费更多的时间来理解和维护代码。
**4、打包后文件较大**：由于Flutter应用包含了自己的渲染引擎和框架代码，因此打包后的应用文件相对较大，可能会影响应用的下载和安装速度。

### Widget生命周期

flutter生命周期其实就是Widget的生命周期，生命周期的回调函数体现在了State上面。 主要可以分成两方面讨论

1、StatelessWidget
2、StatefulWidget

#### 1、StatelessWidget

StatelessWidget是一个无状态的Widget，它不会根据用户交互或内部状态的变化进行重建。因此，StatelessWidget的生命周期相对简单，主要涉及：

- **构建（Build）** ：这是StatelessWidget唯一的方法，用于创建和返回Widget的UI表示。每次在页面刷新时都会调用build方法，但它本身不包含状态，因此不会触发重建过程

#### 2、StatefulWidget

StatefulWidget是一个有状态的Widget，它可以根据状态的变化进行更新和重建。StatefulWidget的生命周期更为复杂，包括以下阶段：

**1、createState()**

在StatefulWidget被创建时调用，用于创建与该Widget关联的State对象。

**2、initState()**

在State对象创建后立即调用，通常用于初始化数据，如网络请求、动画控制器等。在这个阶段，不能使用BuildContext进行UI构建。

**3、didChangeDependencies()**

当State依赖的对象改变时调用
- 例如InheritedWidget更新时会触发此方法。它会在initState之后立即执行，并且可能会在依赖的InheritedWidget发生变化后再次执行。

**4、build(BuildContext context)**

用于构建Widget的子树。每次Widget状态改变时都会被调用。这是实际创建和返回UI表示的地方。

**5、didUpdateWidget(covariant T oldWidget)**

当Widget被重新构建时调用，例如父Widget的状态改变时。这允许子Widget响应父Widget的变化。

**6、setState(VoidCallback fn)**

**用于更新状态并触发Widget的重新构建**。
- 当调用setState时，Flutter会安排该Widget的build方法在未来的某个时间点被调用，以反映新的状态。

**7、deactivate()**

在State对象从Widget树中移除时调用。如果Widget被移除后没有被添加到其他Widget树中，则随后会调用dispose方法。

**8、dispose()**

在State对象被永久移除并释放资源时调用。这是清理资源（如取消订阅、释放控制器等）的好地方。



### 简单的解释下Flutter的FrameWork层和Engine层

#### FrameWork层

Dart编写的UI框架

#### Engine层

Skia是Google的一个 2D的绘图引擎库

### flutter渲染原理（或者页面绘制逻辑）

自写UI渲染引擎实现跨平台

1、 Dart 来构建UI 视图 

2、`Skia` 交给 GPU 渲染

**Flutter渲染流程大致如下：**

1、在Dart framework中构建Widget树。

2、调用rendering库，将Widget树转换为Element树。

3、再将Element树转换为RenderObject树，进行布局和绘制。

4、通过Skia引擎，将RenderObject树转换为GPU命令，并提交给GPU执行。

5、GPU执行这些命令，最终将渲染结果显示在屏幕上。

### 说下Widgets、RenderObjects 和 Elements的关系

- 1、Widget 是不可变的（immutable），且只有一帧，且不是真正工作的对象，每次画面变化，都会导致一些 Widget 重新 build
- 2、Render Tree是实际渲染的结果

因为是Flutter是一种树结构，而我们的widget不稳定，会经常变化，如果每次修改一个小的widget都要重新渲染一次界面会比较消耗性能。

为了解决这个问题，引入了Element Tree。Element Tree和widget Tree根据key和widget进行比较，如果发生改变，就渲染改变的部分

#### 关系

- 一个Widget会创建一个Element对象，是通过createElement()创建的。
- 一个Element持有一个RenderObject和一个Widget。Widget 、Element都不负责最后的渲染，真正负责的渲染只有 RenderObject。

#### BuildContext build(BuildContext context)

BuildContext 主要用于构建 UI 和管理状态 

**Context记录当前Widget在Widget Tree中的具体位置和相关信息，用以保证各个子Widget和其parent的相对位置等信息**

1、获取当前主题：final theme = Theme.of(context);

2、查找父级 Widget：final parentWidget = context.widget;

3、获取设备信息：
- final mediaQuery = MediaQuery.of(context);
- final size = mediaQuery.size;
- final orientation = mediaQuery.orientation;

4、导航到新页面

5、访问资源与数据

资源访问：BuildContext可以用于访问与当前Widget相关联的资源，如本地化字符串、图片等。

数据传递：虽然BuildContext本身不直接用于数据传递，但它通过Widget树的结构，使得数据可以通过Widget的构造函数或InheritedWidget等方式在Widget之间传递。

#### 无BuildContext跳转

MaterialApp的构造函数有个navigatorKey的参数，定义一个全局的GlobalKey, 再在需要使用当前BuildContext的地方，直接从GlobalKey中获取即可

```dart
static final GlobalKey<NavigatorState> globalNavigatorKey =
      GlobalKey<NavigatorState>();
var currentContext = GlobalKey.currentContext
```



### **Flutter事件响应过程**

**1、** **事件接收**

**入口点：** Flutter中事件的接收通常从引擎层开始，通过`_dispatchPointerDataPacket`等函数接收来自系统或设备的手势数据。这些数据包括按下、移动、抬起等基本的指针事件

**数据转换**：接收到的手势数据（如PointerData）会被转换成Flutter内部使用的PointerEvent类或其子类

**2、命中测试（Hit Test）**

**触发时机**：当PointerDownEvent事件发生时，Flutter会触发命中测试。

**测试过程**：命中测试按照深度优先的顺序遍历当前的渲染树（Render Tree），对每一个渲染对象（RenderObject）进行“命中测试”（hit test）。如果事件的位置与某个渲染对象相交，则认为该对象通过了命中测试，并将其添加到HitTestResult列表中。

**3、事件分发（Event Dispatch）**

**分发过程**：命中测试完毕后，Flutter会遍历HitTestResult列表，并调用列表中每个渲染对象的事件处理方法（如handleEvent）来处理事件。这个过程称为“事件分发”。

**顺序**：由于命中测试是按照深度优先的顺序进行的，因此子组件会比父组件先加入HitTestResult列表，并在事件分发时先被调用。这保证了子组件能够优先响应事件

**4、事件处理**

**Widget响应**

**5、事件清理**

**触发时机**：当PointerUpEvent或PointerCancelEvent发生时，表示手势结束或取消，此时会进行事件清理。

**清理过程**：事件清理包括分发最后的事件（如PointerUpEvent）并清空HitTestResult列表，为下一次事件处理做准备。

### Flutter中的路由管理是如何实现的？

- Flutter中的路由管理通常通过Navigator组件来实现。Navigator是一个Widget，它维护了一个路由栈，用于管理页面之间的跳转和返回。
- 你可以使用Navigator.push()方法来将一个新的页面推入路由栈，并使用Navigator.pop()方法来从路由栈中弹出当前页面。
- Flutter还提供了Named Routes等高级路由管理功能，允许你通过名称来导航到不同的页面，而不是直接引用Widget实例。

#### Flutter 路由管理中有两个非常重要的概念：

- Route：路由是应用程序页面的抽象，对应 Android 中 Activity 和 iOS 中的 ViewController，由 Navigator 管理。
- Navigator：Navigator 是一个组件，管理和维护一个基于堆栈的历史记录，通过 push 和 pop 进行页面的跳转。

**根据是否需要提前注册页面标识符，Flutter 中的路由管理可以分为两种方式：**

- 基本路由。无需提前注册，在页面切换时需要自己构造页面实例。
- 命名路由。需要提前注册页面标识符，在页面切换时通过标识符直接打开新的路由。

#### 我有三个页面，A、C是flutter页面，B是原生页面，我从A到B再到C，然后返回A，页面会卡死。但是如果只是从A到B再返回，或者是从A到C再返回

1、**检查原生页面的生命周期**

**2、** **Flutter与原生页面的交互**：

- 如果在Flutter和原生页面之间有数据交换或状态同步，确保这些操作在页面切换时能够正确处理。例如，使用`MethodChannel`或`EventChannel`时，确保在Flutter页面销毁时取消监听或发送停止信号

3、检查A页面是否有任何未完成的异步操作或监听器，确保在返回A页面时，A页面的状态被正确重置或恢复

4、**导航和路由管理**：

- 如果你使用的是Flutter的Navigator进行页面跳转，确保在返回A页面时，Navigator的堆栈被正确管理。例如，使用`Navigator.pop()`或`Navigator.popUntil()`来确保返回到正确的页面。

### flutter run实际走了哪三个命令？分别用于什么操作？

`flutter build apk`：通过`gradle`来构建APK

`adb install`：安装APK

`adb am start`：启动应用

### Flutter开发插件的流程

#### 1、创建Flutter插件项目

使用Flutter命令行工具创建插件项目。命令格式通常为：**flutter create --template=plugin [插件名称]** 。这里--template=plugin参数指定了创建一个插件项目。

可以通过添加--platforms=android,ios来指定同时支持Android和iOS平台。

可以使用-i和-a选项来指定iOS和Android平台的开发语言，如Swift和Java。

#### 2、 编写插件代码

Dart代码：在lib目录下创建Dart文件，该文件将作为Flutter端与原生端通信的桥梁。使用**MethodChannel**来定义与原生代码交互的通道。

原生代码：

- iOS：在ios/Classes目录下创建Objective-C或Swift文件，实现具体的功能，并通过FlutterMethodChannel与Dart代码通信。
- Android：在android/src/main/java/[包名]目录下创建Java或Kotlin文件，实现相应的功能，并通过MethodChannel与Dart代码通信。

#### 3、实现功能

- 在Dart文件中定义插件的API，这些API将作为Flutter应用调用原生功能的接口。
- 在原生代码中实现具体的功能逻辑，并通过`MethodChannel`将结果返回给Dart端。

#### 4、准备发布

- 在`pubspec.yaml`文件中设置插件的元数据，包括插件的名称、描述、作者、主页等。

### Flutter 是如何与原生Android、iOS进行通信的

Flutter 为开发者提供了一个轻量级的解决方案，即逻辑层的**方法通道（** **PlatformChannel** **）机制**

1、**BasicMessageChannel** ：**基础数据传递**，如JSON对象或自定义数据结构。支持双向通信，即Flutter和原生端都可以主动发送消息

2、**MethodChannel ：用于传递方法调用**，允许有返回值，适用于一次性的通信。同样支持双向通信，Flutter和原生端都可以调用对方的方法。

- 当需要从Flutter调用原生端的方法，并获取执行结果时，如打开系统设置、相机等。
- 原生端也可以主动调用Flutter的方法，实现更复杂的交互逻辑

3、**EventChannel : 用于数据流（event streams）的通信**。支持原生端主动向Flutter发送数据流，如**传感器数据、用户输入**等。

- 当原生端需要向Flutter发送实时数据流时，如实时定位数据、音频流等。
- 监听原生平台的状态变化，如网络状态、电池电量等。

**使用注意点**

1、BasicMessageChannel

- **Channel名称一致性**
- **编解码器一致性**
- 基本数据类型和集合类型，不支持自定义类型。

2、MethodChannel

- **Channel名称唯一性**
- **方法调用与响应**：Flutter端可以调用原生端的方法，并获取返回值；同样，原生端也可以调用Flutter端的方法。需要确保方法名称和参数在两端一致。
- **异常处理**：在原生端处理方法时，应捕获并处理可能发生的异常，以避免应用崩溃。

3、EventChannel

- **线程安全**：在原生端处理Flutter发来的消息或调用时，需要注意线程安全问题。尤其是在Android平台上，可能需要使用主线程或其他线程来处理耗时操作。

  

### 简述下Flutter 的热重载

**热重载是指，在不中断 App 正常运行的情况下，动态注入修改后的代码片段。**

Flutter 的热重载功能可帮助您在无需重新启动应用程序的情况下快速添加功能以及修复错误。**通过将更新的源代码文件注入到正在运行的 Dart 虚拟机（VM） 来实现热重载**。在虚拟机使用新的字段和函数更新类之后， Flutter 框架会自动重新构建 widget 树，以便您可以快速查看更改的效果

#### Flutter编译方式

Flutter在Debug和Relase执行不同的编译模式

- JIT：Just In Time . 动态解释，一边翻译一边执行，也称为即时编译，如JavaScript，Python等,在开发周期中使用，可以动态下发和执行代码，开发测试效率高，但是运行速度和性能则会受到影响，Flutter中的热重载正是基于此特性
- AOT: Ahead of Time. 静态编译，是指程序在执行前全部被翻译为机器码，提前编译,如 C ，C++ ，OC等，发布时期使用AOT，就不需要像RN那样在跨平台JavaScript代码和原生Android、iOS代码间建立低效的方法调用映射关系。

### Widget 唯一标识Key有哪几种？

**key的作用：** 主要决定是否要刷新widget

- 这是因为修改了key之后，Element会强制刷新，那么对应的State也会重新创建

Key本身是一个抽象，不过它也有一个工厂构造器，创建出来一个ValueKey

直接子类主要有：**LocalKey和GlobalKey**

LocalKey，它应用于具有相同父Element的Widget进行比较，也是diff算法的核心所在；

- LocalKey 是 Key 的基类，分为两种：ValueKey 和 ObjectKey，用于在局部 Widget 树中唯一标识某个 Widget。
- LocalKey有三个子类
  - `ValueKey` 使用具体的值（如字符串、数字）作为键，通常用于标识列表项等。
  - ObjectKey：`ObjectKey` 使用某个对象作为键，适用于需要通过对象标识的场景。
  - UniqueKey

- 如果我们要确保key的唯一性，可以使用UniqueKey；
- 比如我们之前使用随机数来保证key的不同，这里我们就可以换成UniqueKey；

GlobalKey，**通常我们会使用GlobalKey某个Widget对应的Widget或State或Element**

- GlobalKey 是一个全局唯一的键，用于标识整个应用中的特定 Widget。
- 它可以跨越 Widget 树的多个层次来访问 Widget，并提供更强的控制和状态管理。

## Flutter跟原生交互遇到过什么问题

#### 1、OOM堆内存溢出，内存暴涨

- 1、长列表出现内存溢出问题
  - 图片通过阿里云处理，下载小图
  - 图片尽可能使用缓存，这个可以使用CachedNetworkImage插件实现
  - 要么手动删缓存，要么把缓存大小弄小一点，就不会内存溢出了`
  - 运行过程中不变的组件使用const修饰（例如固定的icon），这样可以复用组件，渲染效率更高。
  - item设置key
  - 设置ScrollAwareImageProvider：判断快速滑动, 下载和解码会停止

- **2、数据传递**：在交互过程中传递大量数据（如图片、视频等）而未进行适当处理（如压缩、缓存等）也会增加内存负担。

- 3、Flutter内存泄露

#### 2、插件实现缺失错误

iOS中运行Flutter应用时，可能会遇到`MissingPluginException`错误，提示某个方法在指定的channel上没有找到实现。这通常是因为Flutter插件在iOS端没有被正确注册或初始化。

**解决方案**：

- 确保在iOS项目的`AppDelegate.swift`或`AppDelegate.m`文件中正确注册了插件。对于Swift项目，可以使用`GeneratedPluginRegistrant.register(with: self)`来注册所有插件，或者手动注册特定插件。
- 检查插件的iOS端实现是否完整，包括Objective-C或Swift代码以及必要的配置。

##### 问题解决

**定义全局字符串常量**

#### 3、消息传递问题

使用`MethodChannel`、`BasicMessageChannel`或`EventChannel`等通道进行Flutter与iOS原生代码之间的消息传递时，可能会遇到消息丢失、格式错误或解析失败等问题。

解决方案：

- 确保Flutter端和iOS原生端使用的channel名称完全一致。
- 使用合适的编解码器来序列化和反序列化消息，确保数据类型和格式在两端保持一致。
- 在调试过程中，可以使用日志打印来检查消息是否成功发送和接收。

##### 问题解决

**确定好格式**

```css
{
  "code":"",
  "message":"",
  "data":""
}
```

#### 4、权限和配置问题

问题描述：
 在iOS设备上运行Flutter应用时，可能会遇到与权限相关的错误，如相机、麦克风或位置权限未被授予。

解决方案：

- 在iOS项目的`Info.plist`文件中添加必要的权限声明。
- 在Flutter应用中请求并处理权限请求的结果。
- 确保在iOS设备的设置中已授予应用相应的权限。

#### 5、iOS混合flutter，网络请求用iOS原生的还是用flutter的

最好用dio

##### iOS原生的网络请求

- **性能**：iOS原生的网络请求库（如`NSURLSession`）通常具有较高的性能
- 缺点：**代码复用性**：如果项目中同时包含Flutter和iOS原生代码，使用iOS原生网络请求可能需要在Flutter和iOS原生之间传递数据，这可能会增加代码的复杂性。

##### iOS flutter混编，怎么解决用户信息同步问题

- **使用共享数据存储**：

- - 可以在iOS原生和Flutter之间共享一个数据存储方案，如SQLite、Core Data或Realm等。
  - 利用iOS的`NSUserDefaults`或Flutter的`SharedPreferences`等存储简单的用户配置信息。

- **即时同步**：

- - 当用户信息发生变化时，立即触发同步操作，确保信息的实时性。这通常通过事件总线或平台通道实现。

## FlutterViewController

FlutterViewController是iOS平台上的一个控制器，用于**管理Flutter引擎和Flutter视图的展示**。它允许iOS开发者在iOS应用中嵌入Flutter模块，实现跨平台的UI和功能。

使用方法

- 1、**创建FlutterViewController**：

- - FlutterViewController是通过与FlutterEngine关联来创建的。FlutterEngine是Flutter的运行时环境，负责管理Dart虚拟机、UI渲染等。

- 2、**设置初始路由**：

- - FlutterViewController支持设置初始路由，以便在加载时直接显示指定的Flutter页面。

- 3、**展示FlutterViewController：**

- - 创建并配置好FlutterViewController后，可以将其添加到iOS应用的UI中展示。

#### FlutterViewController与Flutter引擎的关系

- Flutter引擎是一个用C++编写的底层框架，它负责Flutter应用的线程管理、Dart虚拟机（VM）状态管理以及Dart代码的加载和执行
- FlutterViewController是Flutter嵌入原生iOS应用的一个关键类。它作为Flutter内容的载体，负责在iOS应用中显示和管理Flutter页面。

**管理方式**：

- FlutterViewController可以外部管理Flutter引擎（即FlutterEngine实例），也可以不管理。如果不管理，则每初始化一个FlutterViewController实例，其内部都会创建一个新的FlutterEngine实例。这种方式的缺点是每个FlutterEngine实例都会占用一定的内存空间，并且在页面销毁时内存可能不会被释放。
- 为了节省内存和提高性能，通常建议外部管理FlutterEngine实例，即不管创建多少个FlutterViewController实例，都只使用一个FlutterEngine实例。这样可以确保所有的Flutter页面都共享同一个引擎资源，从而避免内存浪费和性能下降。

## iOS 项目中混编Flutter路由管理问题、怎么统一路由表

### Flutter与iOS混编中的路由问题

1. **路由注册**：

- - 在Flutter端，需要在应用中注册路由表，以便在需要时跳转到指定的页面。
  - 在iOS原生端，需要注册Flutter引擎，并配置FlutterViewController来承载Flutter页面。同时，还需要实现原生与Flutter之间的路由跳转协议。

1. **路由跳转**：

- - 在Flutter端，可以使用`Navigator.pushNamed`或`Navigator.push`方法来实现页面跳转。
  - 在iOS原生端，可以通过调用Flutter引擎的路由跳转接口来实现对Flutter页面的跳转。

1. **路由返回**：

- - 在Flutter中，可以使用`Navigator.pop`方法返回上一个页面。
  - 在iOS原生端，需要实现相应的返回逻辑，以便在返回时能够**正确地关闭Flutter页面并返回到原生页面。**

#### FlutterBoost是单页面模式还是多页面模式

在FlutterBoost1.0版本中，它主要是单页面模式，即不管你打开多少Flutter页面，其实呈现页面的FlutterViewController或者FlutterView其实仅有一个。这种模式是出于节省内存的考虑，但也可能导致一些问题，如页面切换时的白屏或黑屏问题，以及页面生命周期管理的复杂性。

在FlutterBoost的新版本中，它不再维护单一的FlutterViewController（或FlutterView），而是和原生一样，每次有新页面请求时就直接打开新的ViewController或者FlutterView。这样，每个Flutter页面都可以有自己的生命周期管理，解决了之前单页面模式下的一些问题。

## iOS Flutter混编，单引擎、多引擎

### 单引擎方案

单引擎方案指的是在整个iOS应用中，只使用一个FlutterEngine来渲染所有的Flutter页面。

**FlutterBoost是单引擎方案**

1. **优点**：

- - 内存占用小：由于只有一个FlutterEngine，因此可以节省内存资源。
  - 插件通道管理方便：所有的Flutter页面都共享同一个FlutterEngine，因此插件通道也只需要管理一套。

1. **缺点**：

- - 页面动态性不足：如果页面在跳转期间有变动或更新，由于采用截图方式（在某些实现中），可能会导致页面闪烁或延迟。
  - 页面跳转可能有延迟：由于需要在新页面打开之前进行截图，且截图过程可能耗时，因此会影响页面跳转的流畅性。

##### 适用场景：















































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































































+++

































- - **页面跳转响应速度快**：由于采用不同的FlutterEngine交替渲染页面，因此页面跳转过程中无需对页面进行截图处理，从而提高了跳转速度。
  - **支持页面的动态变化**：每个显示在屏幕上的可见视图都是实际的FlutterView，因此页面的更新也是即时的。

1. **缺点**：

- - **插件管理相对复杂：** 每个FlutterEngine都有单独的插件通道，因此需要协调不同插件在不同FlutterEngine上的调用。
  - **内存占用可能较大**：由于使用了多个FlutterEngine，因此会占用更多的内存资源。

##### 适用场景：

适用于需要在iOS应用中**嵌入多个独立的Flutter模块**，或**者需要实现复杂的页面跳转和动态更新的情况**。

#### 为什么市面多引擎用的人那么少？

内存崩溃风险、**插件管理相对复杂**

- 缺乏内部固定布局方式，只能通过外部布局位置大小来让 FlutterView 自适应。
- 通信层极其繁琐，从有限的 Demo 中看出需三端各自实现 Bridge Channel。桥方法通过“字符串”作为对应类型，导致个性化开发维护成本非常高。
- 应用场景狭窄，多 FlutterEngine 间只能通过 Native 交互通信。
- Flutter Debug 模式下多引擎 = 内存炸裂，要用 Flutter Release 才可以稳定正常到官方描述的 180K / Engine 的内存占用效果

## iOS 项目中混编Flutter，怎么单独调试Flutter模块

**flutter attach 调试**

1. **连接设备**：

- - 将iOS设备通过USB连接到计算机，并确保Xcode已经识别到该设备。

1. **使用**`flutter attach`**命令**：

- - 在Android Studio的终端中，输入`flutter attach`命令，并指定要连接的iOS设备ID。这个命令会将Flutter开发工具与正在运行的iOS设备上的Flutter模块连接起来。
  - 可以通过`flutter devices`命令查看已连接到计算机的设备列表，并获取设备ID。

1、使用vscode

VSCode 编辑 launch.json -> 追加如下代码：

```json
json
{
    "name": "Flutter: Attach to Device",
    "type": "dart",
    "request": "attach"
}
```

## Flutter内存泄露

### Dart的内存分配与垃圾回收是怎么样的？

内存分配策略比较简单，创建对象时只需要在堆上移动指针，内存增长始终是线性的，省去了查找可用内存的过程

Dart 的垃圾回收，则是采用了多生代算法。新生代在回收内存时采用“半空间”机制，触发垃圾回收时，Dart 会将当前半空间中的“活跃”对象拷贝到备用空间，然后整体释放当前空间的所有内存。回收过程中，Dart 只需要操作少量的“活跃”对象，没有引用的大量“死亡”对象则被忽略，这样的回收机制很适合 Flutter 框架中大量 Widget 销毁重建的场景。

### 内存泄露原因

- 1、**不正确的状态管理**：**BuildContext 泄漏**：错误地长时间持有 BuildContext 引用，尤其是在异步操作中，可能导致相关的 Widget 无法被释放。

- - **避免在异步回调中直接持有** `BuildContext`：当你需要在异步操作完成后更新 UI 时，不要直接将 `BuildContext` 传递给异步回调。相反，你应该使用某种机制（如全局键、全局状态管理库等）来在异步操作完成后找到正确的 Widget 并更新它。

- 2、**未关闭的资源**：

- - 打开的文件、网络连接、数据库连接等资源如果没有正确关闭，会导致这些资源占用的内存无法被释放。
  - **Isolate 未正确关闭**：在 Flutter 中创建新的 Isolate 以执行后台任务，如果任务完成后未正确关闭 Isolate，可能会导致内存泄漏。

- 3、**未取消的订阅和监听**

- - **事件监听和回调**：添加了事件监听器或回调函数，如定时器（`Timer`）、动画监听（`AnimationController`监听）、外部库事件订阅等，而在 Widget 销毁时没有正确取消或移除。
  - **Stream 和 StreamController**：监听 Stream 时未取消订阅，特别是在 Widget 被销毁后，如果 Stream 还在发送事件，会导致订阅者（通常是一个 Widget）无法被垃圾收集器回收

- 4、**图片缓存**：

- - Flutter的图片缓存机制在加载大量图片时可能会导致内存泄露，特别是如果未设置合理的缓存大小限制。

- 5、**闭包中的引用**：

- - Dart中的闭包可以捕获并持有外部变量的引用，如果闭包被长时间持有，那么它捕获的变量也将被长时间持有，可能导致内存泄露

- 6、**循环引用**：

- - 对象之间的循环引用是内存泄露的常见原因。在Dart中，虽然垃圾回收器可以处理一些简单的循环引用情况，但在复杂情况下仍可能导致内存泄露。

- 7、**原生插件和方法通道**：

- - 在使用Flutter的原生插件和方法通道时，如果未正确管理原生资源，也可能导致内存泄露

- 8、**全局状态或单例对象**：未被正确管理的全局状态或单例对象，尤其是当它们持有大量数据或资源时，可能导致内存泄漏。

### 内存泄露检测

#### 线下

- 1、**使用Flutter DevTools进行监控**

- - 在DevTools中，切换到“Memory”标签。你可以查看内存使用情况的实时图表，进行堆快照分析等。

- 2 **、Dart Observatory**：这是一个命令行工具，提供更详细的内存使用情况分析，有助于深入了解内存分配和回收情况。

- - Observatory /əbˈzɜːvətri/ 天文台 观测台

- **3、Dart vm_service 查找哪些对象没有被释放**

- - 1、Dart VM Service（vm_service）是Dart虚拟机内部提供的一套Web服务。通过vm_service，可以获取到Dart堆内存中的对象信息，进而分析内存泄露。

- **4、借助插件flutter_memory**: 查看内存

## Flutter错误收集

## 布局溢出

- 1、**使用**`DebugPaintSizeEnabled`将`DebugPaintSizeEnabled` widget包裹在你的应用根widget上，来在屏幕上绘制出每个widget的边界和大小。这有助于你直观地看到哪些widget可能超出了其父widget的边界。
- 2、检测工具 Flutter Inspector (debug模式下)

- - **DevTools**

- 3、在你的widget树中，你可以编写自定义的溢出检测逻辑。这通常涉及到在widget的`build`方法或自定义`RenderObject`的`performLayout`方法中检查子widget的布局尺寸和父widget的约束。

## 线上异常收集

- 1、 **Dart层异常捕获**；可以通过try-catch机制直接捕获
- 2、**Flutter框架错误捕获**

- - 异步异常 Future.catchError适用于捕获异步操作中的异常
  - 全局异常捕获 FlutterError.onError全局异常捕获
  - 全局异步回调异常 PlatformDispatcher.instance.onError
  - 使用Zone.runZoned集中捕获：（过时）

- 3、**原生崩溃捕获：集成原生崩溃收集工具、入Bugly等**

## Flutter 3.0相对于2.0有哪些优化

#### 1. 全平台支持增强

- **新增平台支持**：Flutter 3.0进一步强化了对各种平台的支持，不仅仅是移动平台（iOS和Android），还包括Web、Windows、MacOS和Linux。这意味着开发者可以使用同一套代码基础为几乎所有主流平台构建应用，大大提高了开发效率并减少了维护成本。

#### 2. 性能提升

- **渲染性能优化**：Flutter 3.0在渲染性能上进行了多项优化，使得用Flutter开发的应用能够更加流畅地运行，提升用户体验。
- **应用启动时间缩短**：通过优化启动流程和减少不必要的资源加载，Flutter 3.0显著缩短了应用的启动时间。

#### 3. 组件更新与改进

- **新组件与改进**：Flutter 3.0引入了多个新的组件和对现有组件的改进，这些组件使得创建复杂和功能丰富的用户界面变得更加简单。例如，新的导航和滚动组件提供了更多的灵活性和更好的性能。

#### 4. 开发工具增强

- **Dart DevTools增强**：Flutter 3.0对Dart DevTools进行了增强，使得调试和分析Flutter应用更加容易。
- **IDE支持增强**：集成开发环境（IDE）的支持也得到了增强，提供了更多的代码完成和分析功能，帮助开发者提高开发效率。

## Dart 3.0相对于2.0有哪些优化

#### 1、健全的空值安全

- **100%健全的空值安全**：Dart 3.0将只支持健全的Null安全，这意味着所有之前不安全的空值处理方式都将不再被支持。这种改变为Dart带来了更加健全的类型系统，减少了空指针异常等类型的编码错误，并允许编译器和运行时以更优化的方式处理代码。

### Flutter怎么适配屏幕

#### 普通手机

- 1、使用自适应布局，Flutter提供了丰富的布局Widget，如`Row`、`Column`、`Flex`、`Grid`等
- 2、`flutter_screenutil`是一个流行的Flutter插件，用于调整屏幕和字体大小
- 3、自定义适配类，可以定义一个`AdaptiveScreen`类，该类包含获取屏幕宽度、高度和像素比的方法，以及根据设计稿尺寸计算UI元素尺寸的方法。

#### 折叠屏适配

- **三种形态适配**：折叠屏设备通常有展开大屏、折叠主屏和折叠副屏三种形态。开发者需要针对这三种形态分别进行适配，确保应用在每种形态下都能正确显示。
- **动态热切换适配**：当折叠屏设备从一种形态切换到另一种形态时，应用需要能够实时感知并调整布局。Flutter提供了相关的API，如`MediaQuery`中的`DisplayFeatures`列表，用于描述显示部件的边界和状态，如铰链、折叠和刘海。

#### 大屏幕适配

- **高分辨率与宽高比**：大屏幕设备通常具有更高的分辨率和更宽的宽高比。开发者需要确保应用能够充分利用这些额外的屏幕空间，同时保持UI元素的清晰度和可读性。
- **布局重构**：对于大屏幕设备，可能需要重新设计布局以适应更大的屏幕。例如，可以使用`Grid`布局或自定义布局来更好地利用屏幕空间。

