## 对象

### id和NSObject *的区别?

- `id`是`struct objc_object`结构体指针，可以指向任何`OC对象`，理解为万能指针
- `NSObject *`是`NSObject`类型的指针

### id类型为什么无法用点语法

`id类`型无法确定所指的类型，无法查找`setter和getter`。

### OC中 Null、nil、Nil、NSNull

- `NULL`，指针是空值，用来判断`C指针`
- `nil`是指一个`OC对象`，指针为空
- `Nil`是指一个`OC类`为空
- `NSNull`则用于填充集合元素；这个类只有一个方法`null`

### ==、 isEqualToString、isEqual区别？

- 1、`==`比较的是两个内存地址是否相同。
- 2、`isEqualToString`比较的是两个字符串是否相等。
- 3、`isEqual` 判断两个对象在类型和值上是否都一样。

### isEqual和hash的关系

- `isEqual` 比较对象是否相等，可以重写判断地址指针，判断类型，判断属性
- `hash`，在对象添加到 `NSSet` 和 `NSDictionary` 的时候调用，查找效率高
- `isEqual`返回`YES`，`hash`一定相等。
- `hash`相等，`isEqual`不一定相等，`hash`可能冲突

### iOS中内省的几个方法？

- `isMemberOfClass` 对象是否是某个类型的对象
- `isKindOfClass` 对象是否是某个类型或某个类型子类的对象
- `isSubclassOfClass` 某个类对象是否是另一个类型的子类
- `isAncestorOfObject` 某个类对象是否是另一个类型的父类
- `respondsToSelector` 是否能响应某个方法
- `conformsToProtocol` 是否遵循某个协议

### 深拷贝和浅拷贝

- 1、浅拷贝是引用的复制
- 2、深拷贝是值的复制，会新建一个对象
- 3、`copy`出来的对象是不可变类型，`mutableCopy`出来的对象是可变类型

### 阻止编译器自动合成，有哪几种方式呢？

- `@dynamic`
- `readonly`关键字

### NSString类型为什么要用copy修饰 ？

- 主要防止`NSString`被修改。
- 当`NSString`的赋值来源是`NSString`时，`strong`和`copy`作用相同。
- 当`NSString`的赋值来源是`NSMutableString`，`copy`会做深拷贝重新生成一个新的对象，修改赋值来源不会影响`NSString`的值。

### int * const p 、int const *p 、const int *p 、 const int * const p

- `int const *p 、const int *p` 一样的，不能改数值，可以改指向
- `int * const p` 值可以改，指向不可以改
- `const int * const p` 数值和指向都不可以改

### `@public、@protected、@private、@package` 声明各有什么含义？

- `@public` 任何地方都能访问;
- `@protected` 该类和子类中访问,是默认的;
- `@private` 只能在本类中访问;
- `@package` 本包内使用,跨包不可以。

### `@synthesize` 和 `@dynamic` 分别有什么作用

- `@property`默认使用`@syntheszie， var = _var;`
- `@synthesize`：如果没有手动实现`setter & getter`，编译器会自动加上这两个方法。
- `@dynamic`：告知编译器， `setter & getter` 由程序员实现，不需要自动生成。如果没有实现`setter & getter`，编译的时候不会报错，但是使用到的时候会因找不到方法实现crash

### `static`有什么作用?

- `static`修饰的变量可以被本文件此语句之后的所有函数访问，外部文件无法访问
- `static`的变量能且只能被初始化一次，默认为0
- `static`修饰的函数可以被类方法访问

## 类

### `@property`的本质是什么？`ivar、getter、setter`是如何生成并添加到这个类中的？

- 声明属性，系统给当前类自动生成一个带下划线的成员变量，`setter、getter`方法。`@property = ivar（实例变量） + getter + setter`
- 通过自动合成（`autosynthesis`），编译器在编译期自动编写访问这些属性所需的方法。除了生成 `getter、setter`外，编译器自动向类中添加适当类型的实例变量，并且在属性名前面加下划线，以此作为实例变量的名字。

### `final`关键字有什么用

- 这个关键字可以用来修饰 `class、func、var`。
- 被修饰的对象无法被继承，无法被重写。

### 实例方法和类方法的区别？

- 1、实例方法能访问成员变量。
- 2、类方法中必须创建或者传入对象才能调用对象方法。
- 3、实例方法存储在类对象的方法列表里，类方法存在于元类的方法列表里。
- 4、类方法可以和对象方法重名。

### 如果在类方法中调用`self`会发生什么？

- 访问到的是类对象而不是实例对象。可以通过`self`调用其他的类方法，但是无法调用对象方法。

### 讲一下对象，类对象，元类结构体的组成以及他们是如何相关联的？

1. 实例对象的结构体是`objc_object`，主要存储的是`isa`以及相关的一些函数，例如`getIsa`、`initIsa`……还有一些关于对象内存管理的相关方法，`isTaggedPointer`、`isWeaklyReferenced`……
2. 类对象和元类对象其实都是`Class`，其结构体都是`objc_class`，里面存储了`isa`、`superClass`、成员变量集合、方法集合、协议集合、`cache`（方法缓存）等等……
3. 成员变量的`isa`指向其类对象、类对象的`isa`指向元类、元类的`isa`指向根元类，也就是`NSObject`的元类。

### 为什么对象方法中没有保存在对象结构体里面，而是保存在类对象的结构体里面？

- 每个对象都存储同一份实例方法列表太浪费。调用的时候对象只需要通过`isa`找到类对象，在其方法列表里查找就可以了。

### 类方法存在哪里？ 为什么要有元类的存在？

1. 类方法存储在元类的方法列表里，元类保存了类对象以及类方法的信息。(单一职责)
2. 为了调用类方法，这个类对象的`isa`指针必须指向一个包含类方法的`objc_class`结构体。

### `objc_getClass`、`object_getClass`、`class`方法有什么区别?

- ```
  objc_getClass
  ```

  - 传入类名字符串，返回类对象（`Class`）

- ```
  class
  ```

  方法

  - 实例对象调用`class`实际上调用 `object_getClass(self)`，返回的是类对象
  - 类对象调用`class`返回的是本身

- ```
  object_getClass
  ```

  - 传入实例对象（`instance`），返回类对象（`Class`）
  - 传入类对象（`Class`），返回元类（`meta-class`）对象
  - 传入元类（`meta-class`），返回基类的元类（`NSObject`的`meta-class`）

### 类与类之间的消息传递，有哪几种方式呢？

- 委托代理`delegate`
- 消息通知`Notification`
- `KVO`键值监听
- `block`

### 什么情况使用 weak 关键字，和 assign 有什么不同？

- ```
  weak
  ```

  - `weak` 关键字解决了一些循环引用的问题， 比如`delegate`，`block`，`xib`连线出来的控件一般也是`weak`（也可以用`strong`）
  - `weak`表明了一种“非拥有的关系”，不保留新值，也不释放旧值。`weak`只能修饰OC对象。

- ```
  assign
  ```

  - `assign`一般用于修饰非OC对象，常用于基本数据类型，MRC时可以修饰OC对象。
  - `assign`修饰对象时，当对象释放后（因为不存在强引用，离开作用域对象内存可能被回收），指针的地址还是存在的。指针并没有被置为nil，下次再访问该对象就会造成野指针异常。
  - `assign`修饰基本数据类型时，因为基本数据类型是分配在栈上的，由系统分配和释放，所以不会造成野指针。

### copy assign retain weak关键字

- `copy`：建立一个索引计数为1的对象，然后释放旧对象
- `assign`：简单赋值，不更改引用计数
- `retain`：释放旧对象，将新对象赋予成员变量，再提高新对象的引用计数
- `weak`：非持有关系

### assign可以用于对象吗

- 这么写大概率出现野指针崩溃。因为没有强引用对象，对象会被释放但是指针还在。

### 什么时候用copy 关键字？

- ```
  NSString、NSArray、NSDictionary
  ```

  等类型有可变的子类型，为了确保其不被更改，常用 copy修饰

  - copy修饰的本质是为了设置`setter`方法，例如，`setName:`传进一个`nameStr`参数，那么有了`copy`修饰词后，传给对应的成员变量`_name`的其实是[nameStr copy];。

- `block`用`copy`修饰，是MRC沿用下来的习惯。在 MRC 中方法内部的`block`是在栈区的， 使用`copy`可以把它复制到堆区。在 ARC 中可以用`strong`或者`copy`修饰，因为`block`的`retain`操作也是靠`copy`实现。

### 不用copy会有什么问题？

- `strong`修饰的`NSString`类型的`name`属性，传一个`NSMutableString`。在`strong`修饰下，把可变字符串`mutableString`赋值给`myString`后，改变`mutableString`的值导致了`myString`值的改变。而`copy`修饰下，却不会有这种变化。
- 当修饰可变类型的属性时，如`NSMutableArray、NSMutableDictionary、NSMutableString`，用`strong`。当修饰不可变类型的属性时，如`NSArray、NSDictionary、NSString`，用`copy`。

### 自定义类的对象怎样才能用copy修饰符？

- 自定义类的对象具有拷贝功能，需要实现`NSCopying`协议。
- 如果自定义的对象分为可变版本与不可变版本，要同时实现 `NSCopying` 与 `NSMutableCopying` 协议。

### MRC如何重写带`copy`关键字的`setter`？

```
- (void)setName:(NSString *)name {
    [_name release];
    _name = [name copy];
}
复制代码
```

### 怎样实现外部只读的属性，让它不被外部篡改？

- 头文件用`readonly`修饰并声明该属性。
- 正常情况下，属性默认是`readwrite`，可读写，如果我们设置了只读属性，就表明不能使用`setter`方法。
- 在`.m`文件中不能使用`self.ivar = @"aa";` 只能使用实例变量`_ivar = @"aa";`
- 外界想要修改只读属性的值，需要用到kvc赋值`[object setValue:@"mm" forKey:@"ivar"];`。

### 编程题：实现以下功能

- 1、编写一个自定义类：`Person`，父类为`NSObject`。
- 2、该类有两个属性，外部只读的属性`name`，还有一个属性`age`。
- 3、为该类编写一个初始化方法`initWithName:(NSString *)nameStr`，并依据该方法参数初始化`name`属性。
- 4、如果两个`Person`类的`name`相等，则认为两个`Person`相等

```
@interface Person : NSObject
@property (nonatomic, copy, readonly) NSString *name;
@property (nonatomic, assign) NSInteger age;
- (instancetype)initWithName:(NSString *)nameStr;
@end
 
@implementation Person
- (instancetype)initWithName:(NSString *)nameStr {
    if (self = [super init]) {
        _name = nameStr;
    }
    return self;
}

- (BOOL)isEqual:(id)object {
    if ([object isMemberOfClass:[Person class]] == NO) {
        return NO;
    }
    Person *p = (Person *)object;
    NSString *name1 = self.name;
    NSString *name2 = p.name;
    if (!name1 && !name2) {
        return YES;
    }
    return [name1 isEqualToString:name2];
}

@end
复制代码
```

- 初始化的方法中为什么将参数赋给

  ```
  _name
  ```

  ，为什么这样写就能访问到属性声明的示例变量？

  - 编辑器在编译期会自动补全出`@synthesize name = _name`的代码

- 初始化方法中的

  ```
  _name
  ```

  是在什么时候生成的？分配内存的时候还是初始化的时候？

  - 编译的时候自动的为`name`属性生成一个实例变量`_name`

- ```
  _name
  ```

  存放在什么地方

  - 成员变量存储在堆中(当前对象对应的堆得存储空间中) ，不会被系统自动释放，只能有程序员手动释放。

- 初始化

  ```
  return
  ```

  的

  ```
  self
  ```

  是在上面时候生成的？

  - 在`alloc`时候分配内存，在`init`初始化的。

  ## 分类

### `category`的作用

- 从架构上说，可以分门别类存放代码，增加代码的可读性，外部可以按需加载功能
- 为已有的类做扩展，添加属性、方法、协议
- 复写方法
- 公开私有方法
- 模拟多继承

### `category` 的实现原理，如何被加载的

- `category` 编译完成的时候和类是分开的，在程序运行时才通过`runtime`合并在一起。
- `_objc_init`是`Objcet-C runtime`的入口函数，主要读取`Mach-O`文件完成`OC`的内存布局，以及初始化`runtime`相关数据结构。这个函数里会调用到两外两个函数，`map_images`和`load_Images`
- `map_images`追溯进去发现其内部调用了`_read_images`函数，`_read_images`会读取各种类及相关分类的信息。
- 读取到相关的信息后通过`addUnattchedCategoryForClass`函数建立类和分类的关联。
- 建立关联后通过`remethodizeClass -> attachCategories`重新规划方法列表、协议列表、属性列表，把分类的内容合并到主类
- 在`map_images`处理完成后，开始`load_images`的流程。首先会调用`prepare_load_methods`做加载准备，这里面会通过`schedule_class_load`递归查找到`NSObject`然后从上往下调用类的`load`方法。
- 处理完类的`load`方法后取出非懒加载的分类通过`add_category_to_loadable_list`添加到一个全局列表里
- 最后调用`call_load_methods`调用分类的`load`函数

### `load`方法加载顺序

- 类的`load`方法在其父类`load`方法后执行
- 分类的`load`方法在主类`load`方法后执行
- 两个分类的`load`方法执行顺序遵循先编译先执行

### `load`、`initialize`方法的区别什么？在继承关系中他们有什么区别

- `load`方法在运行时调用，加载类或分类的时候调用一次，继承关系参考`load`方法加载顺序。
- `initialize`在第一次自身或子类接受`objc_msgSend`消息的时候调用。如果子类没有实现`initialize`，会调用父类的。所以父类的`initialize`可能被调用多次
- `load`方法是直接通过方法调用地址调用的，`initialize`则是通过`isa`走位查找调用的
- `load`方法不会被覆盖，`initialize`可以覆盖

### 代码写在load和initialize中会影响启动吗

- `load`方法中添加操作会影响启动速度，因为`load`方法在启动时调用
- `initialize`方法在第一次调用方法时触发，对启动影响相对较小

### +load与+initialize

##### 1、+load

`（1）+load`方法是一定会在runtime中被调用的。只要类被添加到runtime中了，就会调用`+load`方法，即只要是在`Compile Sources`中出现的文件总是会被装载，与这个类是否被用到无关，因此`+load`方法总是**在main函数之前调用**。

`（2）+load`方法**不会覆盖**。也就是说，如果子类实现了`+load`方法，那么会先调用父类的`+load`方法（无需手动调用super），然后又去执行子类的`+load`方法。

（3）+load方法只会调用一次。

（4）+load方法执行顺序是：类 -> 子类 ->分类。而不同分类之间的执行顺序不一定，依据在`Compile Sources`中出现的顺序**（先编译，则先调用，列表中在下方的为“先”）**。

（5）+load方法是函数指针调用，即遍历类中的方法列表，直接根据函数地址调用。如果子类没有实现+load方法，子类也不会自动调用父类的+load方法。

##### 2、+initialize

`（1）+initialize`方法是在类或它的子类收到第一条消息之前被调用的，这里所指的消息包括实例方法和类方法的调用。因此`+initialize`方法总是**在main函数之后调用**。

（2）`+initialize`方法只会调用一次。

`（3）+initialize`方法实际上是一种惰性调用，如果一个类一直没被用到，那它的`+initialize`方法也不会被调用，这一点有利于节约资源。

`（4）+initialize`方法**会覆盖**。如果子类实现了`+initialize`方法，就不会执行父类的了，直接执行子类本身的。如果分类实现了`+initialize`方法，也不会再执行主类的。

（5）`+initialize`方法的执行覆盖顺序是：分类 -> 子类 ->类。**且只会有一个`+initialize`方法被执行**。

（6）`+initialize`方法是发送消息（objc_msgSend()），如果子类没有实现`+initialize`方法，也会自动调用其父类的`+initialize`方法。

##### 3、两者的异同

（1）相同点

1. load和initialize会被自动调用，不能手动调用它们。
2. 子类实现了load和initialize的话，会隐式调用父类的load和initialize方法。
3. load和initialize方法内部使用了锁，因此它们是线程安全的。

（2）不同点

1. 调用顺序不同，以main函数为分界，`+load`方法在main函数之前执行，`+initialize`在main函数之后执行。
2. 子类中没有实现`+load`方法的话，子类不会调用父类的`+load`方法；而子类如果没有实现`+initialize`方法的话，也会自动调用父类的`+initialize`方法。
3. `+load`方法是在类被装在进来的时候就会调用，`+initialize`在第一次给某个类发送消息时调用（比如实例化一个对象），并且只会调用一次，是懒加载模式，如果这个类一直没有使用，就不回调用到`+initialize`方法。

##### 4、使用场景

（1）`+load`一般是用来交换方法`Method Swizzle`，由于它是线程安全的，而且一定会调用且只会调用一次，通常在使用UrlRouter的时候注册类的时候也在`+load`方法中注册。
`（2）+initialize`方法主要用来对一些不方便在编译期初始化的对象进行赋值，或者说对一些静态常量进行初始化操作。

### 多个category的都有同名方法，会使用哪一个

- 重新规划方法列表的时候，后处理的分类的方法会被放在方法列表前面。
- 调用的时候只要找到考前的那个方法就会立刻执行，所以就会执行后加载的那个分类的方法。
- `Build Phases -> Compile Source`靠后的分类。

### `category` & `extension`区别，能给`NSObject`添加`extension`吗？

- `extension`扩展是特殊的`category`，称为匿名分类或者私有分类，可以为类添加成员变量和方法。
- `extension`在编译期决议，`category`则是在运行时加载。
- `extension`一般用来隐藏私有信息，`category`可以公开私有信息
- 无法给系统类添加`extension`，但是可以给系统类添加`category`
- `extension`可以添加成员变量，而`category`不可以
- `extension`和`category`都可以添加属性，但是`category`的属性不能自动生成`成员变量、getter、setter`

### 分类中添加实例变量和属性分别会发生什么，还是什么时候会发生问题？为什么

- 添加实例变量编译时报错。
- 添加属性没问题，但是在运行的时候使用这个属性程序crash。原因是没有实例变量也没有`set/get`方法。
- 可以通过关联对象去实现

### 分类中为什么不能添加成员变量（`runtime`除外）？

- 类对象在创建的时候已经定好了成员变量，但是分类是运行时加载的，无法添加。
- 类对象里的 `class_ro_t` 类型的数据在运行期间不能改变，再添加方法和协议都是修改的 `class_rw_t` 的数据。
- 分类添加方法、协议是把`category`中的方法，协议放在`category_t`结构体中，再拷贝到类对象里面。但是`category_t`里面没有成员变量列表。
- 虽然`category`可以写上属性，其实是通过关联对象实现的，需要手动添加`setter & getter`。

### 分类可以添加那些内容

- 实例方法
- 类方法
- 协议
- 属性

### 关联对象的实现和原理

- 关联对象不存储在关联对象本身内存中，而是存储在一个全局容器中；

- 这个容器是由

   

  ```
  AssociationsManager
  ```

   

  管理并在它维护的一个单例

   

  ```
  Hash
  ```

   

  表

  ```
  AssociationsHashMap
  ```

   

  ；

  - 第一层 AssociationsHashMap：类名object ：bucket（map）
  - 第二层 ObjectAssociationMap：key（name）：ObjcAssociation（value和policy）

- `AssociationsManager` 使用 `AssociationsManagerLock` 自旋锁保证了线程安全。

- 通过`objc_setAssociatedObject`给某对象添加关联值

- 通过`objc_getAssociatedObject`获取某对象的关联值

- 通过`objc_removeAssociatedObjects`移除某对象的关联值

### 使用关联对象，需要在主对象 dealloc 的时候手动释放么？

- 不需要，主对象通过 `dealloc -> object_dispose -> object_remove_assocations` 进行关联对象的释放。

### 能否向编译后得到的类中增加实例变量， 能否向运行时创建的类中添加实例变量？

- 不可以，编译完成的类已经生成了不可变的成员变量集合。
- 可以，通过`class_addIvar`函数给运行时创建的类添加实例变量。

### 主类存在了`foo`方法，分类也存在`foo`方法，调用时会出现什么情况？ 如果想执行主类的`foo`方法，如何去做？

- 主类的方法不会被调用，分类的方法会被调用。分类和主类的`foo`方法都存在方法列表里，只是分类方法在前，主类方法在后， 调用的时候会首先找到第一次出现的方法。

- 如果想要要执行主类的方法，需要逆序遍历方法列表，第一次遍历到的foo方法就是主类的方法

  ```
  - (void)foo{   
    [类 invokeOriginalMethod:self selector:_cmd];
  }
  
  + (void)invokeOriginalMethod:(id)target selector:(SEL)selector {
      uint count;
      Method *list = class_copyMethodList([target class], &count);
      for ( int i = count - 1 ; i >= 0; i--) {
          Method method = list[i];
          SEL name = method_getName(method);
          IMP imp = method_getImplementation(method);
          if (name == selector) {
              ((void (*)(id, SEL))imp)(target, name);
              break;
          }
      }
      free(list);
  }
  复制代码
  ```

## runtime

### `OC`是动态运行时语言是什么意思？

- 动态类型：运行时确定对象的类型，编译时期能通过，但不代表运行过程中没有问题
- 动态绑定：运行时才确定对象调用的方法（消息转发）
- 动态加载：动态库的方法实现不拷贝到程序中，只记录引用，直到使用相关方法的时候才到库里面查找方法实现

### `runtime`能做什么？

- 获取类的成员变量、方法、协议
- 为类添加成员变量、方法、协议
- 动态改变方法实现

### `class_copyIvarList`与`class_copyPropertyList`的区别?

- 1、`class_copyIvarList`可以获取`.h`和`.m`中的所有属性以及`@interface`大括号中声明的变量，获取的属性名称有下划线(大括号中的除外)。
- 2、`class_copyPropertyList`只能获取由`@property`声明的属性（包括`.m`），获取的属性名称不带下划线。

### `class_ro_t`和`class_rw_t`的区别？

- `class_rw_t`提供了运行时对类拓展的能力，`class_rw_t`结构体中存储了`class_ro_t`。
- `class_ro_t`存储的是类在编译时已经确定的信息，是不可改变的。
- 二者都存有类的方法、属性（成员变量）、协议等信息，不过存储它们的列表实现方式不同。简单的说`class_rw_t`存储列表使用的二维数组，`class_ro_t`使用的一维数组。
- 运行时修改类的方法，属性，协议等都存储于`class_rw_t`中

### 什么是 Method Swizzle（黑魔法），什么情况下会使用？

- `Method Swizzle` 是改变一个已存在的选择器（`SEL`）对应的实现（`IMP`）的过程。
- 类的方法列表存放着`SEL`的名字和`IMP`的映射关系。
- 开发者可以利用 `method_exchangeImplementations` 来交换2个方法中的`IMP`
- 开发者可以利用 `method_setImplementation` 来直接设置某个方法的IMP
- 这就可以在运行时改变`SEL`和`IMP`的映射关系，从而实现方法替换。

### `Method Swizzle`注意事项

- 为了确保`Swizzle Method`方法替换一定被执行调用，可以在`load`中执行
- `+load`里面使用的时候不要调用`[super load]`。如果多次调用了`[super load]`，可能会出现“Swizzle无效”的假象
- 避免调用`[super load]`导致`Swizzling`多次执行，在`load`中使用`dispatch_once`确保交换只被执行一次。
- 子类替换没有实现的继承方法，会替换掉父类中的实现，影响父类及其他子嘞
- `+initialize` 里面使用要加`dispatch_once`
- 进行版本迭代的时候需要进行一些检验，防止系统库的函数发生了变化

### 如何`hook`一个对象的方法，而不影响其它对象

- 方法1：新建一个子类重写方法
- 方法2：让这个对象的类遵循某个协议，`hook`时判断。弊端是其他对象遵循了这个协议会受到影响。
- 方法3：运行时创建一个新的子类，修改对象 `isa` 指针指向子类，`hook` 时使用`isKindOf` 判断类型

## 消息发送

### 消息机制

- 1、快速查找，方法缓存
- 2、慢速查找，方法列表
- 3、消息转发
  - 3-1、方法的动态解析，`resolveInstanceMethod`
  - 3-2、快速消息转发，`forwardingTargetForSelector`
  - 3-3、标准消息转发，`methodSignatureForSelector & forwardInvocation`

### objc中向一个nil对象发送消息将会发生什么？

- 在寻找对象的`isa`指针时，返回地址`0x0`，不回做任何操作，也不会有任何错误。

### objc在向一个对象发送消息时，发生了什么？

- 方法调用实际上是发送消息，通过调用 `objc_msgSend()`实现的。
- 首先，通过`obj`的`isa`指针找到对应的`class`。
- 然后，开启快速查找流程。在`class`的缓存方法列表（`objc_cache`）里查找方法，如果找到就直接返回对应`IMP`。
- 如果在缓存中找不到，开始慢速查找流程。在`class`的`Method List`查找对应方法，找到了返回对应`IMP`。
- 都找不到就会走消息转发流程

#### `_objc_msgForward` 函数是做什么的？

- `_objc_msgForward`用于消息转发：向一个对象发送一条消息，但它并没有实现的时候，就调用`_objc_msgForward`尝试做消息转发。

### 为什么需要做方法缓存？

- 每次执行这个方法的时候都查一遍`Method List`太消耗性能。
- 使用`objc_cache`把调用过的方法做一个缓存， 把`method_name`作为`key`， `method_IMP`作为`value`。
- 下次接收到消息的时候，直接通过`objc_cache`去找到对应的`IMP`即可， 避免每一次都去遍历`objc_method_list`

### 一直都找不到方法怎么办？

- 会触发消息转发机制，我们一共有三次机会补救以防止`crash`
- 方法的动态解析，通过`resolveInstanceMethod`添加一个IMP使其执行。
- 快速消息转发，在 `forwardingTargetForSelector`返回一个可以执行该方法的对象。
- 标准消息转发，`methodSignatureForSelector`创建相同方法类型的方法签名（`NSMethodSignature`），然后重写`forwardInvocation`并把拥有该签名的方法赋值到`anInvocation.selector`。

### 消息转发机制的优劣

- 优点：消息转发机制提供了找不到方法时的补救机会。
- 缺点：一般情况下会在基类做crash处理，那么有可能把一部分的crash忽略过去导致无法暴露问题。

### `IMP`、`SEL`、`Method`的区别和使用场景

- `SEL`相当于一个代号，方便查找方法的代号，处理通知/定时器等都会用到
- `IMP`是指向方法实现的指针，动态方法解析的时候会用到
- `Method`是一个对象，里面就存有`SEL`和`IMP`，消息转发流程获取方法签名的时候会用到



## KVO & KVC

### `KVO`用法和底层原理

- 使用方法：添加观察者，然后怎样实现监听的代理
- `KVO`底层使用了 `isa-swizling`的技术.
- `OC`中每个对象/类都有`isa`指针, `isa` 表示这个对象是哪个类的对象.
- 当给对象的某个属性注册了一个 observer，系统会创建一个新的中间类（`intermediate class`）继承原来的`class`，把该对象的`isa`指针指向中间类。
- 然后中间类会重写`setter`方法，调用`setter`之前调用`willChangeValueForKey`, 调用`setter`之后调用`didChangeValueForKey`，以此通知所有观察者值发生更改。
- 重写了 `-class` 方法，企图欺骗我们这个类没有变，就是原本那个类。

### KVO的优缺点

- 优点
  - 1、可以方便快捷的实现两个对象的关联同步，例如`view & model`
  - 2、能够观察到新值和旧值的变化
  - 3、可以方便的观察到嵌套类型的数据变化
- 缺点
  - 1、观察对象通过`string`类型设置，如果写错或者变量名改变，编译时可以通过但是运行时会发生`crash`
  - 2、观察多个值需要在代理方法中多个`if`判断
  - 3、忘记移除观察者或重复移除观察者会导致`crash`

### 怎么手动触发`KVO`

- `KVO`机制是通过`willChangeValueForKey:`和`didChangeValueForKey:`两个方法触发的。
- 在观察对象变化前调用`willChangeValueForKey:`
- 在观察对象变化后调用`didChangeValueForKey:`
- 所以只需要在变更观察值前后手动调用即可。

### 给KVO添加筛选条件

- 重写`automaticallyNotifiesObserversForKey`，需要筛选的`key`返回`NO`。
- `setter`里添加判断后手动触发`KVO`

```
+ (BOOL)automaticallyNotifiesObserversForKey:(NSString *)key {
    if ([key isEqualToString:@"age"]) {
        return NO;
    }
    return [super automaticallyNotifiesObserversForKey:key];
}

- (void)setAge:(NSInteger)age {
    if (age >= 18) {
        [self willChangeValueForKey:@"age"];
        _age = age;
        [self didChangeValueForKey:@"age"];
    }else {
        _age = age;
    }
}
复制代码
```

### 使用KVC修改会触发KVO吗？

- 会，只要`accessInstanceVariablesDirectly`返回`YES`，通过KVC修改成员变量的值会触发KVO。
- 这说明KVC内部调用了`willChangeValueForKey:`方法和`didChangeValueForKey:`方法

### 直接修改成员变量会触发KVO吗？

- 不会

### KVO的崩溃与防护

崩溃原因：

- KVO 添加次数和移除次数不匹配，大部分是移除多于注册。
- 被观察者`dealloc`时仍然注册着 KVO，导致崩溃。
- 添加了观察者，但未实现 `observeValueForKeyPath:ofObject:change:context:` 。 防护方案1：
- 直接使用facebook开源框架`KVOController` 防护方案2：
- 自定义一个哈希表，记录观察者和观察对象的关系。
- 使用`fishhook`替换 `addObserver:forKeyPath:options:context:`，在添加前先判断是否已经存在相同观察者，不存在才添加，避免重复触发造成bug。
- 使用`fishhook`替换`removeObserver:forKeyPath:`和`removeObserver:forKeyPath:context`，移除之前判断是否存在对应关系，如果存在才释放。
- 使用`fishhook`替换`dealloc`，执行`dealloc`前判断是否存在未移除的观察者，存在的话先移除。

### KVC底层原理

#### `setValue:forKey:`的实现

- 查找`setKey:`方法和`_setKey:`方法，只要找到就直接传递参数，调用方法；
- 如果没有找到`setKey:`和`_setKey:`方法，查看`accessInstanceVariablesDirectly`方法的返回值，如果返回`NO`（不允许直接访问成员变量），调用`setValue:forUndefineKey:`并抛出异常`NSUnknownKeyException`；
- 如果`accessInstanceVariablesDirectly`方法返回`YES`（可以访问其成员变量），就按照顺序依次查找 `_key、_isKey、key、isKey` 这四个成员变量，如果查找到了就直接赋值；如果没有查到，调用`setValue:forUndefineKey:`并抛出异常`NSUnknownKeyException`。

#### `valueForKey:`的实现

- 按照`getKey，key，isKey`的顺序查找方法，只要找到就直接调用；
- 如果没有找到，`accessInstanceVariablesDirectly`返回`YES`（可以访问其成员变量），按照顺序依次查找`_key、_isKey、key、isKey` 这四个成员变量，找到就取值；如果没有找到成员变量，调用`valueforUndefineKey`并抛出异常`NSUnknownKeyException`。
- `accessInstanceVariablesDirectly`返回`NO`（不允许直接访问成员变量），那么会调用`valueforUndefineKey:`方法，并抛出异常`NSUnknownKeyException`；

## 多线程

### 进程和线程的区别

- 进程：进程是指在系统中正在运行的一个应用程序，一个进程拥有多个线程。
- 线程：线程是进程中的一个单位，一个进程想要执行任务， 必须至少有一条线程。应程序启动默认开启主线程。

### 进程都有什么状态

- `Not Running`：未运行。
- `Inactive`：前台非活动状态。处于前台，但是不能接受事件处理。
- `Active`：前台活动状态。处于前台，能接受事件处理。
- `Background`：后台状态。进入后台，如果又可执行代码，会执行代码，代码执行完毕，程序进行挂起。
- `Suspended`：挂起状态。进入后台，不能执行代码，如果内存不足，程序会被杀死。

### 什么是线程安全？

- 多条线程同时访问一段代码，不会造成数据混乱的情况

### 怎样保证线程安全？

- 通过线程加锁
- `pthread_mutex` 互斥锁（C语言）
- `@synchronized`
- `NSLock` 对象锁
- `NSRecursiveLock` 递归锁
- `NSCondition & NSConditionLock` 条件锁
- `dispatch_semaphore` GCD信号量实现加锁
- `OSSpinLock`自旋锁（不建议使用）
- `os_unfair_lock`自旋锁（IOS10以后替代`OSSpinLock`）

### 你接触到的项目，哪些场景运用到了线程安全？

- 在线列表的增员和减员，需要加锁保持其线程安全。

### `iOS`开发中有多少类型的线程？分别说说

- 1、

  ```
  pthread
  ```

  - C语言实现的跨平台通用的多线程API
  - 使用难度大，没有用过

- 2、

  ```
  NSThread
  ```

  - `OC`面向对象的多线程`API`
  - 简单易用，可以直接操作线程对象。
  - 需要手动管理生命周期

- 3、

  ```
  GCD
  ```

  - C语言实现的多核并行CPU方案，能更合理的运行多核`CPU`
  - 可以自动管理生命周期

- 4、

  ```
  NSOperation
  ```

  - `OC`基于`GCD`的封装
  - 完全面向对象的多线程方案
  - 可以自动管理生命周期

### GCD有什么队列，默认提供了哪些队列

- 串行同步队列，任务按顺序（串行），在当前线程执行（同步）
- 串行异步队列，任务按顺序（串行），开辟新的线程执行（异步）
- 并行同步队列，任务按顺序（无法体现并行），在当前线程执行（同步）
- 并行异步队列，任务同时执行（并行），开辟新的线程执行（异步）
- 默认提供了主队列和全局队列

### `GCD`主线程 & 主队列的关系

- 主队列任务只在主线程中被执行的
- 主线程运行的是一个 `runloop`，除了主队列的任务，还有 UI 处理和绘制任务。

### 描述一下线程同步与异步的区别?

- 线程同步是指当前有多个线程的话，必须等一个线程执行完了才能执行下一个线程。
- 线程异步指一个线程去执行，他的下一个线程不用等待他执行完就开始执行。

### 线程同步的方式

- `GCD`的串行队列，任务都一个个按顺序执行
- `NSOperationQueue`设置`maxConcurrentOperationCount = 1`，同一时刻只有1个`NSOperation`被执行
- 使用`dispatch_semaphore`信号量阻塞线程，直到任务完成再放行
- `dispatch_group`也可以阻塞到所有任务完成才放行

### 什么情况下会线程死锁

- 串行队列，正在进行的任务A向串行队列添加一个同步任务B，会造成AB两个任务互相等待，形成死锁。
- 优先级反转，`OSSpinlock`

### `dispatch_once`实现原理

- `dispatch_once`需要传入`dispatch_once_t`类型的参数，其实是个长整形
- 处理`block`前会判断传入的`dispatch_once_t`是否为0，为0表示`block` 尚未执行。
- 执行后把`token`的值改为1，下次再进来的时候判断非0直接不处理了。

### `performSelector`和`runloop`的关系

- 调用 `performSelecter:afterDelay:` ，其内部会创建一个`Timer`并添加到当前线程的`RunLoop` 。
- 如果当前线程`Runloop`没有跑起来，这个方法会失效。
- 其他的`performSelector`系列方法是类似的

### 子线程执行 `[p performSelector:@selector(func) withObject:nil afterDelay:4]` 会发生什么？

- 上面这个方法放在子线程，其实内部会创建一个`NSTimer`定时器。

- 子线程不会默认开启

  ```
  runloop
  ```

  ，如果需要执行

  ```
  func
  ```

  函数得手动开启

  ```
  runloop
  ```

  ```
  dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
      dispatch_async(queue, ^{
          // [[NSRunLoop currentRunLoop] run]; 放在上面无效
          // 只开启runloop但是里面没有任何事件，开启失败
          [self performSelector:@selector(test) withObject:nil afterDelay:2];
          [[NSRunLoop currentRunLoop] run];
  });
  复制代码
  ```

### 为什么只在主线程刷新UI

- `UIKit`是线程不安全的，UI操作涉及到渲染和访问`View`的属性，异步操作会存在读写问题，为其加锁则会耗费大量资源并拖慢运行速度。
- 程序的起点`UIApplication`在主线程初始化，所有的用户事件（触摸交互）都在主线程传递，所以`view`只能在主线程上才能对事件进行响应。

### 一个队列负责插入数据操作，一个队列负责读取操作，同时操作一个存储的队列，如何保证顺利进行

- 使用`GCD`栅栏函数实现多度单写
- 读取的时候使用 `dispatch_sync` 立刻返回数据
- 写入的时候使用 `dispatch_barrier_async` 阻塞其他操作后写入
- 注意尽量不要使用全局队列，因为全局队列里还有其他操作

## 锁

### 为什么需要锁？

- 多线程编程中会出现线程相互干扰的情况，如多个线程访问一个资源。
- 需要一些同步工具，确保当线程交互的时候是安全的。

### 什么是互斥锁

- 如果共享数据已经有了其他线程加锁了，线程会进行休眠状态等待锁
- 一旦被访问的资源被解锁，则等待资源的线程会被唤醒。
- 任务复杂的时间长的情况建议使用互斥锁
- 优点
  - 获取不到资源时线程休眠，cpu可以调度其他的线程工作
- 缺点
  - 存在线程调度的开销
  - 如果任务时间很短，线程调度降低了cpu的效率

### 什么是自旋锁

- 如果共享数据已经有其他线程加锁了，线程会以死循环的方式等待锁
- 一旦被访问的资源被解锁，则等待资源的线程会立即执行
- 适用于持有锁较短时间
- 优点：
  - 自旋锁不会引起线程休眠，不会进行线程调度和CPU时间片轮转等耗时操作。
  - 如果能在很短的时间内获得锁，自旋锁效率远高于互斥锁。
- 缺点：
  - 自旋锁一直占用CPU，未获得锁的情况下处于忙等状态。
  - 如果不能在很短的时间内获得锁，使CPU效率降低。
  - 自旋锁不能实现递归调用。

### 读写锁

- 读写锁又被称为 `rw锁`或者 `readwrite锁`
- 不是最常用的，一般是数据库操作才会用到。
- 具体操作为多读单写，写入操作只能串行执行，且写入时不能读取；读取需支持多线程操作，且读取时不能写入

### 说说你知道的锁有哪些

- `pthread_mutex` 互斥锁（C语言）
- `@synchronized`
- `NSLock` 对象锁
- `NSRecursiveLock` 递归锁
- `NSCondition & NSConditionLock` 条件锁
- `dispatch_semaphore` GCD信号量实现加锁
- `OSSpinLock`自旋锁（暂不建议使用）
- `os_unfair_lock`自旋锁（IOS10以后替代`OSSpinLock`）

### 说说`@synchronized`

- 原理
  - 内部应该是一个可重入互斥锁（`recursive_mutex_t`）
  - 底层是链表，存储SyncData，SyncData里面包含一个 `threadCount`，就是访问资源的线程数量。
  - `objc_sync_enter(obj)，objc_sync_exit(obj)`，通过`obj`的地址作为`hash`传参查找`SyncData`，上锁解锁。
  - 传入的`obj`被释放或为`nil`，会执行锁的释放
- 优点
  - 不需要创建锁对象也能实现锁的功能
  - 使用简单方便，代码可读性强
- 缺点
  - 加锁的代码尽量少
  - 性能没有那么好
  - 注意锁的对象必须是同一个`OC`对象

### 说说`NSLock`

- 遵循`NSLocking`协议
- 注意点
  - 同一线程`lock`和`unlock`需要成对出现
  - 同一线程连续`lock`两次会造成死锁

### 说说`NSRecursiveLock`

- `NSRecursiveLock`是递归锁
- 注意点
  - 同一个程`lock`多次而不造成死锁
  - 同一线程当`lock & unlock`数量一致的时候才会释放锁，其他线程才能上锁

### 说说`NSCondition & NSConditionLock`

- 条件锁：满足条件执行锁住的代码；不满足条件就阻塞线程，直到另一个线程发出解锁信号。

- ```
  NSCondition
  ```

  对象实际上作为一个锁和一个线程检查器

  - 锁保护数据源，执行条件引发的任务。
  - 线程检查器根据条件判断是否阻塞线程。
  - 需要手动等待和手动信号解除等待
  - 一个`wait`必须对应一个`signal`，一次唤醒全部需要使用`broadcast`

- ```
  NSConditionLock
  ```

  是

  ```
  NSCondition
  ```

  的封装

  - 通过不同的`condition`值触发不同的操作
  - 解锁时通过`unlockWithCondition` 修改`condition`实现任务依赖
  - 通过`condition`自动判断阻塞还是唤醒线程

### 说说GCD信号量实现锁

- `dispatch_semaphore_creat(0)`生成一个信号量`semaphore = 0`（ 传入的值可以控制并行任务的数量）
- `dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)` 使`semaphore - 1`，当值小于0进入等待
- `dispatch_semaphore_signal(semaphore)`发出信号，使`semaphore + 1`，当值大于等于0放行

### 说说`OSSpinLock`

- `OSSpinLock`是自旋锁，忙等锁
- 自旋锁存在优先级反转的问题，线程有优先级的时候可能导致下列情况。
  - 一个优先级低的线程先访问某个数据，此时使用自旋锁进行了加锁。
  - 一个优先级高的线程又去访问这个数据，优先级高的线程会一直占着CPU资源忙等访问
  - 结果导致优先级低的线程没有CPU资源完成任务，也无法释放锁。
- 由于自旋锁本身存在的问题，所以苹果已经废弃了`OSSpinLock`。

### 说说 `os_unfair_lock`

- iOS10以后替代`OSSpinLock`的锁，不再忙等
- 获取不到资源时休眠，获取到资源时由内核唤醒线程
- 没有加强公平性和顺序，释放锁的线程可能立即再次加锁，之前等待锁的线程唤醒后可能也没能加锁成功。
- 虽然解决了优先级反转，但也造成了饥饿（`starvation`）
- `starvation` 指贪婪线程占用资源事件太长，其他线程无法访问共享资源。

### 5个线程读一个文件，如何实现最多只有2个线程同时读这个文件

- `dispatch_semaphore`信号量控制

### Objective-C中的原子和非原子属性

- OC在定义属性时有`nonatomic`和`atomic`两种选择
- `atomic`：原子属性，为`setter/getter`方法都加锁（默认就是`atomic`），线程安全，需要消耗大量的资源
- `nonatomic`：非原子属性，不加锁，非线程安全

```
atomic加锁原理:
property (assign, atomic) int age;
 - (void)setAge:(int)age
{ 
    @synchronized(self) {  
       _age = age;
    }
}

- （int）age {
  int age1 = 0;
  @synchronized(self) {
    age1 = _age;
  }
}
复制代码
```

### `atomic` 修饰的属性 `int a`，在不同的线程执行 `self.a = self.a + 1` 执行一万次，这个属性的值会是一万吗？

- 不会，左边的点语法调用的是`setter`，右边调用的是`getter`，这行语句并不是原子性的。

### `atomic`就一定能保证线程安全么？

- 不能，只能保证`setter`和`getter`在当前线程的安全
- 一个线程在连续多次读取某条属性值的时候，别的线程同时在改值，最终无法得出期望值
- 一个线程在获取当前属性的值， 另外一个线程把这个属性释放调了，有可能造成崩溃

### `nonatomic`是非原子操作符，为什么用`nonatomic`不用`atomic`？

- 如果该对象无需考虑多线程的情况，请加入这个属性修饰，这样会让编译器少生成一些互斥加锁代码，可以提高效率。
- 使用`atomic`，编译器会在`setter`和`getter`方法里面自动生成互斥锁代码，避免该变量读写不同步。

### 有人说能`atomic`耗内存，你觉得呢？

- 因为会自动加锁，所以性能比`nonatomic`差。

### `atomic`为什么会失效

- `atomic`修饰的属性靠编译器自动生成的`get/set`方法实现原子操作，如果重写了任意一个，`atomic`关键字的特性将失效

### `nonatomic`实现

```
- (NSString *)userName {
    return _userName;
}

- (void)setUserName:(NSString *)userName {
    _userName = userName;
}
复制代码
```

### `atomic` 的实现

```
- (NSString *)userName {
    NSString *name;
    @synchronized (self) {
        name = _userName;
    }
    return name;
}

- (void)setUserName:(NSString *)userName {
    @synchronized (self) {
        _userName = userName;
    }
}
复制代码
```

## runloop

### `runloop`是什么？

- 系统内部存在管理事件的循环机制
- `runloop` 是利用这个循环，管理消息和事件的对象。

### `runloop` 是否等于 `while(1) { do something ... }` ？

- 不是
- `while(1)` 是一个忙等的状态，需要一直占用资源。
- `runloop` 没有消息需要处理时进入休眠状态，消息来了，需要处理时才被唤醒。

### `runloop`的基本模式

- iOS中有五种`runLoop`模式
- `UIInitializationRunLoopMode`（启动后进入的第一个`Mode`，启动完成后就不再使用，切换到 `kCFRunLoopDefaultMode`）
- `kCFRunLoopDefaultMode`（App的默认`Mode`，通常主线程是在这个 `Mode` 下运行）
- `UITrackingRunLoopMode`（界面跟踪 `Mode`，用于 `ScrollView` 追踪触摸滑动，保证界面滑动时不受其他 `Mode`影响）
- `NSRunLoopCommonModes` （这是一个伪`Mode`，等效于`NSDefaultRunLoopMode`和`NSEventTrackingRunLoopMode`的结合 ）
- `GSEventReceiveRunLoopMode`（接受系统事件的内部 `Mode`，通常用不到）

### `runLoop`的基本原理

- 系统中的主线程会默认开启`runloop`检测事件，没有事件需要处理的时候`runloop`会处于休眠状态。
- 一旦有事件触发，例如用户点击屏幕，就会唤醒`runloop`使进入监听状态，然后处理事件。
- 事件处理完成后又会重新进入休眠，等待下一次事件唤醒

### `runloop`和线程的关系

- `runloop`和线程一一对应。
- 主线程的创建的时候默认开启`runloop`，为了保证程序一直在跑。
- 支线程的`runloop`是懒加载的，需要手动开启。

### `runloop`事件处理流程

- 事件会触发`runloop`的入口函数`CFRunLoopRunSpecific`，函数内部首先会通知`observer`把状态切换成`kCFRunLoopEntry`，然后通过`__CFRunLoopRun`启动`runloop`处理事件
- `__CFRunLoopRun`的核心是是一个`do - while`循环，循环内容如下 ![image-20220123003341954.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/56439e9936bd41d8bfd29e6834867d78~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.image)

### `runloop`是怎么被唤醒的

- 没有消息需要处理时，休眠线程以避免资源占用。从用户态切换到内核态，等待消息；
- 有消息需要处理时，立刻唤醒线程，回到用户态处理消息；
- `source0`通过屏幕触发直接唤醒
- `source0`通过调用`mach_msg()`函数来转移当前线程的控制权给内核态/用户态。

### 什么是用户态、核心态

- 内核态：运行操作系统程序 ，表示一个应用进程执行系统调用后，或I/O 中断，时钟中断后，进程便处于内核执行
- 用户态：运行用户程序 ，表示进程正处于用户状态中执行

### runloop的状态

```
CFRunLoopObserverRef observerRef = CFRunLoopObserverCreateWithHandler(CFAllocatorGetDefault(), kCFRunLoopAllActivities, YES, 0, ^(CFRunLoopObserverRef observer, CFRunLoopActivity activity) {
        switch (activity) {
            case kCFRunLoopEntry: NSLog(@"runloop启动"); break;
            case kCFRunLoopBeforeTimers: NSLog(@"runloop即将处理timer事件"); break;
            case kCFRunLoopBeforeSources: NSLog(@"runloop即将处理sources事件"); break;
            case kCFRunLoopBeforeWaiting: NSLog(@"runloop即将进入休眠"); break;
            case kCFRunLoopAfterWaiting: NSLog(@"runloop被唤醒"); break;
            case kCFRunLoopExit: NSLog(@"runloop退出"); break;
            default: break;
        }
    });
    CFRunLoopAddObserver(CFRunLoopGetCurrent(), observerRef, kCFRunLoopDefaultMode);
}
复制代码
```

### `runLoop` 卡顿检测的方法

- ```
  NSRunLoop
  ```

   

  处理耗时主要下面两种情况

  - `kCFRunLoopBeforeSources` 和 `kCFRunLoopBeforeWaiting` 之间
  - `kCFRunLoopAfterWaiting` 之后

- 上述两个时间太长，可以判定此时主线程卡顿

- 可以添加

  ```
  Observer
  ```

  到主线程

  ```
  Runloop
  ```

  中，监听

  ```
  Runloop
  ```

  状态切换耗时，监听卡顿

  - 用一个`do-while`循环处理路基，信号量设置阈值判断是否卡顿
  - `dispatch_semaphore_wait` 返回值 **非0** 表示`timeout`卡顿发生
  - 获取卡顿的堆栈传至后端，再分析

### 怎么启动一个常驻线程

```
// 创建线程
NSThread *thread = [[NSThread alloc]  initWithTarget:self selector:@selector(play) object:nil];
[thread start];

// runloop保活
[[NSRunLoop currentRunLoop] addPort:[NSPort port] forMode:NSDefaultRunLoopMode];
[[NSRunLoop currentRunLoop] run];
    
// 处理事件
[self performSelector:@selector(test) onThread:thread withObject:nil waitUntilDone:NO];
复制代码
```

## 计时器

### `NSTimer、CADisplayLink、dispatch_source_t` 的优劣

- `NSTimer`
  - 优点在于使用的是`target-action`模式，简单好用
  - 缺点是容易不小心造成循环引用。需要依赖`runloop`，`runloop`如果被阻塞就要延迟到下一次`runloop`周期才执行，所以时间精度上也略为不足
- `CADisplayLink`
  - 优点是精度高，每次刷新结束后都调用，适合不停重绘的计时，例如视频
  - 缺点容易不小心造成循环引用。`selector`循环间隔大于重绘每帧的间隔时间，会导致跳过若干次调用机会。不可以设置单次执行。
- `dispatch_source_t`
  - 基于`GCD`，精度高，不依赖`runloop`，简单好使，最喜欢的计时器
  - 需要注意的点是使用的时候必须持有计时器，不然就会提前释放。

### NSTimer在子线程执行会怎么样？

- `NSTimer`在子线程调用需要手动开启子线程的`runloop`
- `[[NSRunLoop currentRunLoop] run];`

### `NSTimer`为什么不准？

- 如果`runloop`正处在阻塞状态的时候`NSTimer`到达触发时间，`NSTimer`的触发会被推迟到下一个`runloop`周期

### `NSTimer`的循环引用？

`timer`和`target`互相强引用导致了循环引用。可以通过中间件持有`timer & target`解决

### GCD计时器

```
NSTimeInterval interval = 1.0;
_timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0));
dispatch_source_set_timer(_timer, dispatch_walltime(NULL, 0), interval * NSEC_PER_SEC, 0);
dispatch_source_set_event_handler(_timer, ^{
    NSLog(@"GCD timer test");
});
dispatch_resume(_timer);
```

## 渲染

### 屏幕撕裂的原因？

- 单一缓存模式下，帧缓冲区只有一个缓存空间
- 图片需要经过`CPU -> 内存 -> GPU -> 展示` 的渲染过程
- CPU和GPU的协作过程中出现了偏差，GPU应该完整的绘制图片，但是工作慢了只绘制出图片上半部分。
- 此时CPU又把新数据存储到缓冲区，GPU继续绘制的时候下半部分就变成了新数据。
- 造成了两帧同时出现了一部分在屏幕上，看起来就撕裂了。

### 怎么解决屏幕撕裂？

- 解决上一帧和下一帧的覆盖问题，需要使用不同的缓冲区，通过两个图形缓冲区的交替来解决。
- 出现速度差的时候，就把下一帧存储在后备缓冲区，绘制完成后再切换帧。
- 当绘制完最后一个像素点就会发出这个垂直同步信号通知展示完成。
- 所以屏幕撕裂问题需要通过 **双缓冲区 + 垂直同步信号** 解决。

### 掉帧是怎么产生的？

- 屏幕正在展示A帧的时候，CPU和GPU会处理B帧。
- A帧展示完成该切换展示B帧的时候B帧的数据未准备好。
- 没办法切换就只能重复展示A帧，感官上就是卡了，这就是掉帧的问题

### 怎么解决掉帧？

- 掉帧根本原因是CPU和GPU渲染计算耗时过长
- 1、降低视图层级
- 2、提前或减少在渲染期的计算

### CPU渲染职能

- **布局计算**：如果视图层级过于复杂，当试图呈现或者修改的时候，计算图层帧率就会消耗一部分时间，
- **视图懒加载**： `iOS`只会当视图控制器的视图显示到屏幕上才会加载它，这对内存使用和程序启动时间很有好处，但是当呈现到屏幕之前，按下按钮导致的许多工作都不会被及时响应。比如，控制器从数据局中获取数据， 或者视图从一个`xib`加载，或者涉及iO图片显示都会比CPU正常操作慢得多。
- **解压图片**：PNG或者JPEG压缩之后的图片文件会比同质量的位图小得多。但是在图片绘制到屏幕上之前，必须把它扩展成完整的未解压的尺寸(通常等同于图片宽 x 长 x 4个字节)。为了节省内存，`iOS`通常直到真正绘制的时候才去解码图片。根据你加载图片的方式，第一次对图层内容赋值的时候(直接或者间接使用 UIImageView )或者把它绘制到`Core Graphics`中，都需要对它解压，这样的话，对于一个较大的图片，都会占用一定的时间。
- **Core Graphics绘制**：如果对视图实现了`drawRect:`或`drawLayer:inContext:`方法，或者 `CALayerDelegate` 的方法，那么在绘制任何东西之前都会产生一个巨大的性能开销。为了支持对图层内容的任意绘制，`Core Animation`必须创建一个内存中等大小的寄宿图片。然后一旦绘制结束之后， 必须把图片数据通过`IPC`传到渲染服务器。在此基础上，`Core Graphics`绘制就会变得十分缓慢，所以在一个对性能十分挑剔的场景下这样做十分不好。
- **图层打包**：当图层被成功打包，发送到渲染服务器之后，`CPU`仍然要做如下工作:为了显示 屏幕上的图层，`Core Animation`必须对渲染树种的每个可见图层通过`OpenGL`循环转换成纹理三角板。由于`GPU`并不知晓`Core Animation`图层的任何结构，所以必须要由`CPU`做这些事情。这里`CPU`涉及的工作和图层个数成正比，所以如果在你的层 级关系中有太多的图层，就会导致`CPU`没一帧的渲染，即使这些事情不是你的应用 程序可控的。

### GPU渲染职能

- GPU会根据生成的前后帧缓存数据，根据实际情况进行合成，其中造成**GPU渲染负担**的一般是：**离屏渲染，图层混合，延迟加载。**

### 一个`UIImageView`添加到视图上以后，内部如何渲染到手机上的？

- 图片显示分为三个步骤： **加载、解码、渲染**
- 通常，我们程序员的操作只是加载，至于解码和渲染是由UIKit内部进行的。
- 例如：`UIImageView`显示在屏幕上的时候需要`UIImage`对象进行数据源的赋值。而`UIImage`持有的数据是未解码的压缩数据，当赋值的时候，图像数据会被解码变成RGB颜色数据，最终渲染到屏幕上。

### 说说渲染流程

- 1、CPU确定绘制图形的位置，拿到iOS的系统坐标，需要换算成屏幕坐标系。
- 2、转换后确定好顶点的位置，这时候就需要图元装配，这个就是确定顶点间的连线关系。
- 3、确定连接方式以后需要进行光栅化，就是把展示用到的像素点摘出来。
- 4、摘出来以后GPU进行片元着色器处理，计算摘出来的像素点展示的颜色，并存入缓冲区。
- 5、屏幕展示

### 什么是离屏渲染

- 普通渲染流程：APP - 帧缓冲区 - 展示
- 离屏渲染流程：APP - 离屏渲染缓冲区 - 帧缓冲区 - 展示
- 离屏渲染，是无法一次性处理渲染，需要分部处理并存储中间结果引起的。
- 所以判断是否出现离屏渲染的根本条件就是判断渲染是否需要分部处理～
  - 需要分步处理，会产生离屏渲染
  - 一次性渲染，不产生离屏渲染

### 离屏渲染的影响

- 需要分几步就需要开辟出几个离屏渲染缓冲区存储中间结果，造成空间浪费。
- 最后合并多个离屏渲染缓冲区才能展示结果，会影响性能。

### 什么操作会触发离屏渲染?

- 光栅化 `layer.shouldRasterize = YES`
- 遮罩`layer.mask`
- 圆角`layer.maskToBounds = Yes`，`Layer.cornerRadis`
- 阴影`layer.shadowXXX`

## 视图

#### `AutoLayout`的原理，性能如何

- `Auto Layout` 只关注视图之间的关系，通过布局引擎和已有的约束计算出各个视图的`frame`
- 每当约束改变时会重新计算各个视图的`frame`
- 获得`frame`的过程，就是根据各个视图已有的约束条件解方程式的过程。
- 性能会随着视图数量的增加呈指数级增加
- 达到一定数量的视图时，布局所需要的时间就会大于16.67ms，超过屏幕的刷新频率时会出现卡顿。

### 你认为自动布局怎么实现的

- 原理是线性公式，使用了系统提供的`NSLayoutConstraint`
- `Masonry`基于它封装

### `ViewController`生命周期

- `initWithCoder`：通过nib文件初始化时触发。
- `awakeFromNib`：nib文件被加载的时候，会发生一个`awakeFromNib`的消息到`nib`文件中的每个对象。
- `loadView`：开始加载视图控制器自带的`view`。
- `viewDidLoad`：视图控制器的`view`被加载完成。
- `viewWillAppear`：视图控制器的`view`将要显示在`window`上。
- `updateViewConstrains`：视图控制器的`view`开始更新`AutoLayout`约束。
- `viewWillLayoutSubviews`：视图控制器的`view`将要更新内容视图的位置。
- `viewDidLayoutSubviews`：视图控制器的`view`已经更新视图的位置。
- `viewDidAppear`：视图控制器的`view`已经展示到`window`上。
- `viewWillDisappear`：视图控制器的`view`将要从`window`上消失。
- `viewDidDisappear`：视图控制器的`view`已经从`window`上消失。

### `LayoutSubviews`调用时机

- `init` 初始化不会调用 `LayoutSubviews` 方法
- `addsubView` 时候会调用
- 改变一个 `view` 的 `frame` 的时候调用
- 滚动 `UIScrollView` 导致 `UIView` 重新布局的时候会调用
- 手动调用 `setNeedsLayout` 或者 `layoutIfNeeded`

### `layoutIfNeeded`和`setNeedsLayout`的区别

- ```
  setNeedsLayout
  ```

   

  标记为需要重新布局

  - 异步调用`layoutIfNeeded`刷新布局，不立即刷新，在下一轮runloop结束前刷新。
  - 对于这一轮`runloop`之内的所有布局和UI上的更新只会刷新一次，`layoutSubviews`一定会被调用。

- ```
  layoutIfNeeded
  ```

  - 如果有需要刷新的标记，立即调用`layoutSubviews`进行布局
  - 如果没有标记，不会调用`layoutSubviews`
  - 如果想在当前`runloop`中立即刷新，调用顺序应该是

  ```
  [self setNeedsLayout];
  [self layoutIfNeeded];
  复制代码
  ```

### `drawRect`调用时机

- `drawRect`在`loadView`和`ViewDidLoad`之后调用

### `UIView`和`CALayer`是什么关系?

- `View`可以响应并处理用户事件，`CALayer` 不可以。

- 每个 `UIView` 内部都有一个 `CALayer` 提供尺寸样式（模型树），进行绘制和显示。

- 两者都有树状层级结构，`layer` 内部有 `subLayers`，`view` 内部有 `subViews`。

- `CALayer`是支持隐式动画的，`View` 作为`Layer`的代理，通过 `actionForLayer:forKey:`向`Layer`提交相应的动画

- layer 内部维护着三分

   

  ```
  layer tree
  ```

  - 动画树`presentLayer Tree`，修改动画的属性改的是动画树的属性值
  - 模型树`modeLayer Tree`，最终展示在界面上的其实是提供视图的模型树
  - 渲染树`render Tree`。

### `UIView`显示原理

- `UIView` 可以显示是因为内部有一个`layer`作为根图层，根图层上可以放其他子图层。
- `UIView` 中所有能够看到的内容都包含在`layer`中
- 当`UIView`需要显示到屏幕上会调用`drawRect:`方法进行绘图，并且会将所有内容绘制在自己的`layer`上
- 绘图完毕后，系统将图层展示到屏幕上，完成了UIView的显示。

### `UIView`显示过程

- `view.layer`创建一个图层类型的上下文（`Layer Graphics Contex`）
- `view.layer.delegate`，也就是`view`，调用`drawLayer:inContext:`方法，并传入刚才准备好的上下文
- `drawLayer:inContext:`内部会让`view`调用`drawRect:`方法绘图
- 开发者在`drawRect:`方法中实现绘图代码, 所有东西最终都绘制到`view.layer`上面
- 系统将`view.layer`的内容展示 到屏幕, 于是完成了`view`的显示

### `UITableView`的重用机制？

- `UITableView` 通过重用单元格来节省内存
- 为每个单元格指定一个重用标识符，即指定了单元格的种类
- 当屏幕上的单元格滑出屏幕时，系统会把这个单元格添加到重用队列中，等待被重用。
- 当有新单元格从屏幕外滑入屏幕内时，从重用队列查找可重用的单元格，如果有就拿来用，如果没有就创建一个使用。

#### `UITableView`卡顿的的原因有哪些？

- 隐式绘制 `CGContext`
- 文本`CATextLayer` 和 `UILabel`
- 光栅化 `shouldRasterize`
- 离屏渲染
- 可伸缩图片
- `shadowPath`
- 混合和过度绘制
- 减少图层数量
- 裁切
- 对象回收
- `Core Graphics`绘制
- `-renderInContext:` 方法

### `UITableVIew`优化

- 重用机制（缓存池）
- 少用有透明度的`View`
- 尽量避免使用`xib`
- 尽量避免过多的层级结构
- iOS8以后出的预估高度
- 减少离屏渲染操作（圆角、阴影）
- 缓存`cell`的高度（提前计算好`cell`的高度，缓存进当前的模型里面）
- 异步绘制
- 滑动的时候，按需加载
- 尽量少`add、remove` 子控件，最好通过`hidden`控制显示

### `imageName`与`imageWithContentsOfFile`区别？

- `imageWithContentsOfFile`：加载本地目录图片，不能加载`image.xcassets`里面的图片资源。不缓存占用内存小，相同的图片会被重复加载到内存中。不需要重复读取的时候使用。
- `imageNamed`：可以读取`image.xcassets`的图片资源，加载到内存缓存起来，占用内存较大，相同的图片不会被重复加载。直到收到内存警告的时候才会释放不使用的`UIImage`。需要重复读取同一个图片的时候用。

### `IBOutlet`连出来的视图属性为什么可以被设置成`weak`?

- 因为`Xcode`内部把链接的控件放进了一个`_topLevelObjectsToKeepAliveFromStoryboard`的私有数组中，这个数组强引用这所有`topLevel`的对象，所以用`weak`也无伤大雅。

### `UIScrollerView`实现原理

- 滚动其实是在修改原点坐标。当手指触摸后，`scrollview`拦截触摸事件创建一个计时器。
- 如果计时器到点后没有发生手指移动事件，`scrollview` 发送 `tracking events` 到被点击的 `subview`。
- 如果计时器到点前发生了移动事件， `scrollview` 取消 `tracking` 自己滚动。

### 如何实现视图的变形?

- 修改`view`的 `transform`。

## 事件响应链和事件传递

### 什么是响应链

- 由链接在一起的响应者（`UIResponse`及其子类）组成的链式关系。
- 最先的响应者称为第一响应者
- 最底层的响应者是`UIApplication`

### 写出一个响应链

```
subView -> view -> superView -> viewController -> window -> application
```

### 什么是事件传递

- 触发事件后，事件从第一响应者到`application`的传递过程

### 事件传递的过程

- 当程序中发生触摸事件之后，系统会将事件添加到`UIApplication`管理的一个队列当中
- `application`将任务队列的首个任务向下分发
- `application -> window -> viewController -> view`
- `view`需要满足条件才可以处理任务，透明度>0.01、触摸在`view`的区域内、`userInteractionEnabled=YES`、`hidden=NO`。
- 满足条件的`view`遍历自身的`subViews`，判断是否满足上述条件
- 如果所有`subView`都无法满足条件，那么最佳响应者就是自己。
- 如果没有任何一个`view`能处理事件，事件会被废弃。

### 找出触摸的View

```
// 返回的View是本次点击的最佳响应者
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event

// 判断点是否落在某区域
- (BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event
复制代码
```

### 重写`hitTest:withEvent`

```
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event {
   // 1.判断当前控件能否接收事件
   if (self.userInteractionEnabled == NO || 
       self.hidden == YES || 
       self.alpha <= 0.01) {
        return nil;
    }
   // 2. 判断点在不在当前控件
   if ([self pointInside:point withEvent:event] == NO) {
        return nil;
   }
   // 3.从后往前遍历自己的子控件
   NSInteger count = self.subviews.count;
   for (NSInteger i = count - 1; i >= 0; i--) {
      UIView *childView = self.subviews[I];
      // 把当前控件上的坐标系转换成子控件上的坐标系
      CGPoint childP = [self convertPoint:point toView:childView];
      UIView *fitView = [childView hitTest:childP withEvent:event];
       if (fitView) { // 寻找到最合适的view
           return fitView;
       }
   }
   // 循环结束 没有比自己更合适的view
   return self;
}
复制代码
```

## Crash

### 常见Crash的原因有哪些？

- 1、找不到方法的实现 `unrecognized selector sent to instance`
- 2、`KVC`造成的crash
- 3、`KVO`造成的crash
- 4、`EXC_BAD_ACCESS`
- 5、集合类相关崩溃，越界等
- 6、多线程中的崩溃，使用了被释放的对象
- 7、后台返回错误的数据结构

### `BAD_ACCESS`在什么情况下出现？

- 这种问题在开发时经常遇到。原因是访问了野指针，比如访问已经释放对象的成员变量或者发消息、死循环等。

### 不使用第三方，排查闪退问题？

- 1、使用`NSSetUncaughtExceptionHandler`统计闪退的信息

- 2、将统计到的信息发给后台

- 3、在后台收集信息，进行排查

  ```
  static void my_uncaught_exception_handler (NSException *exception) {
      //获取NSException 信息
      NSLog(@"***********************************************");
      NSLog(@"%@",exception);
      NSLog(@"%@",exception.callStackReturnAddresses);
      NSLog(@"%@",exception.callStackSymbols);
      NSLog(@"***********************************************");
  }
      
  - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
      NSSetUncaughtExceptionHandler(&my_uncaught_exception_handler);
      return YES;
  }
  复制代码
  ```

## 性能优化

### 优化启动时间

- 启动时间是用户点击App图标，到第一个界面展示的时间。
- 启动时间在小于400ms是最佳的，因为从点击图标到显示`Launch Screen`，到`Launch Screen`消失这段时间是400ms。
- 启动时间不可以大于20s，否则会被系统杀掉。

以`main`函数作为分水岭，启动时间其实包括了两部分：

**`mian`函数之前的启动优化**

- 减少或合并动态库（这是目前为止最耗时的了， 基本上占了95%以上的时间）
- 确认动态库是`optional or required`。如果该`Framework`在支持的所有iOS系统版本都存在，那么就设为`required`，否则就设为`optional`，因为`optional`会有些额外的检查

**`mian`函数之后的启动优化** 首先分析一下从main函数开始执行，到第一个页面显示， 这段时间做了哪些事情

- 1、减少创建线程，线程创建和运行平均消耗大约在 29 毫秒，这是很大的时间开销。若在应用启动时开启多个线程，则尤为明显。线程的启动时间之所以如此之长，是因为多次的上下文切换所带来的开销。所以线程在开发过程中也避免滥用
- 2、编译器插桩获取方法符号，生成`order file`设置到`xcode`。减少页中断带来的耗时。
- 3、合并或者删减不必要的类或者分类
- 4、将不必需在`+load`方法中做的事情，延时放到`+initialize`。
- 5、 SDK 和配置事件，由于启动时间不是必须的，所以我们可以放在第一个界面的 `viewDidAppear` 方法里

### 网络优化

- `IP`直连，将我们原本的请求链接`www.baidu.com` 修改为 `180.149.132.47`
- 运营商在拿到请求后发现是`IP`地址会直接放行，而不会去走DNS解析
- 不走他的`DNS`解析也就不会存在DNS被劫持的问题
- 实现方法1：接使用HTTPDNS等sdk
- 实现方法2：服务端下发`发域名-IP`对应列表，客户端缓存，通过缓存IP来进行业务访问。

### 包体积优化

- 1、删除陈旧代码、删除陈旧xib/sb，删除无用的资源文件（检测未使用图片的工具`LSUnusedResources`）
- 2、图片、音视频资源压缩后使用。
- 3、动图可以使用`webP`格式，加载速度比较慢，但体积小
- 4、能用动态库的尽量用动态库，一般情况静态库的体积会比动态库大
- 5、主题类的资源文件提供按需下载功能，不直接打包在应用包里面
- 6、`App Slicing`，应用程序切片，只打包下载机型所需的资源文件，不需要开发者处理
- 7、`Bitcode`，中间代码， 苹果会对可执行文件进行优化，不需要开发者处理
- 8、`ODR，On Demand Resources`，按需加载资源，需要开发者处理

### 电量优化

- 1.定位，尽量不要实时更新，可以适当降低精度
- 2.网络请求，能用缓存的时候尽量使用缓存，降低请求的频率，减少请求的次数，优化传输结构
- 3.CPU处理，需要复用的数据能缓存的数据尽量缓存，优化算法和数据结构
- 4.GPU处理，减少离屏渲染

### 短视频/直播优化

- 多播放器，像`tableview`那样维护缓存池。多播放器，才能做预加载。
- 边下边播，要实现下一个视频或者几个视频能快速的播放起来，首先应该保证正在播放的短视频能顺畅播放，边下边播任务优先级应该高于预加载任务。没有边下边播时才能执行预加载，边下边播任务进行时应当停止预加载。
- 预加载，预加载的下载应该和边下边播区分开，手势滑动到差不多出现的时候，开始播放预加载的那`1m`数据。滑动下一个的时候，就取消`preload`，播放那`1m`下载的数据，再继续`load`。同时调起下一个`preload`准备下载。

### 短视频缓存管理

- 缓存主要创建了三个目录管理，分别为`temp、media、trash`目录
- 缓存分为临时缓存和最终缓存，当短视频资源未下载完时是放在`temp`、当视频资源缓存完时移动到`media`，这样分别存放便能方便读取和管理两种状态的缓存，
- 所有需要删除的缓存文件都移入`trash`，随后再删除以此来保证较高的删除效率。所有文件命名使用`视频url的md5值`保证唯一性。
- 缓存应该具有自动管理功能，默认配置下`ShortMediaCache`允许临时缓存最多保存1天，最大`100Mb`，而最终缓存则允许最多保存2天最大`200Mb`，如果业务需要可以自定义`ShortMediaCacheConfig`配置实现。

### 一般是怎么用`Instruments`的？

- `Instruments`里面工具很多，常用：
- `Time Profiler`: 性能分析
- `Zombies`：检查是否访问了僵尸对象，但是这个工具只能从上往下检查，不智能。
- `Allocations`：用来检查内存，写算法的那批人也用这个来检查。
- `Leaks`：检查内存，看是否有内存泄露。

## 编译 & 启动

### 编译链接流程

- 0、输入文件：找到源文件

- 1、预处理：包括替换宏和导入头文件

- 2、编译阶段：词法分析、语法分析、语义分析，最终生成IR

  - 2-1、预处理后会进行词法分析，词法分析会把代码切片成一个个`token`。
  - 2-2、语法分析会验证语法是否正确，在词法分析的基础上把单词组合成各种语法短语，然后把节点组成抽象的语法树
  - 2-3、代码生成器根据语法树自顶向下生成`LLVM IR`。`OC`会在这里进行`runtime`的桥接：`property`的合成、`ARC`处理等

- 3、编译器后端：通过一个个

  ```
  Pass
  ```

  去优化，每个

  ```
  Pass
  ```

  完成一部分功能，最终生成汇编代码

  - 3-1、苹果对代码做了进一步优化，并且通过`.ll`文件生成`.bc`文件。
  - 3-2、可以通过`.bc`或`.ll`文件生成汇编代码

- 4、生成目标文件，`.o`格式的目标文件

- 5、链链接需要的动态库和静态库

- 6、通过不同架构，生成对应的可执行文件`MachO`

### APP启动过程

- 1、加载可执行文件（读取`Mach-O`）

- 2、加载动态库`（Dylib`）

- 3、

  ```
  Rebase & Bind
  ```

  - 3-1、`Rebase`的作用是修正`ASLR`的偏移，把当前`MachO`的指针指向正确的内存
  - 3-2、`Bind`的作用是重新修复外部方法指针的指向，`fishhook`原理

- 4、`Objc`，加载类和分类那套

- 5、`Initializers`，调用`load`方法，初始化`C & C++`的对象等

- 6、`main()`函数

- 7、执行`AppDelegate`的代理方法（如：`didFinishLaunchingWithOptions`）。

- 8、初始化`Windows`，初始化`ViewController`。

### dyld做了什么

- 1、`dyld`读取`Mach-O`的`Header`和`Load Commands`
- 2、找可执行文件的依赖动态库，将依赖的动态库加载到内存中。这是一个递归的过程，依赖的动态库可能还会依赖别的动态库，所以`dyld`会递归每个动态库，直至所有的依赖库都被加载完毕。

## 内存管理

### 堆和栈区的区别

- 栈
  - 栈由系统分配和管理
  - 栈的内存增长是向下的
  - 栈内存速率比堆快
  - 栈的大小一般默认为1M，但可以在编译器中设置
  - 操作系统中具有专门的寄存器存储栈指针，以及有相应的硬件指令去操作栈内存分配
- 堆
  - 堆由开发者申请和管理
  - 堆的内存增长是向上的
  - 堆内存速率比栈慢
  - 内存比较大，一般会达到4G
  - 忘记释放会造成内存泄漏

### 堆为什么默认4G？

- 系统是32位的，最多只支持32位的2进制数来表示内存地址
- 2^32 = 4G，没法表示比4G更大的数字了，所以寻址只能寻到 4G

### 机器内存条16G，虚拟内存只有4G，岂不是浪费？

- 虚拟内存大小和物理内存大小无关
- 虚拟内存是物理内存不够用时把一部分硬盘空间做为内存来使用
- 由于硬盘传输的速度要比内存传输速度慢的多，所以使用虚拟内存比物理内存效率要慢

### 一个进程的地址和物理地址之间的关系是什么？

- CPU能够访问到的是进程中记录的逻辑地址，
- 使用页式内存管理方案，逻辑地址包括页号和页内偏移量
- 页号可以在页表中查询得到物理内存中划分的页
- 找到页以后用进程的起始地址拼接上页内偏移量可以得到实际物理地址

### 这样有什么更快的方法去计算物理地址？

- TLB快表

### 同一个进程里哪些资源是线程间共享的，哪些是独有的。

- 堆：所有线程共有的
- 栈：单个线程私有的

### 哪些变量保存在堆里，哪些保存在栈里

- 指针在栈里，对象在堆里，指针指向对象。

### 什么是野指针？

- 指向被释放/回收对象的指针。

### 如何检测野指针？

引用`Bugly`工程师陈其锋的思路，`fishhook free`函数，把释放的空间填入`0x55`。`XCode`的僵尸对象填充的就是`0x55`。这样可以使偶现的野指针问题问题（对象释放仍被调用）变为必现，方便排查。

```
bool init_safe_free() {
    _unfreeQueue = ds_queue_create(MAX_STEAL_MEM_NUM);
    orig_free = (void(*)(void*))dlsym(RTLD_DEFAULT, "free");
    rebind_symbols((struct rebinding[]){{"free", (void*)safe_free}}, 1);
    return true;
}
复制代码
void safe_free(void* p){
    size_tmemSiziee=malloc_size(p);
    memset(p,0x55, memSiziee);
    orig_free(p);
    return;
}
复制代码
```

但是如果上述内存被重新填充了可用数据，就无法检测到了。

所以其实可以直接在替换的`free函数`中做更多的操作。

用哈希表记录需要别释放的对象，但实际上并不释放，只是把里面的数据替换成0x55，该指针再被调用时就会crash。

在发生内存警告的时候再清理一部分内存。

这种改动不可以出现在线上版本，只能用于排查crash。

```
DSQueue* _unfreeQueue=NULL;//用来保存自己偷偷保留的内存:1这个队列要线程安全或者自己加锁;2这个队列内部应该尽量少申请和释放堆内存。
int unfreeSize=0;//用来记录我们偷偷保存的内存的大小

#define MAX_STEAL_MEM_SIZE 1024*1024*100//最多存这么多内存，大于这个值就释放一部分
#define MAX_STEAL_MEM_NUM 1024*1024*10//最多保留这么多个指针，再多就释放一部分
#define BATCH_FREE_NUM 100//每次释放的时候释放指针数量

//系统内存警告的时候调用这个函数释放一些内存
void free_some_mem(size_t freeNum){
    size_t count=ds_queue_length(_unfreeQueue);
    freeNum=freeNum>count?count:freeNum;
    for (int i=0; i<freeNum; i++) {
        void* unfreePoint=ds_queue_get(_unfreeQueue);
        size_t memSiziee=malloc_size(unfreePoint);
        __sync_fetch_and_sub(&unfreeSize,memSiziee);
        orig_free(unfreePoint);
    }
}

void safe_free(void* p){
#if 0//之前的代码我们先注释掉
    size_t memSiziee=malloc_size(p);
    memset(p, 0x55, memSiziee);
    orig_free(p);
#else
    int unFreeCount=ds_queue_length(_unfreeQueue);
    if (unFreeCount>MAX_STEAL_MEM_NUM*0.9 || unfreeSize>MAX_STEAL_MEM_SIZE) {
        free_some_mem(BATCH_FREE_NUM);
    }else{
        size_t memSiziee=malloc_size(p);
        memset(p, 0x55, memSiziee);
        __sync_fetch_and_add(&unfreeSize,memSiziee);
        ds_queue_put(_unfreeQueue, p);
    }
#endif

    return;
}
bool init_safe_free()
{
    _unfreeQueue=ds_queue_create(MAX_STEAL_MEM_NUM);

    orig_free=(void(*)(void*))dlsym(RTLD_DEFAULT, "free");
    rebind_symbols1((struct rebinding[]){{"free", (void*)safe_free}}, 1);

    return true;
}
- (void)applicationDidReceiveMemoryWarning:(UIApplication *)application
{
    free_some_mem(1024*1024);
}
复制代码
```

### 简单谈一下内存管理

- 通过引用计数管理对象的释放时机。创建的时候引用计数+1，出现新的持有关系的时候引用计数+1。当持有对象放弃持有的时候引用计数-1，当对象的引用计数减至0的时候，就要把对象释放。MRC模式需要手动管理引用计数。ARC模式引用计数交由系统管理
- 自动释放池`AutoReleasePool`是`OC`的一种内存自动回收机制，回收统一释放声明为`autorelease`的对象；系统中有多个内存池，系统内存不足时，取出栈顶的池子把引用计数为0的对象释放掉，回收内存給当前应用程序使用。 自动释放池本身销毁的时候，里面所有的对象都会做一次`release`。

### `autoreleasepool`的使用场景

- 创建了大量对象的时候，例如循环的时候

### `autoreleasePool`的数据结构

- `autoreleasePool`底层是`AutoreleasePoolPage`
- 可以理解为双向链表，每张链表头尾相接，有 `parent`、`child`指针
- 每次初始化调用`objc_autoreleasePoolPush`，会在首部创建一个哨兵对象作为标记，释放的时候就以哨兵为止
- 最外层池子的顶端会有一个`next`指针。当链表容量满了（4096字节，一页虚拟内存的大小），就会在链表的顶端，并指向下一张表。

### `autoreleasePool` 什么时候被释放？

- `ARC中所`有的新生对象都是自动添加`autorelese`的
- `@atuorelesepool`解决了大部分内存暴增的问题。
- `autoreleasepool`中的对象在当前`runloop`循环结束的时候自动释放。

### 子线程中的`autorelease`变量什么时候释放？

- 子线程中会默认生成一个`autoreleasepool`， 当线程退出的时候释放。

### `autoreleasepool`是如何实现的？

- `@autoreleasepool{}` 本质是一个结构体
- `autoreleasepool`会被转换成`__AtAutoreleasePool`
- `__AtAutoreleasePool` 里面有`objc_autoreleasePoolPush`、`objc_autoreleasePoolPop`两个关键函数
- 最终调用的是`AutoreleasePoolPage`的 `push` 和 `pop` 方法
- `push`是压栈，`pop`是出栈，`pop`的时候以哨兵作为参数，对所有晚于哨兵插入的对象发送`release`消息进行释放

### 放入`@autuReleasePool`的对象，当自动释放池调用`drain`方法时，一定会释放吗

- `drain`和`release`都会促使自动释放池对象向池内的每一个对象发送release消息来释放池内对象的引用计数
- `release`触发的操作，不会考虑对象是否需要`release`，
- `drain`会在自动释放池向池内对象发送`release`消息的时候，考虑对象是否需要`release`
- 对象是否释放取决于引用计数是否为0，池子是否释放还是取决于里面的所有对象是否引用计数都为0。

### `@aotuReleasePool`的嵌套使用，对象内存是如何被释放的

- 每次初始化调用`objc_autoreleasePoolPush`，会在首部创建一个哨兵对象作为标记
- 释放的时候就会依次对每个pool里晚于哨兵的对象都进行`release`
- 从内到外的顺序释放

### ARC环境下有内存泄漏吗？举例说明

- 有。例如两个`strong`修饰的对象相互引用。
- `block`中的循环引用
- `NSTimer`的循环引用
- `delegate`的强引用
- 非`OC`对象的内存处理（需手动释放）

### 出现内存泄漏，该如何解决？

- 使用`Instrument`当中的`Leak`检测工具
- 使用僵尸变量，根据打印日志，然后分析原因，找出内存泄漏的地方

### `ARC`对`reatain & release`优化了什么

- 根据上下文及阴影关系，减少了不必要的`retain`和`release`
- 例如`MRC`环境下引用一个`autorelease`对象，对象会经历`new -> autorelease -> retain -> release`，但是仅仅只是引用而已，中间的`autorelease`和`retain`操作其实可以去除，所以`ARC`就是把这两步不需要的操作优化掉了

### MRC转成ARC管理，需要注意什么

- 去掉所有的`retain，release，autorelease`
- `NSAutoRelease`替换成`@autoreleasepool{ }`块
- `assign`修饰的属性需要根据ARC规定改写
- `dealloc`方法来管理一些资源释放，但不能释放实例变量，`dealloc`里面去掉`[super dealloc]`，ARC下父类`dealloc`由编译器来自动完成
- `Core Foundation`的对象可以用`CFRetain，CFRelease`这些方法
- 不能在使用`NSAllocateObject、NSDeallocateObject`
- `void * 和 id`类型的转换，oc对象和c对象的转换需要特定函数

### 实际开发中，如何对内存进行优化呢？

- 使用ARC管理内存
- 使用`Autorelease Pool`
- 优化算法
- 避免循环引用
- 定期使用`Instrument`的`Leak`检测内存泄漏

### 结构体对齐方式

```
struct { 
  char a;
  double b;
  int c;
} 
char    1
short   2
int     4
float   4
long    8
double  8
复制代码
```

### new和malloc的区别

- `new`调用了实例方法初始化对象，`alloc + init`
- `malloc`函数从堆上动态分配内存，没有`init`

### `delete`和`free`的区别

- ```
  delete
  ```

  是一个运算符，做了两件事

  - 调用析构函数
  - 调用`free`释放内存

- `free()` 是一个函数

### 内存分布，常量是存放在哪里（重点！）

- 栈区
- 堆区
- 全局静态区
- 代码区

## weak

### weak是怎么实现的

- `weak`通过`SideTable`实现，`SideTable`里面包含了一个锁，一个引用计数表，一个弱引用表
- `weak`关键字修饰的对象会被记录到弱引用表里面
- `weak_table_t`里面有一个数组记录多个弱引用对象（`weak_entry_t`），每个`weak_entry_t`对应一个被弱引用的OC对象
- `weak_entry_t`里面有记录弱引用指针的数组，存放的是`weak_referrer_t`，每个`weak_referrer_t`对应一个弱引用指针
- 创建的时候判断是否已经创建了`weak_entry_t`，有的话就把新的`weak_referrer_t`插入数组，没有的话就创建`weak_referrer_t`和`weak_entry_t`一起插入到表里。
- 添加的时候还会进行容量判断，如果超过3/4就会容量乘以2进行扩容。
- `SideTable`最多只能存储64个节点

### 为什么需要多张SideTable

每个对象都有可能被弱引用，如果都存在一个表里，不同线程、不同操作对这个单表频繁的加锁和解锁，这样处理起事务更容易出现问题。

### weak对象为什么可以自动置为nil

- `dealloc`的过程里面有一步是调用`clear_weak_no_lock`，会取出弱引用表遍历每个弱引用对象置为`nil`
- `dealloc -> rootDealloc -> object_dispose -> obj_desturctInstance -> clearDeallocating -> clearDeallocating_slow -> weak_clear_no_lock`

## 单例

### 什么是单例

- 只有一个实例对象。而且向整个系统提供这个实例。

### 你实现过单例模式么？ 你能用几种实现方案？

```
+ (instancetype)shareInstance {
    static ShareObject *share = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        share = [[super allocWithZone:NULL] init];
    });
    return share;
}

+ (instancetype)allocWithZone:(struct _NSZone *)zone {
    return [self shareInstance];
}

- (id)copyWithZone:(NSZone *)zone {
    return self;
}
复制代码
```

### 单例怎么销毁

- `dispatch_once` 当 `onceToken`为0的时候才会被调用，调用完成后`onceToken`会被置为-1
- 必须把`onceToken`变成全局的，在需要的时候重置为0

```
+ (void)removeShareInstance {
    //置0，下次调用shareInstance才会再次创建对象
    onceToken = 0; 
    _sharedInstance = nil;
}
复制代码
```

### 不使用`dispatch_once`如何实现单例

- 重写`allocWithZone:`方法;

```
+ (instancetype)allocWithZone:(struct _NSZone *)zone {
    static id instance = nil;
    @synchronized (self) {
        if (instance == nil) {
            instance = [super allocWithZone:zone];
        }
    }
    return instance;
}
复制代码
```

### 项目开发中，你用单例都做了什么？

- 用户登录后，用`NSUserDefaults`存储用户信息，采用单例封装方便全局访问
- IM聊天管理器使用单例，方便全局访问

## Block

### 什么是`block`？

- 闭包，可以获取其它函数局部变量的匿名函数。

### `block` 的内部实现

- `block`是个对象，`block`的底层结构题也有`isa`，这个`isa`会指向`block`的类型

- ```
  block
  ```

  的底层结构体是

   

  ```
  __main_block_impl_0
  ```

  ，存储了下列数据

  - 方法实现的指针`impl`
  - `block`的相关信息`Desc`
  - 如果有捕获外部变量，结构体内还会存储捕获的变量。

- 使用`block`时就会根据`impl`找到方法所在，传入存储的变量调用。

### `block`的类型

`block`有3种类型，可以通过调用class方法或者isa指针查看具体类型

- `__NSGlobalBlock__ （ _NSConcreteGlobalBlock ）`，存在全局区
- `__NSStackBlock__ （ _NSConcreteStackBlock ）`，存在栈区
- `__NSMallocBlock__ （ _NSConcreteMallocBlock ）`，存在堆区

### `int`变量被 `__block` 修饰与否的区别？

- `block`对未经`__block`修饰的`int`变量的引用是值拷贝，在block中是不能改变外部变量的。
- 通过`__block`修饰后的`int`变量，`block`对这个变量的引用是指针引用。它会生成一个结构体复制这个变量的指针，从而达到可以修改外部变量的作用。

### `block`在修改`NSMutableArray`，需不需要添加`__block`

- 不需要，不改变数组指针的指向，只是添加数组内容

### `block` 捕获外部局部变量实际上发生了什么？`__block`又做了什么？

- `block`捕获外部变量的时候，会记录下外部变量的瞬时值，存储在`block_impl_0`结构体里
- `__block` 所起到的作用就是只要观察到该变量被 `block` 所持有，就将“外部变量”在栈中的内存地址放到了堆中。进而在`block`内部也可以修改外部变量的值。
- 总之，`block`内部可以修改堆中的内容， 不可以直接修改栈中的内容。

### 在`ARC`和`MRC`下`block`访问对象类型的变量时，有什么区别

- `ARC`环境会根据外部变量是`__strong`还是`__weak`修饰进行引用计数管理，达到强引用或弱引用的效果
- `MRC环`境下，`block`属于栈区，外部变量是`auto`修饰的，不手动`copy`的话变量就不会被`block`强引用。

### `block`可以用`strong`修饰吗

- `MRC`环境下，不可以。`strong`只会把`block`进行一次`retain`操作，栈上的`block`不会被复制到堆区，依旧无法共享
- `ARC`环境下，可以。`block`在堆区，而且`block`的`retain`操作也是通过`copy`完成

### `block`为什么用`copy`修饰？

- `MRC`环境，`block`创建在栈区，只要函数作用域消失就会被释放，外部再去调用就会崩溃。通过`copy`修饰可以把它复制到堆区，外部调用也没问题，从而解决了这个问题。
- `ARC`环境，`block`创建在堆区，用`strong`和`copy`都一样。`block`的`retain`操作也是通过`copy`完成。以前用`copy`就一直延续了。

### `block`在什么情况下会被`copy`

- 主动调用`copy`方法
- 当`block`作为返回值时
- 将`block`赋值给`__strong`指针时
- `block`作为`GCD API`的方法参数时
- `block`作为`Cocoa API`方法名含有`usingBlock`的方法参数时

### `block`的内存管理

- `block`通过`block_copy`、`block_release`两个方法管理内存
- `NSGlobalBlock`，使用`retain、copy、release`都不会不会改变引用计数，`copy`方法不会复制，只会返回`block`的指针
- `NSStackBlock`，使用`retain、release`都不会改变引用计数，`使用copy`会把`block`复制到堆区
- `NSMallocBlock`，使用`retain、copy`会增加一次引用，使用`release`会减少一次引用
- 被`block`引用到外部变量，如果`block`存在堆区或者被复制到堆区，变量的引用计数+1，`block`释放后-1

### 解决循环引用时为什么要用`__strong、__weak`修饰

- 在`block`外部使用`__weak`修饰外部引用对象，可以打破互相持有造成的循环引用
- 在`block`中使用`__strong`修饰外部引用对象，`block`强持有外部变量，可以防止外部变量被提前释放

### 在`Masonry`的`block`中，使用`self`，会造成循环引用吗？如果是在普通的`block`中呢？

- 不会，因为这是个栈`block`，没有延迟使用，使用后立刻释放
- 普通的`block`会，一般会使用强引用持有，就会触发`copy`操作

### 在普通的`block`中只使用下划线属性去访问，会造成循环引用吗

- 会，和调用`self.`是一样的

## NSNotification

### 消息通知的理解

- 通知（`NSNotification`）支持一对多的信息传递方式
- 使用时先注册绑定接收通知的方法，然后通知中心创建并发送通知
- 不再监听时需要移除通知

### 实现原理（结构设计、通知如何存储的、`name & observer & SEL`之间的关系等）

- `Observation`是通知观察对象，存储通知名、`object`、`SEL`的结构体

- ```
  NSNotificationCenter
  ```

  持有一个根容器

  ```
  NCTable
  ```

  ，根容器里面包含三个张链表

  - `wildCards`，存放没有`name & object`的通知观察对象（`Observation`）
  - `nameless`，存放没有`name`但是有`object`的通知观察对象（`Observation`）
  - `named`，存放有`name & object`的通知观察对象（`Observation`）

- 当添加通知观察的时候，

  ```
  NSNotificationCenter
  ```

  根据传入参数是否齐全，创建

  ```
  Observation
  ```

  并添加到不同链表

  - 创建一个新的通知观察对象（`Observation`）
  - 如果传入参数包含名称，在`named`表里查询对应名称，如果已经存在同名的通知观察对象，将新的通知观察对象插入其后，如果不存在则添加到表尾。存储结构为链表，节点内先以`name`作为key，一个字典作为`value`。如果通知参数带有`object`，字典内以`object`为`key`，以`Observation`作为`value`。
  - 如果传入的参数如果只包含`object`，在`nameless`表查询对应名称，将新的通知观察对象插入其后，如果不存在则添加到表尾。存储结构为链表，节点内以`object`为`key`，以`Observation`作为`value`。
  - 如果传入参数没有`name`也没有`object`，直接添加到`wildCards`表尾。结构为链表，节点内存储`Observation`。

### 通知的发送是同步的，还是异步的

- 通知的接收和发送是在一个线程里，实际上发送通知都是同步的，不存在异步操作
- 通知提供了枚举设置发送时机
- `NSPostWhenIdle`，`runloop`空闲的时候发送
- `NSPostASAP`，尽快发送，会穿插在事件完成的空隙中发送
- `NSPostNow`，立刻发送或合并完成后发送

### `NSNotificationCenter` 接受消息和发送消息是在一个线程里吗？如何异步发送消息

- 是的
- 异步发送，也就是延迟发送，可以使用`addObserverForName：object: queue: usingBlock:`

### `NSNotificationQueue`是异步还是同步发送？在哪个线程响应

- 异步发送，也就是延迟发送
- 在同一个线程发送和响应

### `NSNotificationQueue`和`runloop`的关系

- `NSNotificationQueue`只是把通知添加到通知队列，并不会主动发送
- `NSNotificationQueue`依赖`runloop`，如果线程`runloop`没开启就不生效。
- `NSNotificationQueue`发送通知需要`runloop`循环中会触发`NotifyASAP`和`NotifyIdle`从而调用`NSNotificationCenter`
- `NSNotificationCenter` 内部的发送方法其实是同步的，所以`NSNotificationQueue`的异步发送其实是延迟发送。

### 如何保证通知接收的线程在主线程

- 1、在主线程发送通知
- 2、使用`addObserverForName: object: queue: usingBlock`方法注册通知，指定在主线程处理

### 页面销毁时不移除通知会崩溃吗

- iOS9之前会，因为强引用观察者
- iOS9之后不会，因为改为了弱引用观察者

### 多次添加同一个通知会是什么结果？多次移除通知呢

- 多次添加，重复触发，因为在添加的时候不会做去重操作
- 多次移除不会发生崩溃

### 下面的方式能接收到通知吗？为什么

```
// 注册通知
[[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleNotification:) name:@"TestNotification" object:@1];
// 发送通知
[NSNotificationCenter.defaultCenter postNotificationName:@"TestNotification" object:nil];
复制代码
```

- 不能
- 这个通知存储在`named`表里，原本记录的通知观察对象内部会用`object`作为字典里的`key`，查找的时候没了`object`无法找到对应观察者和处理方法。

## 其他模式

### 继承与组合的优缺点

- 继承
  - 通过父类派生子类
  - A继承自B，可以理解为A是B的某一种分支，B的变化会对A产生影响
    - 优点：
      - 易于使用、扩展继承自父类的能力
    - 缺点：
      - 都是白盒复用，父类的细节一般会暴露给子类
      - 父类修改时，除非子类自行实现，否则子类会跟随变化
- 组合
  - 设计类的时候把需要组合的类（成员）的对象加入到该类（容器）中作为成员变量。
  - 例如眼(Eye)、鼻(Nose)、口(Mouth)、耳(Ear)是头(Head)的一部分
  - 容器类（头）仅能通过被包含对象（眼耳口鼻）的接口来对其进行访问。
    - 优点：
      - 黑盒复用，因为被包含对象的内部细节对外是不可见。
      - 封装性好，每一个类只专注于一项任务，实现上的相互依赖性比较小。
    - 缺点：
      - 导致系统中的对象过多
      - 为了组成组合，必须仔细地对成员的接口进行定义

### 工厂模式是什么，工厂模式和抽象工厂的区别

- 工厂模式，定义一个用于创建对象的接口，让子类决定实例化哪一个类。工厂方法使一个类的实例化延迟到其子类。
- 抽象工厂，使用了工厂模式后工厂提供的能力非常多，需要分类这些工厂，就可以根据工厂的共性进行抽象合并。
- 抽象工厂其实就是帮助减少工厂数量的，前提条件就这些工厂要具备两个及以上的共性。

### 原型模式是什么

- 通过复制原型实例创建新的对象。
- 需要遵循`NSCoping`协议 并重写`copyWithZone`方法

## 网络基础

### 什么是 `TCP / UDP` ?

- `TCP`：传输控制协议。面向连接的，建立连接需要经历三次握手，是可靠的传输层协议。
- `UDP`：用户数据协议。是面向无连接的，数据传输快但不可靠，它只管发，不管收不收得到。

### `TCP`和`UDP`的区别

- `TCP`是传输控制协议，是面向字节流的可靠传输，通过分组编号，确认应答，超时重发，流量控制和拥塞控制机制保证数据分组正确有序完整的传输到接收方
- `UDP`是用户数据包协议，不具有TCP的以上机制来保证可靠传输，是以数据报的形式发出，从最下层发出后，它认为发送成功，是不具有保序 正确和完整传输的性质。

### TCP是如何保证可靠性

- 1、校验和，检验范围包括TCP首部及数据部分。将报文分段，所有段反码相加，最终结果为全1
- 2、确认应答与序列号，按序，不缺，去重，多次发送，一次确认
- 3、超时重传
- 4、连接管理，三次握手四次挥手
- 5、流量限制，16位的窗口。
- 6、拥塞控制，慢启动、拥塞避免、拥塞发生、快重传、快恢复

### TCP拥塞控制

- 1）慢启动
- 2）拥塞避免
- 3）拥塞发生
- 4）快速恢复

### 什么是XMPP？

- XMPP是以XML为基础的开放式实时通信协议。

### HTTP建立连接的过程？

- 三次握手

### TCP三次握手和四次挥手？

三次握手

- 1、客户端向服务端发起请求链接，发送报文，`seq=x`，客户端进入SYN_SENT状态
- 2、服务端收到请求链接，回复确认报文，`seq=y`，`ACK=x+1`，并且服务端进入到SYN_RCVD状态
- 3、客户端收到确认报文后，回复报文，`ACK=y+1`，此时客户端进入到ESTABLISHED，服务端收到用户端发送过来的确认报文后，也进入到ESTABLISHED状态，此时链接创建成功

四次挥手

- 1、客户端向服务端发起关闭链接请求，并停止发送数据
- 2、服务端收到关闭请求时回复，收到，然后停止接收数据
- 3、当服务端发送数据结束之后，向客户端发起关闭链接请求，并停止发送数据
- 4、客户端收到关闭请求时回复，收到，然后停止接收数据

### 为什么需要三次握手？

- 防止已失效的连接请求报文段突然又传送到了服务端，产生错误。
- 假设不采用三次握手，只要服务器收到链接请求并发出确认，新的连接就建立了。
- 假设现在有一个已失效的连接请求报文段延迟收到了，服务端误认为是客户端要建立新连接，于是回复了确认报文并认为成功建立了连接。实际上客户端并没有建立连接的打算，不会处理服务端的确认报文也不会发送数据。服务端此时一直等待数据，这样就浪费掉很多资源。

### 为什么需要四次挥手？

- `TCP`是全双工通信的，服务端接收到客户端的关闭请求时，可能还在向客户端发送着数据。
- 只能先回复一个收到，发送完原来的数据后再发送关闭请求。

### `HTTP` 长/短连接

- `HTTP` 请求是在`TCP`连接上进行发送的，`TCP`的连接分为：长连接，短连接
- `HTTP` 请求发送的时候，先创建一个TCP连接，然后在这个`TCP`连接上，发送`HTTP`的请求并接受返回数据。
- 一次`HTTP`请求结束了，请求端跟服务端商量是否关闭`TCP`连接。
- 短链接：
  - 关闭，下次请求的时候重新创建连接，好处是能减少服务端与客户端并发连接数
- 长链接：
  - 不关闭，`TCP` 连接一直开着会有一定的消耗，如果还有请求可以直接在这个`TCP`上发送，省去三次握手的消耗
  - 网址的并发量可能比较大，当创建连接的次数很多时，它实际的开销可能比维持一个长连接的开销还要高一些。
  - 长连接可以设置`timeout`，这个长连接一定时间内没有请求会自动关闭

### HTTP的安全性

- `http`本身并不具备加密功能，`http`报文使用明文方式发送就可能被窃听

### HTTP常见状态码

- 1xx：指示信息–表示请求已接收，继续处理。
- 2xx：成功–表示请求已被成功接收、理解、接受。
- 3xx：重定向–要完成请求必须进行更进一步的操作。
- 4xx：客户端错误–请求有语法错误或请求无法实现。
- 5xx：服务器端错误–服务器未能实现合法的请求。

### HTTP的请求过程？

- 1、建立`TCP`连接
- 2、客户端发送请求命令
- 3、客户端发送请求头
- 4、服务器应答
- 5、服务器回应头信息
- 6、服务器发送数据
- 7、断开连接

### HTTP和TCP的区别

- `TCP`协议是传输层协议,主要解决数据如何在网络中传输
- `HTTP`是应用层协议,主要解决如何包装数据。

### HTTP2 和 1 的区别

- **新的二进制格式**（`Binary Format`），`HTTP1.x`的解析是基于文本。基于文本协议的格式解析存在天然缺陷，文本的表现形式有多样性，要做到健壮性考虑的场景必然很多，二进制则不同，只认0和1的组合。基于这种考虑H`TTP2.0`的协议解析决定采用二进制格式，实现方便且健壮。
- **多路复用**（`MultiPlexing`），即连接共享，即每一个`request`都是是用作连接共享机制的。一个`request`对应一个`id`，这样一个连接上可以有多个`request`，每个连接的`request`可以随机的混杂在一起，接收方可以根据`request id`将`request`再归属到各自不同的服务端请求里面。
- **header压缩**，`HTTP1.x`的`header`带有大量信息且每次都发送，`HTTP2.0`使用`encoder`来减少需要传输的`header`大小，通讯双方各自`cache`一份`header fields`表，既避免了重复header的传输，又减小了需要传输的大小。
- **服务端推送**（`server push`），同`SPDY`一样，`HTTP2.0`也具有`server push`功能。

### HTTP和HTTPS有什么区别？

- `HTTP`协议是一种使用明文数据传输的网络协议。
- `HTTPS`协议可以理解为`HTTP`协议的升级，就是在`HTTP`的基础上加密数据后再发送到服务器。这样就算数据被截获，也能保证信息的安全。

### HTTPS的SSL加密方式？

- `https`采用对称加密和非对称加密结合的方式来进行通信。
- 对称加密： 加密和解密都是同一个钥匙
- 非对称加密：密钥成对，分为公钥和私钥，公钥加密需要私钥解密，私钥加密需要公钥解密

### HTTPS传输数据的过程？

- 使用HTTPS需要保证服务端配置好安全证书
- 客户端发送请求到服务端，服务端返回证书和公钥
- 客户端拿到证书验证安全性，然后生成一个随机数，用公钥加密随机数后发送给服务端
- 服务端用私钥解密得到随机数，再用这个随机数作为私钥对传输的数据加密，最后发送数据
- 客户端拿到传输的数据后用之前生成的随机数对其解密获取数据

### SSL建立连接的过程是什么？

- 首先客户端向服务器发送自身的`SSL`版本以及加密参数给服务器
- 服务器返回自己的`SSL`版本和参数以及数字证书包括了服务器的公钥
- 客户端生成浏览器会话秘钥通过公钥进行加密返回给服务器，服务器通过私钥解密出会话秘钥
- 客户端再发送一个报文通知服务器以后通过该会话秘钥进行加密传输，并发送加密报文表示我方SSL链接建立完成
- 服务器也回复相同的表示自己也建立连接完成

### 什么是中间人攻击

- 中间人攻击（Man-in-the-MiddleAttack，简称MITM攻击）
- 攻击者与通讯的两端分别创建独立的联系，并交换其所收到的数据，使通讯的两端认为他们正在通过一个私密的连接与对方直接对话，事实上整个会话被攻击者完全控制。
  - 客户端发送请求到服务端，请求被中间人截获。
  - 服务器向客户端发送公钥1。
  - 中间人拦截公钥1，伪造公钥2发送给客户端。
  - 客户端收到公钥2后，生成加密hash值发给服务器。
  - 中间人拦截服务器数据，使用私钥2解码获得真数据，用公钥1加密修改后的假数据返回服务端
- Charles抓包原理就是中间人攻击

### 防范中间人攻击

- 需要找一个通信双方都信任的第三方来为双方确认身份。类似公证处用公章为信件盖上章，证明这封信由原发件人发出
- CA，数字证书认证机构，就是负责颁发数字证书证明身份的
- 申请人将一些必要信息（包括公钥、姓名、电子邮件、有效期）等提供给 CA
- CA用自己的私钥对信息的散列值加密，形成数字签名，附在证书最后
- 将数字证书颁发给申请人，申请人就可以使用 CA 的证书向别人证明身份
- 相当于传送双方的信件都需要通过CA盖章，中间人攻击能伪造真实公钥但是无法
- 收件方收到数字证书之后，用 CA 的公钥解密证书最后的签名得到加密之前的散列值
- 计算数字证书中信息的散列值，将两者进行对比，只要散列值一致，就证明这张数字证书是有效且未被篡改过的

### MD5、SHA1、SHA256的区别

- 这3种本质都是摘要函数
- MD5 是 128 位
- SHA256 是 256 位
- SHA1 是 160 位 ，已经不再安全，不使用

### GET请求的过程

- 三次握手，第三次发送请求头和数据
- 服务端返回200

### POST请求过程

- 三次握手，第三次发送请求头
- 服务端返回100
- 客户端发送数据
- 服务端返回200

### GET和POST请求的区别

- `POST`比`GET`更安全，GET把请求参数拼接到URL后面，POST把请求参数放在请求体里面
- `GET`比`POST`请求速度快，因为少了一次确认过程

### DNS及解析流程

- 本地 - 根 - 顶级 - 权限 - 主机IP

### 解释一下 TTL

- `time to live`，指定数据报被路由器丢弃之前允许通过的网段数量
- `TTL` 是由发送主机设置的，转发IP数据包时，要求路由器至少将 `TTL` 减小 1。
- `TTL`的主要作用是避免包在网络中的无限循环和收发，节省了网络资源，并能使IP包的发送者能收到告警消息。

### HTTP的header都有哪些？

- `HTTP`分为请求消息和响应消息，请求方式不同，`header`也不同
- 请求消息格式：
  - 请求方法
  - URL
  - 协议版本
- 响应消息格式：
  - 状态响应码
  - 协议版本

### 请求头都有什么内容？

- 1、请求的地址域名和端口，`Host: [www.study.com](www.study.com)`
- 2、连接类型，`Connection: keep-alive`
- 3、`http` 自动升级到 `https`，`Upgrade-Insecure-Requests：1`
- 4、浏览器的用户代理信息 `User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.96 Safari/537.36`
- 5、浏览器支持的请求类型，`Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8`
- 6、浏览器能处理的压缩代码，`Accept-Encoding: gzip, deflate, sdch`
- 7、浏览器当前设置语言，`Accept-Language: zh-CN,zh;q=0.8,en;q=0.6`
- 8、请求参数长度，`Content-Length: 29`
- 9、强制要求服务器返回最新的文件内容,也就是不走缓存，返回的200，`Cache-Control: max-age=0`
- 10、请求来源地址，`Origin: <http://www.study.com>`
- 11、原始的url，`Referer: <http://www.study.com/day02/01-login.html>`

### 用户是如何通过url地址访问到服务器的，它怎么知道要访问哪个浏览器

- DNS域名解析，通过这一步还原IP（迭代查询，递归查询）
- TCP连接，三次握手
- HTTP请求，请求报文主要包括请求行，请求头，请求正文
- 处理请求返回HTTP响应，返回Response，主要包括状态码，响应头，响应报文三个部分
- 页面渲染
- 关闭连接，四次挥手

### 在浏览器中输入URL，发生的事情都有什么？

- 输入URL后，`URL = 协议 + 域名 + 服务器资源位置`
- 在通信子网中是通过`IP`为标志进行分组转发，因此需要通过`DNS`进行解析出`IP`
- 封装`HTTP`消息请求下发到传输层，在传输数据之前需要双方简历`TCP`链接
- 链接建立完成后，根据`TCP`协议进行首部封装，然后下发到网络层根据`IP`协议进行`IP`数据单元封装，
- 到数据链路层根据`ARP`协议对`IP`进行转换为`MAC`地址，然后加帧首帧尾巴，进行帧封装，
- 然后到物理层转为`bit`流进行通信传输到目的主机，自底向上进行解封装到达应用层，
- 根据资源位置，在服务器上查询到`web`对象，将`HTML`文档加到响应消息返回给客户端浏览器。

### 上述只是协议建立通信的过程，得到服务器返回的内容后做了什么？

- 客户端渲染服务器返回的内容
- 和服务端断开`TCP`连接（四次挥手）

### 验证一个字符串是否为合法的ipv4地址

- 1、校验是否有3个小数点;
- 2、以小数点将字符分割为4部分，校验每部分的字符;
- 3、校验该部分转为`int`值之后, 满足0<n<255，而且第一部分不为0

### 网络的七层协议

- 1、物理层，网卡。数据单位是bit
- 2、数据链路层，交换机，负责节点间的通信传输，保障数据传输可靠性。数据单位是帧
- 3、网络层，创建节点之间的传输逻辑链路，通过路由实现不同局域网间的通信。数据单位是数据包
- 4、传输层，建立了主机端到端服务，处理数据包错误和保证次序。`tcp、udp。ipv6`传输效率高就和这层有关
- 5、会话层，维护两个结点间的传输连接，确保点到点传输不中断，以及管理数据交换。
- 6、表示层，设备的数据格式和网络标准格式的转换
- 7、应用层，提供各种网络服务，比如文件服务器、数据库服务、电子邮件与其他网络软件服务。

### 你平时怎么解决网络请求的依赖关系：当一个接口的请求需要依赖于另一个网络请求的结果

- 方法1：在上一个网络请求的响应回调中进行下一网络请求的激活
- 方法2：线程：`NSOperation` 操作依赖和优先级，操作B依赖于操作
- 方法3：`GCD` 信号量
- 方法4：`GCD group`

## SDWebImage

### SDWebImage是怎么做缓存的？

- 缓存采用了二级缓存策略。 图片缓存的时候， 在内存和磁盘中都缓存
- 其中内存缓存是用`NSCache`实现的。

### SDWebImage缓存步骤：

- 0、下载图片
- 1、将图片缓存在内存中
- 2、判断图片的格式`png`或`jpeg`，将图片转成`NSData`数据
- 3、获取图片的存储路径， 其中图片的文件名是通过传入`URLKey`经过`MD5`加密后获得的
- 4、将图片存在进磁盘中

### SDWebImage如何获取图片

- 1、在内存缓存中找
- 2、如果内存中找不到， 会去默认磁盘目录中寻找， 如果找不到，在去自定义磁盘目录中寻找
- 3、如果磁盘也找不到就会下载图片
- 4、获取图片数据之后， 将图片数据根据图片的类型从`NSData`转化`UIImage`。
- 5、默认对图片进行解压缩，生成位图图片
- 6、将位图图片返回

### SDWebImage图片是如何被解压缩的？

- 1、判断图片是否是动态图片，如果是，不能解压缩
- 2、判断图片是否透明，如果是，不能解压缩
- 3、判断图片的颜色空间是不是RGB如果不是、不能解压缩
- 4、根据图片的大小创建一个上下文
- 5、将图片绘制在上下文中
- 6、从上下文中读取位图图像，该图像就是解压缩后的图像
- 7、将位图图像返回

### SDWebImage怎么使用NSCache

- `NSCache`是缓存专用的系统类
- `NSCache`是线程安全的
- 内存不足，`NSCache`会自动释放掉存储的对象。

```
//名称
@property (copy) NSString *name;

//NSCacheDelegate代理
@property (nullable, assign) id<NSCacheDelegate> delegate;

//通过key获取value，类似于字典中通过key取value的操作
- (nullable ObjectType)objectForKey:(KeyType)key;

//设置key、value
- (void)setObject:(ObjectType)obj forKey:(KeyType)key; // 0 cost

/*
设置key、value
cost表示obj这个value对象的占用的消耗？可以自行设置每个需要添加进缓存的对象的cost值
这个值与后面的totalCostLimit对应，如果添加进缓存的cost总值大于totalCostLimit就会自动进行删除
感觉在实际开发中直接使用setObject:forKey:方法就可以解决问题了
*/
- (void)setObject:(ObjectType)obj forKey:(KeyType)key cost:(NSUInteger)g;

//根据key删除value对象
- (void)removeObjectForKey:(KeyType)key;

//删除保存的所有的key-value
- (void)removeAllObjects;

/*
当NSCache缓存的对象的总cost值大于这个值则会自动释放一部分对象直到占用小于该值
非严格限制意味着如果保存的对象超出这个大小也不一定会被删除
这个值就是与前面setObject:forKey:cost:方法对应
*/
@property NSUInteger totalCostLimit;    // limits are imprecise/not strict

/*
缓存能够保存的key-value个数的最大数量
当保存的数量大于该值就会被自动释放
非严格限制意味着如果超出了这个数量也不一定会被删除
*/
@property NSUInteger countLimit;    // limits are imprecise/not strict
/*
这个值与NSDiscardableContent协议有关，默认为YES
当一个类实现了该协议，并且这个类的对象不再被使用时意味着可以被释放
*/
@property BOOL evictsObjectsWithDiscardedContent;

@end

//NSCacheDelegate协议
@protocol NSCacheDelegate <NSObject>
@optional
//上述协议只有这一个方法，缓存中的一个对象即将被删除时被回调
- (void)cache:(NSCache *)cache willEvictObject:(id)obj;
@end**
复制代码
复制代码
```

`countLimit**`这个属性就是设置最大缓存数量。 和栈一样遵循先进先出（`FIFO`）原则。

`countLimit`设置为5，当缓存第6个对象的时候，第一个会缓存会被移除，所以这有一个风险。

### 为什么通过key去取值的时候，一定要判断一个获取的对象是否为nil？

- 就因为很有可能某些对象被释放（顶）掉了。

### NSCache里面缓存的对象，在什么场景下会被释放？

- `NSCache`自身释放了，其中存储的对象也就释放了。
- 手动调用释放方法`removeObjectForKey、removeAllObjects`
- 缓存对象个数大于`countLimit`
- 缓存总消耗大于`totalCostLimit`
- 程序进入后台会释放
- 收到内存警告，不一定立刻释放，根据`discardContent`协议

### SDWebImage如何解决tableView的复用时出现图片错乱问题的呢

- 设置新图片前调用`UIImageView+WebCache`的`[self sd_cancelCurrentImageLoad]`

### 使用SDWebImage过程中碰到的问题？怎么解决？

- 开发项目的时候涉及到图像处理的问题，具体现象是`SDWebImage`下载超清大图出现崩溃
- 原因是库的内部下载成功后会使用 `decodeImageWithImage`这个方法解压缩并缓存图片
- 用此方法加载 `tableview/collectionview` 中多张超清大图的时候，会导致内存暴涨，很有可能crash
- 因为获取到`UImage`的时候还不是位图，在主线程赋值的时候才会绘制位图，高清会导致卡顿
- 解决办法是对于高清图片，禁止直接解压缩并缓存，自己压缩绘制出位图后再赋值。

```
 - SD4.X的解决办法
[[SDImageCache sharedImageCache] setShouldDecompressImages:NO];
[[SDWebImageDownloader sharedDownloader] setShouldDecompressImages:NO];

- SD5.0 及以上的解决办法
SDWebImageAvoidDecodeImage添加了这个枚举，意思是在子线程程解压缩图片
[self.imageView sd_setImageWithURL:self.url placeholderImage:[UIImage imageNamed:@"logo"] options:SDWebImageAvoidDecodeImage];
复制代码
复制代码
```

## AFNetworking

### AFNetworking的源码架构？

- 网络通信模块(`NSURLSession`)
- 网络状态监听模块(`Reachability`)
- 网络通信安全策略模块(`Security`)
- 网络通信信息序列化/反序列化模块(`Serialization`)

### 为什么AFN3.0中需要设置`self.operationQueue.maxConcurrentOperationCount = 1`

- 2.x是基于`NSURLConnection`的，其内部实现要在异步并发，所以不能设置1。
- 3.x是基于`NSURLSession`其内部是需要串行的鉴于一些多线程数据访问的安全性考虑， 设置这个达到串行回调的效果。

### AFNetworking 2.0 和3.0 的区别？

- AFN3.0移除了所有的`NSURLConnection`
- AFN3.x是基于`NSURLSession`的
- AFN3.0使用`NSOperationQueue`代替AFN2.0的常驻线程

### 2.x版本为什么需要常驻线程

- 在请求完成后我们需要对数据进行一些序列化处理，或者错误处理。如果我们在主线中处理这些事情很明显是不合理的。不仅会导致UI的卡顿，甚至受到默认的`RunLoopMode`的影响，我们在滑动`tableview`的时候，会导致时间的处理停止。
- 这里时候我们就需要一个子线程来处理事件和网络请求的回调了。但是，子线程在处理完事件后就会自动结束生命周期，这个时候后面的一些网络请求得回调我们就无法接收了。所以我们就需要开启子线程的`RunLoop`来保存线程的常驻。
- 当然我们可以每次发起一个请求就开启一条子线程，但是这个想一下就知道开销有多大了。所以这个时候保活一条线程来对请求得回调处理是比较好的一个方案。

### 3.x版本为什么不需要常驻线程？

- 在3.x的AFN版本中使用的是`NSURLSession`进行封装。对比于`NSURLConnection`，`NSURLSession`不需要在当前的线程等待网络回调，而是可以让开发者自己设定需要回调的队列。

- 所以在3.x版本中AFN使用了`NSOperationQueue`对网络回调的管理，并且设置`maxConcurrentOperationCount`为1，保证了最大的并发数为1，也就是说让网络请求串行执行。避免了多线程环境下的资源抢夺问题。

   

  
