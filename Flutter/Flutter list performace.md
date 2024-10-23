# Flutter列表性能优化

### 一.内存优化

#### 1.识别出消耗多余内存的图片

`Flutter Inspector`：点击`Highlight oversized images`，会解码出那些大小超过展示大小的图片，并且系统会将其倒置突出显示。

此类图片会导致性能不佳，尤其是在低端设备上，并且当有许多图片(例如在列表视图中)时，这种性能影响可能会增加。

如果图片使用的大小至少比要求的大128KB，则图片被视为太大。

修复图片：
 <1>.解决此问题的最佳方法是调整图片资源文件的大小，使其更小。
 <2>.如果这是不可能的，可以在加载图片时使用`cacheWidth`和`cacheHeight`参数，这使引擎以指定大小解码该图片，并减少内存使用(解码和存储仍然比缩小图片资源本身更昂贵)。

#### 2.避免在快速滚动时加载图片

使用官方的`ScrollAwareImageProvider`组件，会在列表快速滑动时中断图片的下载和解码，从而减少不必要的内存占用。

虽然这种方法不能100%解决图片加载时 OOM 的问题，但是很大程度优化了列表中的图片内存占用，官方提供的数据上看理论上可以在原本基础上节省出 70% 的内存。

#### 3.不要使用`shrinkWrap`为`true`

`Flutter`团队将从滑动控件里面弃用`shrinkWrap`属性，因为团队觉得现阶段的开发人员大多数时候不知道它的实际含义，只是单纯使用它解决问题，在使用过程中容易出现错误的性能损耗而不自知。

`shrinkWrap:true`的时候，在滑动控件内部会采用一个特殊的`ShrinkWrappingViewport`窗口进行实现，`ShrinkWrappingViewport`是调整自身大小去匹配主轴方向中`Item`的大小，这种收缩的行为成本会变高，因为窗口大小需要通过`Item`去确定。

`shrinkWrap:true`会将列表所有的`item`都构建完成，尽管他们还远没有在 `ViewPort`展示出来，所以`shrinkWrap`让`ListView`失去了懒加载的作用，导致内存性能问题。

#### 4.`addAutomaticKeepAlives`和`addRepaintBoundaries`

`addAutomaticKeepAlives:true`表示将`item`包裹在`AutomaticKeepAlive`组件中，当`item`滑出屏幕时不会被GC(垃圾回收)，会使用`KeepAliveNotification`来保存其状态，从而在再次出现在屏幕的时候能够快速构建。这其实是一个拿空间换时间的方法，会造成一定程度的内存开销。

`addRepaintBoundaries:true`表示将`item`包裹在`RepaintBoundary`组件中，当列表滚动时，包裹在`RepaintBoundary`中的`item`可以避免重绘，当`item`重绘的开销特别小时(比如一个颜色块，一个较短的文本)，不添加`RepaintBoundary`反而更高效。

若禁用`addAutomaticKeepAlives`和`addRepaintBoundaries`，会在一定程度上优化内存，但是这会影响渲染性能，需要在内存和渲染性能之间权衡。

#### 5.列表中的元素尽可能使用`const`修饰

使用 const 相当于将元素缓存起来实现共用，若列表元素某些部分一直保持不变，那么可以使用 const 修饰。

### 二.滚动优化

#### 1.指定列表项的固定高度

`ListView`的`itemExtent`属性用于指定`item`的固定高度，它告诉`ListView`在构建列表时使用固定大小的缓冲区。

如果知道`item`的固定高度，可以讲`itemExtent`设置为该值，有利于`Flutter`更有效的管理列表的布局，`并且在滚动时可以提高性能`。

#### 2.合理使用缓冲区

`ListView`可以通过设置`cacheExtent`来设置预先加载的内容大小。通过预先加载可以提升`view`渲染的速度。但是这个值需要合理设置，并非越大越好。因为预加载缓存越大，对页面整体内存的压力就越大。

### 三.渲染优化

#### 1.`setState`状态刷新位置尽量放置于视图树的低层级，推荐使用`Selector`构建需要刷新的组件，以维持最小刷新范围。

#### 2.对于频繁更新的控件(如动画)，使用`RepaintBoundary`隔离它，创建单独`layer`减少重绘区域。

#### 3.使用图片替换半透明效果。

#### 4.减少`SaveLayer`(`ShaderMask`、`ColorFilter`、`Text Overflow`)、`ClipPath`的使用，提高渲染线程性能。

#### 5.避免使用`Opacity widget`，尤其是在动画中避免使用，用`AnimatedOpacity`或`FadeInImage`进行代替。







https://www.jianshu.com/p/dd29262491fd
