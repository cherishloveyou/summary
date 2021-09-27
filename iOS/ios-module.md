## Module化

单一Target比较简单，在Build setting中设置bridging header，使Swift可以访问OC，Swift也会编译出-Swift.h来给OC调用，这样就可以实现Swift和OC的混编开发。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kTR2HQBMfv0BQjFOVdnAV4CWorB9MXfhsakyWOlYMy7CH3IAbUu7tz4eRde3K0QDMN72LQHYBjQdAq9ia7ibsTsQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

需要注意的是，如果使用bridging header的方式就无法开启Swift版本兼容功能。所以，我们更推荐使用的方式是OTHER_SWIFT_FLAGS标记-import-underlying-module，配合HEADER_SEARCH_PATHS来使用Swift访问OC的接口。在下文中的多Target混编部分会对此有详细的讨论。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kTR2HQBMfv0BQjFOVdnAV4CWorB9MXfhk0WqMafFNuQiaus13q726X7T2qqdvDSyOgQFmAhibyfud7MFbgn2Wlcw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**多Target混编**

快手海外已经启动组件化的改造工作，组件化以后，每个Pod组件都使用独立的Target编译，这就要求我们必须解决多Target下的Swift/OC混编问题。这个过程里我们碰到了不少问题，总结下来，有以下几道坎需要迈过。



**第一道坎 OC组件Clang module化**

Swift组件若依赖其它含OC代码的Pod组件，那些组件必须使用Clang module来组织，这个在戴铭老师的《A站的 Swift 实践——上篇》一文中也提到过。Clang module（以下简称Module)定义了一种新的OC模块向外暴露公开接口的方式，Clang编译器在v13版本添加了新的编译参数来支持这种方式。这里不对Module的细节做进一步展开，有兴趣的读者可以去查阅*Modules-Clang 13 Document[1]。*

从代码实践上说，Module带来了两个好处：

第一，避免了宏对代码上下文的污染。

第二，提高了编译的效率。在每次编译过程中，每个Module只会被加载一次，并且编译器会将编译好的部分缓存起来，避免了多次引入加载相同头文件带来的耗时问题，所以使用module的编译效率会更高，被复用越多的组件，module化后的收益越明显。

那么在实践中，如何对Cocoapods组件进行Module化改造呢？

打开Module的方法很简单，在Podfile中把pod参数设置为: modular_header => true，pod install执行之后便会自动生成modulemap与umbrella.h。但是**Module和非Module的传统OC组件混合编译时，会存在很多问题：**

其一，module具有传递性，如果一个module组件A所暴露的头文件中导入了组件B的头文件，那么组件B也必须是module。

其二，引用module组件的头文件时，必须采用#import <Framework/FrameworkHeader.h>和@import Framework.FrameworkHeader 的方式，而不像传统header search path那样可以直接使用双引号、尖括号头文件的方式。

如果module中含有Swift代码并且构建产物不是Framework，建议在 build phase 中添加 script，处理编译后需要做两件事：一是将编译生成的 -Swift.h 拷贝到 public header 路径以供其他组件使用；二是将编译生成的swiftinterface追加到modulemap目录下以供其他组件使用。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kTR2HQBMfv0BQjFOVdnAV4CWorB9MXfhoUrHOkQMUPWWU0SeNapYVp7u8UoWr9KO5btDtcTKMMFgUSZpNUmicMw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

如上图所示，一个Module需要对外提供两部分内容，其中modulemap + umbrella + header file为OC 的对外接口，swiftinterface为Swift的对外接口。将组件module化后，必须通过modulemap才能真正地以module的形式依赖它。假设A组件依赖module B组件，需要做如下配置：



○ 如果A组件中的OC代码依赖B module，则需要在A组件的 OTHER_CFLAGS 中添加 -fmodule-map-files="${B组件的modulemap路径}"。

○ 如果A组件中的Swift代码依赖B module，则需要在A组件的 OTHER_SWIFT_FLAGS 中添加 -Xcc -fmodule-map-files="${B组件的modulemap路径}"。

○如果B module是Framework，需要在A组件的 FRAMEWORK_SEARCH_PATHS  中添加B组件Framework路径，而且modulemap也需要使用Framework中的modulemap。

○如果B module是静态库，需要在A组件的 HEADER_SEARCH_PATHS 中添加B组件的public header路径，而且modulemap必须能正确找到这些Header文件。

○ 如果B组件的Swift代码被A组件依赖，则需要在A组件的 SWIFT_INCLUDE_PATHS 中配置B组件的 swiftinterface 搜索路径。



值得注意的是：A组件通过modulemap去查找B组件的头文件，**在modulemap和umbrella中引入的头文件必须都在header search path中可以被找到，而且header search path中所找到的头文件必须与modulemap中导入的头文件是同一个文件。**



**第二道坎 处理难以Module化的Pod组件**

在实际工程中，很多OC组件由于自身的代码语言和历史原因，存在需要改造甚至无法Module化的情况，如果不解决这个问题，就难以彻底在实际工程中实现Swift和OC的和平共处。在最初的解决方案中，我们不得不牺牲性能来让这部分OC完全使用源码参与编译。我们的同学经过细致的分析，最终攻克了这个拦路虎。以下我们将分不同类型来介绍相应的解决方案。



***C++组件***

如前文所述，Module的依赖具有传递性，Pods的直接和间接依赖都必须是module，这就会造成Module的连锁反应。音视频、生产Pod组件包含很多C++组件，如果连锁反应遇到了这些C++组件，我们将要面对OC、C++、Swift这三种语言混编的异常复杂的难题。而且，LLVM对于C++的module支持还处于Beta阶段，所以也并不稳定。

我们对C++模块module化进行了一些尝试，但是发现需要投入太多的成本，所以最终决定放弃对这条马奇诺防线进行攻坚，选择绕过它。

1.阻断module对C++组件的依赖。简单来说就是删除掉podspec中的dependency声明，然后再注入header search path，使得module没有dependency依然可以引用头文件且不会编译报错（为方便起见，后文我们称之为“隐性依赖”），这样一来，在pod install阶段就不会强制要求那些暂时不能够module化的C++组件也必须是module化的。

2.对于那些少数无法绕开的C++组件，我们的解决方案是：在C++组件和Swift组件增加一个OC中间层适配组件。这虽然会带来额外的开销，每当C++外部接口变化时，相应的适配组件也需要改动，但好在这部分组件相当少，维护成本是可以接受的。

***头文件层级复杂的Pod组件***

为了支持二进制预编译库（下文中「二进制化混编」部分会提到），我们将会把module化的组件打包为XCFramework，组件的public header会被打平多层目录结构统一放在Framework的Headers目录下。一些组件的public header是有自己的目录结构的，这就导致调用方在导入头文件时，在源码状态下需要写全路径，而在二进制状态下则不能写路径，所以在引入这个组件的源码和二进制时，调用方就必须使用不同的引用方式，这令开发者十分头疼。

要解决这个问题，首先最容易想到的办法就是使用预编译宏，但是这个方法在Swift中似乎并不那么方便。我们采用的办法是：在源码状态下将Pods/Headers/Public路径中的该组件的头文件软链目录结构打平，同时修改umbrella.h，去掉其中的目录层级，这样就可以做到打包的framework与源码都采用一致的没有目录层级的统一方式来引入头文件。

***修改Pod组件代码***

将头文件的层级打平，之前代码中通过路径引用的头文件（#import "xxx/xxx/xxx/xxx.h"）就都会报错，那么我们就需要对组件中的代码做修改。目前采用的方法是在pod install阶段对源文件进行文本替换，其中打平头文件的替换是自动通过正则匹配进行的，其他的各种代码问题则是通过配置指定Pod和文件来替换的。

![图片](https://mmbiz.qpic.cn/mmbiz_png/kTR2HQBMfv0BQjFOVdnAV4CWorB9MXfhfn6Em1frPE9HOkt9RHpqRxMEJib6D4Ou4P5l5rfsfWjJXLLAEVN6kXQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



**第三道坎Swift组件的二进制兼容**

Swift没有头文件，因此编译Swift代码会生成swiftmodule文件，它包含了Swift接口信息的二进制文件，其功能类似于头文件。但残酷的现实是：**不同编译器生成的swiftmodule是不一样的，而且他们互不兼容**，这也就导致了不同版本Xcode编译生成的Swift二进制库无法通用*（Evolving Swift On Apple Platforms After ABI Stabilit[2],* *ABI Stability and More[3]）*。从Swift 5.1版本开始，这个问题得到了改变，编译器新增了一个编译参数：BUILD_LIBRARY_FOR_DISTRIBUTION，打开它就会在编译时生成swiftinterface文件，内容和作用与swiftmodule相同。但是二者的一个关键区别是，swiftinterface是文本格式，而不是二进制格式。后续版本的Swift编译器也可以导入旧版生成的swiftinterface文件，这样就解决了Swift二进制兼容的问题。

需要指出的是，这个属性与单Target混编里使用桥接文件(bridging header)的方式是冲突的。这时，Swift想要引用同模块的OC文件就无法再依靠桥接文件了，而是通过隐式的方式来引入当前模块内所有的OC文件，Xcode是通过一个新的编译参数import-underlying-module来实现的。



**第四道坎 二进制化混编**

为了提高编译速度，我们一般会将Pod组件预先编译成二进制静态库，连同头文件、Podspec和其它支持文件打包成zip文件存放在远程服务器上；然后在Pod install时，将包含二进制化库的zip从远程拉取到本地，解压后一起联合编译链接。

上文提到，Swift组件在swiftinterface若引入了其它含OC代码的Pod组件，那些组件必须使用Module（Clang module和swift module）来组织。因此，为了让Swift/OC混编支持预编译的二进制库，我们必须使用Module的结构来改造原有打包结构。对于这些问题，Apple早已提供了解决方案：使用静态Framework来封装静态库（实践中是使用XCFramework格式，因为它可以自动适配发布平台的指令集）。生成静态Framework时，可以选择根据公开接口和Swift文件生成对应的Module信息，包含modulemap、swiftinterface等其它Pod组件引用需要的必要结构。这样，就可以稳定支持Swift/OC二进制混编了。

在实践中，我们需要稳定支持二进制和源码Pod组件的混合编译，因为很多本地库是以源码化的形式存在的。以下图的源码二进制依赖关系为例，组件A依赖二进制Framework B， B是一个Swift组件，B又依赖源码组件C。前文提到Swift组件依赖的Pod组件必须打包成Framework，不管对二进制化的Pod组件还是源码都是一样的要求。我们初期只改造了预编译的Framework，而忽视了仍然使用头文件引用的源码形式Pod组件，导致走了不少弯路。这种情况下，编译器的报错令人非常困惑，开发者会抱怨找不到一堆A依赖的Module。由于正确的提示淹没在了一堆错误提示下，刚开始很难找到实质的问题。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/kTR2HQBMfv0BQjFOVdnAV4CWorB9MXfhdsd155qk8zwkC132wbeiaFLhLBic2q6sTtamccictL0t0iaC4ib40DOfKzA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)**实践中遇到的问题**

前文提到的解决方案，已经能覆盖大部分Swift/OC混编的问题。不过，在实践中，我们还是遇到了一些特殊的棘手问题，主因都与代码不规范有关，包括PODSPEC和头文件引用不规范。在历史包袱较重的大型项目里，这些不规范的问题不是一朝一夕能够彻底解决的，因此我们不得不采取一些非常规的方法。

**Framework search path与Header search path的冲突问题**

我们制作二进制包的时候，会将module化的组件打包成XCframework，并将打包生成的XCframework文件设置为module组件的vendored_framework。为了让framework的头文件可以被搜索到，组件化工具提供了VendoredFrameworkSearchPath服务，会自动把组件的vendored_framework中的头文件软链到${PODS_ROOT}/Headers/Public目录下，这样做是为了让代码中的双引号import也可以找到正确的头文件。但这样做也会有副作用：Swift依赖的组件必须使用module，而module是通过modulemap与umbrella.h来寻找头文件的。如果从search header path中找到了与modulemap中不同的头文件，编译器也会报错（即使它们都是同一个源文件的符号链接）；另一种出错原因是从header search path中找到的头文件并不是通过module的方式引入而报错。在最初的改造版本中，这就像是个定时炸弹，随时可能会引爆我们的编译问题，每次编译不到最后完成，我们都胆战心惊。在经过Demo工程项目试验后，我们最终采取了以下解决方案：



1.禁用VendoredFrameworkSearchPath服务，不再在${PODS_ROOT}/Headers/Public中生成已经在framework中存在的头文件，让module的头文件只能通过modulemap引入，不能通过header search path被找到。

2.把framework的路径写入调用方framework search path中，让调用方可以正确找到framework。

3.把-fmodule-map-file的路径写入调用方的other cflags中，让调用方在编译OC的时候可以找到framework的modulemap。

4.把-Xcc -fmodule-map-files写入调用方的other swift flags中，让调用方在编译Swift的时候可以找到framework的modulemap。

5.修正其它依赖本组件公开头文件的引用方式，即把以下非标的方式：

○ #import "ModuleHeader.h"

○ #import "Module/ModuleHeader.h"

○ #import "Module/DirA/DirB/ModuleHeader.h"etc

都改为如下的方式：

○ #import <Module/ModuleHeader.h>



**Module中使用宏的问题**

Module为了消除组件中的宏对调用代码上下文的污染，禁用了因为条件而改变的宏定义，也就是说：被依赖组件的宏如果会因为条件而修改，则无法被调用它的组件使用到。解决的办法是直接把缺少的宏设置给调用方Target。因为这些宏虽然对于被依赖的组件是动态的，但是对于依赖它的Target来说却是静态的，我们直接给依赖它的Target设置上宏并不会有什么问题。



**FakeFramework**

**解决依赖关系与编译顺序的问题**

在上文“第四道坎”中，提到了要保证Swift组件的源码、二进制的自由转换，需要它所依赖组件的构建产物为Framework，理论虽然如此，但在实际应用过程中事情往往并不会那么一帆风顺：它对组件的编译顺序是有严格要求的，必须先完成对Framework Target的编译才能生成Framework，依赖这个Framework的组件才能够正确编译。如果先编译了上层依赖它的组件，就会出现找不到Framework的错误。

事实上这个问题可以通过在podspec中设置dependency来避免，被依赖的组件会先执行编译，但是现实情况往往会更加复杂：由于代码的历史原因，组件之间存在着一定的依赖环，要解除这些依赖环尚需要一些时间，所以暂时就无法通过在podspec中设置dependency来解决这个问题。

我们在理解了这个问题的本质原因后，构建了一个比较Tricky的方法：在Pod install的时候，由脚本在${PODS_ROOT}/Frameworks目录下生成module组件的framework文件结构，在.framework目录下包含Headers、Modules和PrivateHeaders，并且将公开头文件符号链接到Headers目录中去。在Framework组件编译完成后，添加一个Xcode的Build phase script将.a、-Swift.h和swiftmodule符号都链接进.framework目录中。这样，一个假的Framework文件结构就形成了。我们只需要为依赖它的组件设置Framework search path，这样即使编译顺序不正确（即依赖的Framework组件先编译，而被依赖的后编译），也不用担心找不到Framework中module map和相关文件，因为这些文件早在编译前的Pod install阶段就生成到${PODS_ROOT}/Frameworks目录中了。

需要注意的是上述方法仅仅对OC有效，Swift的“头文件”swiftmodule必须通过编译才能生成，无法在Pod install阶段直接生成。所以，所有Swift组件都要被正确的添加到其它依赖它组件的podspec的dependency列表中，由于目前目录中的Swift组件比较少，这样的工作量是可以接受的。



**未来展望**

**一、基础库规范适配处理**

为了能够正确引用Module，我们在上文中提到通过配置和脚本。在Pod install期间，直接在代码上修改基础库的引用方式，这是一种hack的方式，且当基础库发生变化时，必须持续维护这个修改列表。因此我们正在推动公司内部中台共建以及基础库的规范引用:

○ 必须采用#import <Module/ExternalHeader.h>引用外部模块，#import "InternalHeader.h"；

○ 推荐podspec配置打平公开头文件的组织方式。

除引用规范之外，其它适配Swift的还包括：

○ 使用非值宏(e.g. PRO_APP而不是PRO_APP=1）。



**二、规范公开头文件**

前文提到Module引用，会检查所有公开头文件路径，也就是组件A的modulemap引入的头文件的直接和间接依赖都会检查。而目前我们在公开头文件引入了很多不必要的外部头文件，这些头文件并不一定被本头文件依赖，我们可以通过前向声明的方式避免引入，只需在实现文件引入即可，这样可以减少检查的范围，提高编译的速度。



**三、提高Module化组件占比**

前文提到提高Module占比有利于提高编译性能，尤其是打TF包，Module化的组件可以使用二进制方式编译，将明显地提升编译打包速度。为了提高占比，可以分以下三步来推进：

首先，梳理已经没有循环依赖且对C++组件无间接依赖的组件，直接Module化。

其次，梳理有循环依赖且对C++组件无间接依赖的组件，完成组件解耦，解决组件间的循环依赖后，再Module化。

最后，持续跟踪社区对C++ module化的工作，一旦进入稳定版本，开始尝试接入。



[BeeHive框架全面解析——iOS开发主流方案比较](https://xiaozhuanlan.com/topic/4052613897)

[iOS-底层原理 35：组件化（二）组件间通讯方式](https://www.jianshu.com/p/94f5e90f3e29)

[网易新闻iOS工程组件化实践](https://mp.weixin.qq.com/s?__biz=MzUxODg0MzU2OQ==&mid=2247490694&idx=1&sn=17eccff062d5ce7d50ea03f508810bc5&scene=21#wechat_redirect)

[快手海外Swift/ObjC混编与二进制化工程实践](https://mp.weixin.qq.com/s?__biz=MzkxOTI0MTA2OA==&mid=2247487414&idx=1&sn=fb86c4740bb7e6a5c1cd041879f04fc1&scene=21#wechat_redirect)

