## iOS组件化

组件化通讯方案目前主流的主要有以下三种方式

- 1、`URL`路由

- 2、`target-action`

- 3、`protocol`匹配

  **URL 路由的优点**

  - 极高的动态性，适合经常开展运营活动的app，例如电商
  - 方便地统一管理多平台的路由规则
  - 易于适配URL Scheme

  **URl 路由的缺点**

  - 传参方式有限，并且无法利用编译器进行参数类型检查，因此所有的参数都是通过字符串转换而来
  - 只适用于界面模块，不适用于通用模块
  - 参数的格式不明确，是个灵活的 dictionary，也需要有个地方可以查参数格式。
  - 不支持storyboard
  - 依赖于字符串硬编码，难以管理，蘑菇街做了个后台专门管理。
  - 无法保证所使用的的模块一定存在
  - 解耦能力有限，url 的”注册”、”实现”、”使用”必须用相同的字符规则，一旦任何一方做出修改都会导致其他方的代码失效，并且重构难度大

  ### target-action

  这个方案是基于OC的runtime、category特性动态获取模块，例如通过`NSClassFromString`获取类并创建实例，通过`performSelector + NSInvocation`动态调用方法

  其主要的代表框架是[casatwy的CTMediator](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fcasatwy%2FCTMediator)

  其实现思路是：

  - 1、利用分类为路由添加新接口，在接口中通过字符串获取对应的类
  - 2、通过runtime创建实例，动态调用实例的方法

**优点**

- 利用 `分类` 可以明确声明接口，进行编译检查
- 实现方式`轻量`

**缺点**

- 需要在`mediator` 和 `target`中重新添加每一个接口，模块化时代码较为繁琐
- 在 `category` 中仍然引入了字符串硬编码，内部使用字典传参，一定程度上也存在和 URL 路由相同的问题
- 无法保证使用的模块一定存在，target在修改后，使用者只能在运行时才能发现错误
- 可能会创建过多的 target 类

### protocol class

protocol匹配的`实现思路`是：

- 1、将`protocol`和对应的`类`进行`字典匹配`
- 2、通过用`protocol`获取`class`，在`动态创建实例`

protocol比较典型的三方框架就是[阿里的BeeHive](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Falibaba%2FBeeHive)。`BeeHive`借鉴了Spring Service、Apache DSO的架构理念，`采用AOP+扩展App生命周期API`形式，将`业务功能`、`基础功能`模块以模块方式以解决大型应用中的复杂问题，并让`模块之间以Service形式调用`，将复杂问题切分，以AOP方式模块化服务。

**BeeHive 核心思想**

- 1、各个模块间调用从直接调用对应模块，变成调用`Service`的形式，避免了直接依赖。
- 2、App生命周期的分发，将耦合在`AppDelegate`中逻辑拆分，每个模块以微应用的形式独立存在。

**优点**

- 1、利用接口调用，实现了参数传递时的类型安全
- 2、直接使用模块的protocol接口，无需再重复封装

**缺点**

- 1、用框架来创建所有对象，创建方式不同，即不支持外部传入参数
- 2、用`OC runtime`创建对象，不支持swift
- 3、只做了`protocol` 和 `class` 的匹配，不支持更复杂的创建方式 和依赖注入
- 4、无法保证所使用的protocol 一定存在对应的模块，也无法直接判断某个protocol是否能用于获取模块