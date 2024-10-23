# **stateful and stateless**

Flutter中的Widget主要分为两类StatelessWidget和StatefulWidget。

> StatelessWidget表示无状态的Widget。
>
> StatefulWidget表示有状态的Widget。



### StatelessWidget：

StatelessWidget 是一个无状态的组件，一旦构建后状态就不能改变其状态，无生命周期的回调。StatelessWidget 通常用于构建静态内容不会随时间变化的界面部分。例如，如果你有一个固定的文本或图标，可以使用 StatelessWidget 来表示它，因为它们不需要响应用户交互或动态更新。

- 它们的数据通常是直接写死。
- 从parent widget中传入的而且一旦传入就不可以修改。
- 从InheritedWidget获取来使用的数据；

##### StatelessWidget包含一个必须重写的方法：build方法。

StatelessWidget没办法主动去执行build方法，当我们使用的数据发生改变时，build方法会被重新执行
 build方法什么情况下被执行呢？：

- 当我们的StatelessWidget第一次被插入到Widget树中时（也就是第一次被创建时）；
- 当我们的父Widget（parent widget）发生改变时，子Widget会被重新构建；
- 如果我们的Widget依赖InheritedWidget的一些数据，InheritedWidget数据发生改变时；

### StatefulWidget：

StatefulWidget 是一个有状态的组件，构建Widget的状态还会发生改变的Widget。StatefulWidget 具有一个与之关联的可变状态对象（State），可以根据状态的变化来更新界面。当状态数据发生变化时，Flutter 会调用 build() 方法重新构建界面，以反映最新的状态。StatefulWidget 适用于需要处理用户输入、展示动态数据或实现交互逻辑的场景。

##### Flutter如何做到我们在开发中定义到Widget中的数据一定是final的呢？

有一个很关键的东西@immutable

- 我们似乎在Dart中没有见过这种语法，这实际上是一个 注解，这涉及到Dart的元编程；
   官方有对@immutable进行说明：
- 来源： [https://api.flutter.dev/flutter/meta/immutable-constant.html](https://links.jianshu.com/go?to=https%3A%2F%2Fapi.flutter.dev%2Fflutter%2Fmeta%2Fimmutable-constant.html)
- 说明： 被@immutable注解标明的类或者子类都必须是不可变的

**结论**： 定义到Widget中的数据一定是不可变的，需要使用final来修

##### 如何存储Widget状态？

1)既然Widget是不可变，那么StatefulWidget如何来存储可变的状态呢？

- StatelessWidget无所谓，因为它里面的数据通常是直接定义完后就不修改的。
- 但StatefulWidget需要有状态（可以理解成变量）的改变，这如何做到呢？

2)Flutter将StatefulWidget设计成了两个类：

- 也就是你创建StatefulWidget时必须创建两个类：
- 一个类继承自StatefulWidget，作为Widget树的一部分；
- 一个类继承自State，用于记录StatefulWidget会变化的状态，并且根据状态的变化，构建出新的Widget；

3)创建一个StatefulWidget，我们通常会按照如下格式来做：

- 当Flutter在构建Widget Tree时，会获取State的实例，并且它调用build方法去获取StatefulWidget希望构建的Widget；

##### 为什么Flutter要这样设计呢？

因为在Flutter中，只要数据改变了Widget就需要重新构建（rebuild）

##### InheritedWidget是什么，并说明它在状态管理中的作用。

- `InheritedWidget` 类是一个特殊的无界面（stateless）Widget，它主要用于在整个Widget树中｀自顶向下｀地高效传递和共享数据（`Notifation`用于`自下而上`传递数据）。
- InheritedWidget能够提供数据在widget树中从上到下进行传递。保证数据在不同子widget中进行共享。
- 作用
   1、数据共享
   2、数据监听与重建
   3、性能优化：只重绘受影响的部分子树

##### Provider与其他状态管理解决方案有什么不同？

- provider是基于InheritedWidget的包装，可以实现页面和其子页面间的数据共享。

- Provider具有`缓存`功能，调用setState()方法，那些没有依赖状态的子节点都`不会被重新build`。

  

#### StatefulWidget生命周期

![img](https://upload-images.jianshu.io/upload_images/1013424-d529740bacbba2a4.png?imageMogr2/auto-orient/strip|imageView2/2/w/975/format/webp)



**State生命周期：**

理解State的生命周期对flutter开发非常重要，如果想看详细的请参见 [state生命周期](https://links.jianshu.com/go?to=https%3A%2F%2Fbook.flutterchina.club%2Fchapter2%2Fflutter_widget_intro.html%23state%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F)，我这里对生命周期主要方法总结一下：

- 1、`initState()`当 widget 第一次插入到 widget 树时会被调用，对于每一个State对象，Flutter 框架只会调用一次该回调。
- 2、`didChangeDependencies()` 当State对象的依赖发生变化时会被调用；
- 3、`build()`主要是用于构建 widget 子树的，会在如下场景被调用：

> 1、在调用`initState()`之后。
>  2、在调用`didUpdateWidget()`之后。
>  3、在调用`setState()`之后。
>  4、在调用`didChangeDependencies()`之后。
>  5、在State对象从树中一个位置移除后（会调用deactivate）又重新插入到树的其它位置之后。

- 4、`reassemble()：`此回调是专门为了开发调试而提供的，在热重载(hot reload)时会被调用，此回调在Release模式下永远不会被调用。
- 5、`didUpdateWidget()`在新旧 widget 的key和runtimeType同时相等时didUpdateWidget()就会被调用。
- 6、`deactivate()`当 State 对象从树中被移除时，会调用此回调；
- 7、`dispose()`当 State 对象从树中被永久移除时调用；通常在此回调中释放资源；

##### 小结

前期开发记住几点就行了：
 1、先走`initState()`，后走`build()`；
 2、所有的UI构建要放在`build()`里面；
 3、刷新UI直接调用`setState()`；
 4、回收垃圾在`dispose()`。

# 总结

> - Q：什么时候使用`StatelessWidget`和`StatefulWidget`？
>    A：如果一个UI在初始化之后就不用更新，那就使用`StatelessWidget`，反之`StatefulWidget`。
> - Q：既然都是Widget，啥是VC，啥是view？
>    A：`StatelessWidget`、`StatefulWidget`的build方法里面，如果第一子节点是`Scaffold`（带有`appBar、body、bottomNavigationBar`这种的），那就可以理解为这个页面是VC，反之可以理解为view。
> - Q：iOS中的self在Flutter中是啥？
>    A：首字母小写的`widget`。

。
