#  视图渲染+离屏渲染

##### 视图渲染

UIKit是常用的框架，显示、动画都通过CoreAnimation。
CoreAnimation是核心动画，依赖于OpenGL ES做GPU渲染，CoreGraphics做CPU渲染。
在屏幕上显示视图，需要CPU和GPU一起协作,一部数据通过CoreGraphics、CoreImage由CPU预处理，最终通过OpenGL ES将数据传送到 GPU，最终显示到屏幕。

##### 显示逻辑

1. CoreAnimation提交会话，包括自己和子树（view hierarchy）的layout状态等；
2. RenderServer解析提交的子树状态，生成绘制指令；
3. GPU执行绘制指令；
4. 显示渲染后的数据；

##### 离屏渲染

一个图片显示在屏幕上需要经历三个过程：

第一步：请求jpeg图片数据data（来自磁盘或网络），加载至内存中，即data buffer。
第二步：对data buffer进行解码，解码后的数据称为位图，即image buffer》
第三步：将image buffer传递给GPU的frame buffer，由GPU将图片内容显示在屏幕中。

> 当image buffer需要进行一些额外处理（如圆角、毛玻璃或其他滤镜）并且进行额外处理后无法直接将数据传递至frame buffer进行显示，需要将处理后的数据暂存至offscreen buffer中，再由offscreen buffer传递至frame buffer，最终显示在屏幕上，这个过程就称为离屏渲染。
> offscreen buffer同为内存中的一块连续区域。在对图片进行额外处理时用于存放中间合成数据的区域。

离屏渲染触发条件有两个：

1. **图片（图层）需要额外处理**
2. **数据需要暂存至offscreen buffer**

##### CPU”离屏渲染“

在UIView中实现了drawRect方法，就算它的[函数体](https://zhida.zhihu.com/search?q=函数体)内部实际没有代码，系统也会为这个view申请一块内存区域，等待CoreGraphics可能的绘画操作。

对于类似这种“新开一块CGContext来画图“的操作，有很多文章和视频也称之为“离屏渲染”（因为像素数据是暂时存入了CGContext，而不是直接到了frame buffer）。进一步来说，其实所有CPU进行的[光栅化](https://zhida.zhihu.com/search?q=光栅化)操作（如文字渲染、图片解码），都无法直接绘制到由GPU掌管的frame buffer，只能暂时先放在另一块内存之中，说起来都属于“离屏渲染”。

自然我们会认为，因为CPU不擅长做这件事，所以我们需要尽量避免它，就误以为这就是需要避免离屏渲染的原因。但是[根据苹果工程师的说法](https://link.zhihu.com/?target=https%3A//lobste.rs/s/ckm4uw/performance_minded_take_on_ios_design%23c_itdkfh)，CPU渲染并非真正意义上的离屏渲染。另一个证据是，如果你的view实现了drawRect，此时打开Xcode调试的“Color offscreen rendered yellow”开关，你会发现这片区域不会被标记为黄色，说明Xcode并不认为这属于离屏渲染。

其实通过CPU渲染就是俗称的“软件渲染”，而**真正的离屏渲染发生在GPU**。

##### 离屏渲染的本质

渲染中的常用算法：油画算法。

> 渲染操作都是由CoreAnimation的Render Server模块，通过调用[显卡驱动](https://zhida.zhihu.com/search?q=显卡驱动)所提供的OpenGL/Metal接口来执行的。通常对于每一层layer，Render Server会遵循“[画家算法](https://link.zhihu.com/?target=https%3A//links.jianshu.com/go%3Fto%3Dhttps%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%253A%2F%2Fen.wikipedia.org%2Fwiki%2FPainter%252527s_algorithm)”，按次序输出到frame buffer，后一层覆盖前一层，就能得到最终的显示结果（值得一提的是，与一般桌面架构不同，在iOS中，设备主存和GPU的显存[共享物理内存](https://link.zhihu.com/?target=https%3A//links.jianshu.com/go%3Fto%3Dhttps%3A%2F%2Flink.zhihu.com%2F%3Ftarget%3Dhttps%253A%2F%2Fapple.stackexchange.com%2Fquestions%2F54977%2Fhow-much-gpu-memory-do-iphones-and-ipads-have)，这样可以省去一些数据传输开销）。

复杂的场景，GPU虽然可以一层一层往画布上进行输出，但是无法在某一层渲染完成之后，再回过头来擦除/改变其中的某个部分，因为在这一层之前的若干层layer像素数据，已经在渲染中被永久覆盖了。**对于每一层layer，要么能找到一种通过单次遍历就能完成渲染的算法，要么就不得不另开一块内存，借助这个临时中转区域来完成一些更复杂的、多次的修改/剪裁操作**。

下面我们对之前的三个UIImageView触发离屏渲染的情况进行分析，
第一个UIImageView的设置：

```objective-c
//1.UIImageView 设置图片+背景色;
imgView1.image = image;// contents
imgView1.backgroundColor = [UIColor systemTealColor];// backgroundColor
imgView1.layer.cornerRadius = 50;
imgView1.layer.masksToBounds = YES;
```

即UIImageView的layer有backgroundColor和contents，masksToBounds = YES后，contents会执行圆角操作，因此，backgroundColor和contents都需要执行圆角操作，之后进行叠加合并最终显示到屏幕上。这个过程中存在多个处理操作，*渲染流水线无法找到单次遍历就能完成渲染的算法*，因此数据无法直接传递frame buffer从而触发离屏渲染。从图中可以看出只有UIImageView的四个顶点区域发生了离屏渲染。

第二个UIImageView的设置：

```objective-c
//2.UIImageView 只设置图片,无背景色;
imgView2.image = image;// contents
imgView2.layer.cornerRadius = 50;
imgView2.layer.masksToBounds = YES;
```

即UIImageView的layer只有contents，masksToBounds = YES后，contents会执行圆角操作，最后显示到屏幕上。这个过程中存在*渲染流水线单次遍历就能完成渲染的算法*，因此数据直接传递frame buffer，避免了offscreen buffer的使用，从而没有触发离屏渲染。

第三个UIImageView的设置：

```objc
//3.UIImageView 仅设置背景色,无图片;
imgView3.backgroundColor = [UIColor systemTealColor];// backgroundColor
imgView3.layer.cornerRadius = 50;
imgView3.layer.masksToBounds = YES;// yes or no均不影响结果
```

即UIImageView的layer只有backgroundColor，设置cornerRadius后backgroundColor会执行圆角操作，最后显示到屏幕上。这个过程中存在*渲染流水线单次遍历就能完成渲染的算法*，因此数据直接传递frame buffer，避免了offscreen buffer的使用，从而没有触发离屏渲染。

##### 离屏渲染的坏处

切换上下文、开辟新内存都是耗时的CPU操作，CPU 占用越高，耗电越快，响应速度越慢。
如果性能损耗负担过大，如在tableView或者collectionView中，滚动的每一帧变化都会触发每个cell的重新绘制，16ms（60hz）内无法完成渲染则会导致掉帧。

##### 如何高效地使用离屏渲染

除了尽量避免离屏渲染外，势必不可避免的有离屏渲染发生的场景，那么如何高效地使用离屏渲染呢？
答案是[栅格化](https://zhida.zhihu.com/search?q=栅格化)，在CALayer中有一个shouldRasterize属性，开启后layer会启动栅格化。

1. shouldRasterize会必然产生一次离屏渲染，因为开启了新内存空间来复用结果。
2. layer的内容（包括子layer）必须是静态的，layer非静态意味着需要重新渲染，那么缓存就会失效，每一帧都开辟新内存区域即离屏渲染，这正是渲染流水线中极力避免的。我们可以利用xcode中的“Color Hits Green and Misses Red”的选项，查看缓存的使用是否符合预期。
3. [缓存大小](https://zhida.zhihu.com/search?q=缓存大小)限制 <= 屏幕总像素的2.5倍
4. 缓存有效期 <= 100ms，超过100ms未被使用则视为失效，从而丢弃。

shouldRasterize在另一个场景中也可以使用：如果layer的子结构非常复杂，渲染一次所需时间较长，同样可以打开这个开关，把layer绘制到一块缓存，然后在接下来复用这个结果，这样就不需要每次都重新绘制整个layer树了。



##### 离屏渲染的常见场景

1. 圆角cornerRadius+clipsToBounds
2. 阴影shadow
3. 组透明度group opacity
4. 遮罩mask
5. 毛玻璃效果UIBlurEffect
6. 其他还有一些，如allowsEdgeAntialiasing

### 优化

- 即刻大量应用AsyncDisplayKit(Texture)作为主要渲染框架，对于文字和图片的异步渲染操作交由框架来处理。关于这方面可以看我[之前的一些介绍](https://link.zhihu.com/?target=https%3A//medium.com/jike-engineering/asyncdisplaykit%E4%BB%8B%E7%BB%8D-%E4%B8%80-6b871d29e005)

- 对于图片的圆角，统一采用“precomposite”的策略，也就是不经由容器来做剪切，而是预先使用CoreGraphics为[图片裁剪](https://zhida.zhihu.com/search?q=图片裁剪)圆角

- 对于视频的圆角，由于实时剪切非常消耗性能，我们会创建四个白色弧形的layer盖住四个角，从视觉上制造圆角的效果

- 对于view的圆形边框，如果没有backgroundColor，可以放心使用cornerRadius来做

- 对于所有的阴影，使用shadowPath来规避离屏渲染

- 对于特殊形状的view，使用layer mask并打开shouldRasterize来对渲染结果进行缓存

- 对于模糊效果，不采用系统提供的UIVisualEffect，而是另外实现模糊效果（CIGaussianBlur），并手动管理渲染结果

  

[iOS开发-视图渲染与性能优化](https://www.jianshu.com/p/748f9abafff8)

[关于iOS离屏渲染的深入研究](https://zhuanlan.zhihu.com/p/72653360)

[iOS开发解析什么是离屏渲染](https://zhuanlan.zhihu.com/p/378945322)