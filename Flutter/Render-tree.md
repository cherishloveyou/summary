# Flutter的核心渲染模块三棵树

Flutter 中有三棵树：Widget 树，Element 树和 RenderObject 树。其中 **`Widget`与 `Element` 是一对多的关系** ，**`Element` 与 `RenderObject` 是一一对应的关系**。

当应用启动时 Flutter 会遍历并创建所有的 Widget 形成 Widget Tree，同时与 Widget Tree 相对应，通过调用 Widget 上的 createElement() 方法创建每个 Element 对象，形成 Element Tree。最后调用 Element 的 createRenderObject() 方法创建每个渲染对象，形成一个 Render Tree。 Element就是Widget在UI树具体位置的一个实例化对象，大多数Element只有唯一的renderObject，但还有一些Element会有多个子节点，如继承自RenderObjectElement的一些类，比如MultiChildRenderObjectElement。最终所有Element的RenderObject构成一棵树，我们称之为”Render Tree“即”渲染树“。总结一下，我们可以认为Flutter的UI系统包含三棵树：Widget树、Element树、渲染树。他们的依赖关系是：根据Widget树生成Element树，再依赖于Element树生成RenderObject 树。

##### 三棵树介绍：

1. Widget:只是一个配置，里面存储的是有关视图渲染的配置信息，包括布局、渲染属性、事件响应信息等。是不可变的，主要负责描述UI的属性和布局，不负责实际的渲染绘制，所以创建成本很低。
2. Element 是分离 WidgetTree 和真正的渲染对象的中间层， WidgetTree 用来描述对应的Element 属性,同时持有Widget和RenderObject，存放上下文信息，通过它来遍历视图树，支撑UI结构。
3. RenderObject (渲染树)用于应用界面的布局和绘制，负责真正的渲染，保存了元素的大小，布局等信息，实例化一个 RenderObject 是非常耗能的
4. 根据渲染树生成 Layer 树，然后上屏显示，Layer 树中的节点都继承自 `Layer` 类。

我们可以把 Widget 当做菜谱，Element 是配菜，RenderObject 是烧菜和出菜。

当runApp()被调用时，第一时间会在后台发生以下事件：

1）Flutter会构建包含这三个Widget的Widgets树；

2）Flutter遍历Widget树，然后根据其中的Widget调用createElement()来创建相应的Element对象， 最后将这些对象组建成Element树；

3）接下来会创建第三个树，这个树中包含了与Widget对应的Element通过createRenderObject()创建 的RenderObject；

##### 三棵树的作用:

那这三棵树有啥意义呢？简而言之是为了性能，为了复用Element从而减少频繁创建和销毁 RenderObject。因为实例化一个RenderObject的成本是很高的，频繁的实例化和销毁RenderObject对 性能的影响比较大，所以当Widget树改变的时候，Flutter使用Element树来比较新的Widget树和原来的 Widget树。

总结如下：

1）如果某一个位置的Widget和新Widget不一致，才需要重新创建Element;

2）如果某一个位置的Widget和新Widget一致时（两个widget相等或runtimeType与key相等），则只需要修改RenderObject的配置，不用进行耗费性能的RenderObject的实例化工作了；

3）因为Widget是非常轻量级的，实例化耗费的性能很少，所以它是描述APP的状态(也就是configuration)的最好工具；

4）重量级的RenderObject（创建十分耗费性能）则需要尽可能少的创建，并尽可能的复用；

因为在框架中，Element是被抽离开来的，所以你不需要经常和它们打交道。每个Widget的build （BuildContext context）方法中传递的context就是实现了BuildContext接口的Element。

Flutter遵循一个最基本的原则：判断新的Widget和老的Widget是否是同一个类型：

1）如果不是同一个类型，那就把Widget、Element、RenderObject分别从它们的树（包括它们的子 树）上移除，然后创建新的对象；

2）如果是一个类型，那就仅仅修改RenderObject中的配置，然后继续向下遍历；
