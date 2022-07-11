# iOS 埋点

目前，业界主流的埋点方式主要有如下三 种。

#### 代码埋点

应用程序集成埋点SDK后，在启动时初始化埋点SDK，然后在某个事件发生的时候调用埋点SDK提供的方法来触发事件。

##### 优点

- 可以精准控制埋点的位置
- 可以方便、灵活地自定义事件和属性
- 可以采集更丰富的和业务相关的数据
- 可以满足更加精细化的分析需求

##### 缺点

- 前期埋点的成本相对较高
- 若分析需求或事件发生变化，则需要修改应用程序埋点并发版

#### 全埋点

全埋点也叫无埋点、无码埋点、无痕埋点、自动埋点，指无须应用程序开发工程师写代码或者只写少量的代码，即可预先自动收集用户的所有或者绝大部分的行为数据，然后根据实际的业务分析需求从中筛选出所需的数据并进行分析。

##### 全埋点可以采集的事件

- **应用程序的启动事件**（`$AppStart`）
  · 冷启动：应用程序被系统终止后，在这种状 态下启动的应用程序。
  · 热启动：应用程序没有被系统终止，仍在后 台运行，在这种状态下启动的应用程序
- **应用程序退出事件**（`$AppEnd`）
  ·双击Home键切换到其他应用程序。
  ·单击Home键让当前应用程序进入后台。
  ·双击Home键并上滑，强杀当前应用程序。
  ·当前应用程序发生崩溃导致应用程序退出。
- **页面浏览事件**（`$AppViewScreen`）
  应用程序内的页 面浏览事件，对于iOS应用程序来说，就是指切换 不同的`UIViewController`。
- **控件单击事件**（`$AppClick`）
  控件点击事件，比如 点击`UIButton`、`UITableView`等。
- **应用程序崩溃事件**

##### 优点

- 前期埋点成本相对低
- 若分析需求或事件设计发生变化，无须应用程序修改埋点并发版
- 可以有效地解决历史数据回朔问题

##### 缺点

- 很难做到全面的覆盖
- 无法自动采集和业务相关的数据
- 无法满足更精细化的分析数据
- 各种兼容性能方面问题

#### 可视化埋点

可视化埋点也叫圈选，是指通过可视化的方式进行埋点

##### 可视化埋点一般有两种应用场景

- 默认情况下，不进行任何埋点，然后通过可视化的方式指定给哪些控件进行埋点（指定埋点）
- 默认情况下，全部进行埋点，然后通过可视化的方式指定不给哪些控件进行埋点（排除埋点）。

##### 优缺点

可视化埋点的优点和缺点，整体上与全埋点的优点和缺点类似。
