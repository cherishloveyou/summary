# iOS 中 alloc init 和new的区别

**在重写了allocWithZone的前提下，调用alloc函数时，在callAlloc函数内部最终会执行[cls allocWithZone:nil];**

**在重写了allocWithZone的前提下，调用new函数，在callAlloc函数内部最终会执行[cls alloc]，然后再走一遍上面分析的alloc的逻辑，最终执行 [cls allocWithZone:nil]**

**在重写了allocWithZone函数后，alloc init 和 new的区别只是在于：是否显示的调用allocWithZone函数。使用new时会在callAlloc中调用alloc，从而隐式调用allocWithZone:函数。如果没有重写allocWithZone:函数，使用alloc init同new没有任何区别**



**区别就是使用new只能默认init进行初始化，alloc方式可以使用其它的init开头的方法进行初始化。**

**还有一点,在其实默认的init方法中 , 什么都没有做,直接返回了self , 所以,如果没有重写init的话, [Class alloc] 和 [[Class alloc]init] 是等价的.**



`init`目的是为开发者提供工厂设计初始化方法，方便初始化设置。
