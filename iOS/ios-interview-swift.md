## swift问题



#### 1.什么是函数式编程？

```undefined
函数式编程其实是一种编程思想, 代码写出来只是它的表现形式

在面向对象的编程思想中, 我们将要解决的一个个问题, 抽象成一个个类, 通过给类定义属性和方法, 让类帮助我们解决需要处理的问题.(其实面向对象也叫命令式编程, 就像给对象下一个个命令)

而在函数式编程中, 我们则通过函数描述我们要解决的问题, 以及解决问题需要怎样的方案.

RxSwift
```

#### 2.swift相对于OC有哪些优点？

```ruby
1、swift语法简单易读、代码更少，更加清晰、易于维护
2、更加安全，optional的使用更加考验程序员对代码安全的掌控
3、泛型、结构体、枚举都很强大
4、函数为一等公民，便捷的函数式编程
5、有命名空间 基于module
6、类型判断

oc的优点、运行时
```

#### 3.什么是泛型,swift在哪些地方使用了泛型？

```swift
泛型（generic）可以使我们在程序代码中定义一些可变的部分，在运行的时候指定。使用泛型可以最大限度地重用代码、保护类型的安全以及提高性能。

例如 optional 中的 map、flatMap 、?? (泛型加逃逸闭包的方式，做三目运算)
```

#### 4.defer、guard的作用？

```swift
defer 包体中的内容一定会在离开作用域的时候执行,多个 defer, 那么后加入的先执行

guard 过滤器，拦截器,guard 总是有一个 else 语句, 如果表达式是假或者值绑定失败的时候, 会执行 else 语句, 且在 else 语句中一定要停止函数调用
```

#### 5.swift语法糖 ？！的本质（实现原理）

```swift
？ 为optional的语法糖

Optional< T > 是一个包含了 nil 和 普通类型的枚举，确保使用者在变量为nil的情况下的处理

!  为optional 强制解包 的语法糖
```

#### 6.举例swift中模式匹配的作用？

```csharp
模式匹配： 在switch中体现最明显

通配符模式： _

标识符模式：let i = 1

值绑定模式：case .Student(let name) 或者 case let .Student(name)

元祖模式：case (let code, _)

可选模式：if case let x？ = someOptional { }

类型转换模式：case is Int:  或者  case let n as String:

表达式模式：范围匹配 case (0..<2)  case(0...2, 2...4)

条件句中使用where： case (let age) where age > 30

if case let：if case let .Student(name) = xiaoming { }

for case let： for case let x in array where x > 10 {}  或者 for x in array where x > 10
```

#### 7.swift中closure与OC中block的区别？

```undefined
1、closure是匿名函数、block是一个结构体对象

2、都能捕获变量

3、closure通过逃逸闭包来在block内部修改变量，block 通过 __block 修饰符
```

#### 8.什么是capture list，举例说明用处？

```swift
捕获列表

weak  unowned
```

#### 9.swift中private与fileprivate的区别？

```swift
private的作用域被约束与被定义的当前类作用域，fileprivate作用域是整个文件
```

#### 10.Set 独有的方法有哪些？

```rust
intersect（_:）// 根据两个集合中都包含的值创建的一个新的集合
exclusiveOr(_:) // 根据只在一个集合中但不在两个集合中的值创建一个新的集合
union(_:) // 根据两个集合的值创建一个新的集合
subtract(_:) //根据不在该集合中的值创建一个新的集合

isSubsetOf(_:) //判断一个集合中的值是否也被包含在另外一个集合中
isSupersetOf(_:) //判断一个集合中包含的值是否含有另一个集合中所有的值
isStrictSubsetOf(:) isStrictSupersetOf(:) //判断一个集合是否是另外一个集合的子集合或者父集合并且和特定集合不相等
isDisjointWith(_:) //判断两个集合是否不含有相同的值
```

#### 11.实现一个 min 函数，返回两个元素较小的元素

```swift
func minNum<T: Comparable>(a: T, b: T) -> T {
    return a < b ? a : b
}
```

#### 12.map、filter、reduce 的作用

```swift
1、map 是Array类的一个方法，我们可以使用它来对数组的每个元素进行转换
let intArray = [1, 3, 5]
let stringArr = intArray.map {
    return "\($0)"
}

// ["1", "3", "5"]
2、filter 用于选择数组元素中满足某种条件的元素
let filterArr = intArray.filter {
    return $0 > 1
}
//[3, 5]
3、reduce 把数组元素组合计算为一个值
let result = intArray.reduce(0) {
    return $0 + $1
}
//9
```

#### 13.map 与 flatmap 的区别

```swift
1、map 可以对一个集合类型的所有元素做一个映射操作

2、和map 不同，flatmap 在之前版本有两个定义，分别是：
func flatMap(transform: (Self.Generator.Element) throws -> T?) -> [T]
func flatMap(transform: (Self.Generator.Element) -> S) -> [S.Generator.Element]

swift 4.1 废弃后改为
func flatMap(transform: (Self.Generator.Element) throws -> Sequence) -> [Sequence.Element]
func flatMap<U>(_ transform: (Wrapped) throws -> U?) rethrows -> U?

1) flatMap的第一个作用和map一样，对一个集合类型的所有元素做一个映射操作,但是可以过滤为nil的情况
例如：
let array = [1,2,5,6,7,nil]
let array_map = array.map { $0 }
//[Optional(1), Optional(2), Optional(5), Optional(6), Optional(7), nil]
let array_flatmap = array_map.flatMap { $0 }
//[1, 2, 5, 6, 7]

2) 第二种情况可以进行“降维”操作
let array = [["1", "2"],["3", "4"]]
let array_map = array.map { $0 }
//[["1", "2"], ["3", "4"]]
let array_flatmap = array_map.flatMap { $0 }
//["1", "2", "3", "4"]
```

#### 14.什么是 copy on write

```swift
copy on write, 写时复制，简称COW，它通过浅拷贝(shallow copy)只复制引用而避免复制值；当的确需要进行写入操作时，首先进行值拷贝，在对拷贝后的值执行写入操作，这样减少了无谓的复制耗时。
Copy-on-Write 是一种用来优化占用内存大的值类型的拷贝操作的机制。
对于Int，Double，String 等基本类型的值类型，它们在赋值的时候就会发生拷贝。
对于Array、Dictionary、Set 类型，当它们赋值的时候不会发生拷贝，只有在修改的之后才会发生拷贝。
对于自定义的数据类型不会自动实现COW，可按需实现。
```

#### 15.如何获取当前代码的函数名和行号

```cpp
#function
#line
#file
#column
```

#### 16.如何声明一个只能被类 conform 的 protocol

```kotlin
protocol SomeClassProtocl: class {
    func someFunction()
}
```

#### 17.String 与 NSString 的关系与区别

```dart
两者可以随意转换

String为值类型，拷贝赋值需要值拷贝

NSString 引用类型传递指针
```

#### 20.如何截取 String 的某段字符串

```rust
substring 已废弃

let star = str.index(str.startIndex, offsetBy: 0)

let end = str.index(str.startIndex, offsetBy: 4)

let substr = str[star..<end]
```

#### 21.throws 和 rethrows 的用法与作用

```swift
当闭包参数会抛出异常时 使用throws

同时外部方法体返回结果需要 rethrows 异常

rethrows 可以用 throws 替换, 反过来不行
```

#### 22.try？ 和 try！是什么意思

```go
不处理错误，抛出异常函数时, 如果函数抛出异常, 则返回 nil, 否则返回函数返回值的可选值,

保证不会出现错误 强制解，抛出异常的时候崩溃, 否则则返会函数返回值
```

#### 23.associatedtype 的作用

```swift
关联类型，关联类型为协议中的某个类型提供了一个别名，其代表的真实类型在实现者中定义

//协议，使用关联类型
protocol TableViewCell {
    associatedtype T
    func updateCell(_ data: T)
}
 
//遵守TableViewCell
class MyTableViewCell: UITableViewCell, TableViewCell {
    typealias T = Model
    func updateCell(_ data: Model) {
        // do something ...
    }
}
```

#### 24.public 和 open 的区别

```undefined
是否可以继承,都用于在模块中声明需要对外界暴露的函数, 区别在于, public修饰的类, 在模块外无法继承, 而open则可以任意继承, 公开度来说, public < open
```

#### 25.声明一个只有一个参数没有返回值闭包的别名

```kotlin
typealias MyBlock = (Int) -> (Void)
```

#### 26.Self 的使用场景

```php
Self 通常在协议中使用, 用来表示实现者或者实现者的子类类型
例如：协议定义的时候，如果需要使用到实现者的上下文怎么办？ 我们并不知道谁会实现自己,这个时候可以使用Self进行指代

定义一个复制的协议
protocol CopyProtocol {
    func copy() -> Self
}
```

#### 27.dynamic 的作用

```csharp
swift中的函数是静态调用，静态调用的方式会更快，但是静态调用的时候没救不能从字符串查找到对于的方法地址，这样 与OC交互的时候，OC动态查找方法就会找不到，这个时候就可以通过使用 dynamic 标记来告诉编译器，这个方法要被动态调用的

swift中如果KVO监听属性，那么属性就需要 dynamic 来标记
```

#### 28.什么时候使用 @objc

```undefined
与OC 的交互部分

KOV 监听、动态方法查找等都需要

协议可选方法等
```

#### 29.Optional（可选型） 是用什么实现的

```go
枚举 一个 为nil，一个为属性值
enum Optional<Wrapped> {
  case none
  case some(Wrapped)
}
```

#### 30.如何自定义下标获取

```swift
extension Demo {
   subscript(index: Int) -> Int {
      get {
        // 返回一个适当的 Int 类型的值
      }
      set(newValue) {
        // 执行适当的赋值操作
      }
    }
}
extension AnyList {
    subscript(index: Int) -> T{
        return self.list[index]
    }
    subscript(indexString: String) -> T?{
        guard let index = Int(indexString) else {
            return nil
        }
        return self.list[index]
    }
}
```

#### 31.inout 的作用

```undefined
让输入参数可变 类似__block 的作用
```

#### 32.Error 如果要兼容 NSError 需要做什么操作

```tsx
Error是一个协议, swift中的Error 都是enum， 可以转 NSError

如果需要Error有NSError的功能，实现 LocalizedError CustomNSError 协议
```

#### 33.下面的代码都用了哪些语法糖

```bash
[1, 2, 3].map{ $0 * 2 }
array语法糖
尾部闭包语法糖
$0
```

#### 34.什么是高阶函数

```swift
一个函数如果可以以某一个函数作为参数, 或者是返回值, 那么这个函数就称之为高阶函数.map、flatMap、filter、reduce？
```

#### 35.下面的代码会不会崩溃，说出原因

```csharp
var mutableArray = [1,2,3]
for _ in mutableArray {
    mutableArray.removeLast()
}

不会，值类型,估计是在一开始, for in 就对 mutableArray 进行了一次值捕获, 而 Array 是一个值类型, removeLast() 并不能修改捕获的值.
```

#### 36.给集合中元素是字符串的类型增加一个扩展方法，应该怎么声明

```dart
extension Array where Element == String { 
   var isStringElement:Bool {
       return true
   }
}
```

#### 37.定义静态方法时关键字 static 和 class 有什么区别

```objectivec
非class类型一般统一用 static 例如 枚举 结构体

protocol中 使用 static, 实现协议的 枚举 结构体 用 static

class 中使用 class static 都可以
  
static 定义的方法不可以被子类继承, class 则可以
  
class AnotherClass {
    static func staticMethod(){}
    class func classMethod(){}
}
class ChildOfAnotherClass: AnotherClass {
    override class func classMethod(){}
    //override static func staticMethod(){}// error
}
```

#### 38.一个 Sequence 的索引是不是一定从 0 开始？

```undefined
不一定, 两个 for in 并不能保证都是从 0 开始,
```

#### 39.数组都实现了哪些协议

```undefined
Decodable Encodable Equatable Hashable  CustomStringConvertible, CustomDebugStringConvertible RandomAccessCollection, MutableCollection RangeReplaceableCollection CustomReflectable ExpressibleByArrayLiteral
```

#### 40.如何自定义模式匹配

```swift
infix operator =~

func =~ (str: String, pattern: String) -> Bool {
    
}

infix、  prefix、  postfix  用于自定义表达式的声明， 分别表示 中缀、前缀、后缀  
```

#### 41.autoclosure 的作用

```undefined
自动闭包，将参数自动封装为闭包参数
```

#### 42.下面代码中 mutating 的作用是什么

```swift
struct Person {
    var name: String {
        mutating get {
            return store
        }
    }
}

结构体中的 属性可能发生改变
```

#### 43.如何让自定义对象支持字面量初始化

```undefined
ExpressibleByArrayLiteral 可以由数组形式初始化
ExpressibleByDictionaryLiteral 可以由字典形式初始化
ExpressibleByNilLiteral 可以由nil 值初始化
ExpressibleByIntegerLiteral 可以由整数值初始化
ExpressibleByFloatLiteral 可以由浮点数初始化
ExpressibleByBooleanLiteral 可以由布尔值初始化
ExpressibleByUnicodeScalarLiteral
ExpressibleByExtendedGraphemeClusterLiteral
ExpressibleByStringLiteral
这三种都是由字符串初始化, 上面两种包含有 Unicode 字符和特殊字符

```

#### 44.为什么数组索引越界会崩溃，而字典用下标取值时 key 没有对应值的话返回的是 nil 不会崩溃。

```swift
struct Array<Element> {
    subscript(index: Int) -> Element
}

struct Dictionary<Key: Hashable, Value> {
    subscript(key: Key) -> Value?
}
```

#### 45.一个函数的参数类型只要是数字（Int、Float）都可以，要怎么表示。

```swift
Int、Float 都有一个协议

func myMethod<T>(_ value: T) where T: Numeric {
    print(value + 1)
} 

或者 ExpressibleByIntegerLiteral 协议也行
```

#### 46.Swift的静态派发

```undefined
很显然静态派发是一种更高效的方法，因为静态派发免去了查表操作。

不过静态派发是有条件的，方法内部的代码必须对编译器透明，并且在运行时不能被更改，这样编译器才能帮助我们。

Swift中的值类型不能被继承，也就是说值类型的方法实现不能被修改或者被复写，因此值类型的方法满足静态派发的要求。

默认静态派发，如果需要满足动态派发，需要 dymanic修饰
```

#### 47.Swift有哪些修饰符

```swift
open、public、internal、fileprivate、private
```

#### 48、实现一个函数，输入是任一整数，输出要返回输入的整数 + 2

```kotlin
func plusTwo(one: Int) -> (Int) -> Int {
    return { (two: Int) in return two + one }
}

plusTwo(one: 4)(2)
```

#### 49、Swift 到底是面向对象还是函数式的编程语言？

```swift
Swift 既是面向对象的，又是函数式的编程语言。
说 Swift 是 Object-oriented，是因为 Swift 支持类的封装、继承、和多态，从这点上来看与 Java 这类纯面向对象的语言几乎毫无差别。
说 Swift 是函数式编程语言，是因为 Swift 支持 map, reduce, filter, flatmap 这类去除中间状态、数学函数式的方法，更加强调运算结果而不是中间过程。
```

#### 50、class 和 struct 的区别

```kotlin
class 引用类型，可以继承、多态，通过引用计数来管理

struct 是值类型，不通过引用计数来管理
```

#### 51、Any 和 AnyObject 有什么区别

```kotlin
AnyObject：可以代表任何class类型的实例；
Any：可以代表任何类型，甚至包括方法(func)类型
AnyObject是Any的子集
所有用class关键字定义的对象就是AnyObject
所有不是用class关键字定义的对象就不是AnyObject，而是Any
```

#### 52、逃逸闭包和非逃逸闭包

```kotlin
非逃逸闭包和逃逸闭包一般当做参数传递给函数
逃逸闭包也不能捕获inout
非逃逸闭包：闭包调用发生在函数结束前，闭包调用在函数作用域内
逃逸闭包：闭包调用有可能在函数结束后调用，闭包调用逃离了函数的作用域，需要@escaping声明
```

#### 53、闭包的使用场景

常见使用场景：

1.作为变量

2.作为可选的变量

3.作为类型别名

4.作为常量

5.定义函数时作为函数的参数

6.调用函数时作为函数的参数：完整的闭包格式

7.调用函数时作为函数的参数：根据上下文推断类型

8.调用函数时作为函数的参数：单行表达式闭包隐式返回

9.调用函数时作为函数的参数：参数名称缩写

10.调用函数时作为函数的参数：尾随闭包（不是函数的唯一一个参数时）

11.调用函数时作为函数的参数：尾随闭包（是函数的唯一一个参数时）

12.调用函数时传入一个闭包函数作为函数的参数

13.调用函数时作为函数的参数：循环强引用

```swift
// 1.作为变量（类中的属性）:
// var 闭包名称: (参数列表) -> 返回类型
var closureName1: (_ name: String, _ age: Int) -> String

// 2.作为可选的变量（类中的属性）:
// var closureName: ((parameterTypes) -> returnType)?
var closureName2: ((_ name: String, _ age: Int) -> String)? 

// 3.作为类型别名(闭包类型):
// typealias closureType = (parameterTypes) -> returnType
typealias closureType = (_ name: String, _ age: Int) -> String

// 4.作为常量（类中的属性）:
// let closureName: closureType = 闭包表达式
let closureName3: closureType = { (_ name: String, _ age: Int) -> String in
    return "My name is \(name), age is \(age)"
}
closureName3("Abnerzj", 10)

// 5.定义函数时作为函数的参数:
// 5.1 func 函数名(参数名: 闭包类型)
func closureFuncName(closureParameterName: closureType) -> Void {
    // 函数体部分
}

// 5.2 func 函数名(参数名: 闭包表达式)
func closureFuncName2(closureParameterName: (_ name: String, _ age: Int) -> String) -> Void {
    // 函数体部分
}

// 6.调用函数时作为函数的参数：完整的闭包格式
// 函数名(参数名: 闭包表达式)
// 函数名(参数名: { 闭包参数列表 -> 闭包返回值 in 闭包函数体 })
closureFuncName(closureParameterName: {
    (_ name: String, _ age: Int) -> String in
    return "My name is" + name + ", age is \(age)"
})

// 7.调用函数时作为函数的参数：根据上下文推断类型:
// 函数名(参数名: { 实参名1, 实参名2 in 闭包函数体 })
closureFuncName(closureParameterName: {
    name, age in
    return "My name is" + name + ", age is \(age)"
})

// 8.调用函数时作为函数的参数：单行表达式闭包隐式返回，可以隐藏return关键字
// 函数名(参数名: { 实参名1, 实参名2 in 闭包函数体 })
closureFuncName(closureParameterName: {
    name, age in "My name is" + name + ", age is \(age)"
})

// 9.调用函数时作为函数的参数：参数名称缩写（$0,$1,$2...来顺序代替参数列表中的参数名）
// 函数名(参数名: { 闭包函数体 })
closureFuncName(closureParameterName: {
    "My name is \($0), age is \($1)"
})

// 10.调用函数时作为函数的参数：尾随闭包（作为函数的最后一个参数），不是函数的唯一一个参数时
// 函数名() { 闭包函数体 }
closureFuncName() {
    "My name is \($0), age is \($1)"
}

// 11.调用函数时作为函数的参数：尾随闭包（作为函数的最后一个参数），是函数的唯一一个参数时
// 函数名 { 闭包函数体 }
closureFuncName {
    "My name is \($0), age is \($1)"
}

// 12.调用函数时传入一个闭包函数作为函数的参数
// 函数名(参数名: 闭包函数名)
func closureFunc(_ name: String, _ age: Int) -> String {
    return name + "\(age)"
}
closureFuncName(closureParameterName: closureFunc)

// 13.调用函数时作为函数的参数：循环强引用
// 函数名(参数名: { [弱引用或无主引用列表] 闭包参数列表 -> 闭包返回值 in 闭包函数体 })
// 第一种：推荐
closureFuncName(closureParameterName: {
    [weak self] (_ name: String, _ age: Int) -> String in
    self?.view.backgroundColor = UIColor.red
    return "My name is" + name + ", age is \(age)"
})

// 第二种：
weak var weakself = self
closureFuncName(closureParameterName: {
    (_ name: String, _ age: Int) -> String in
    weakself?.view.backgroundColor = UIColor.red
    return "My name is" + name + ", age is \(age)"
})
// 第三种：
closureFuncName(closureParameterName: {
    [unower self] (_ name: String, _ age: Int) -> String in
    self?.view.backgroundColor = UIColor.red
    return "My name is" + name + ", age is \(age)"
})
```

#### 54、自动闭包

```kotlin
自动闭包是一种自动创建的用来把作为实际参数传递给函数的表达式打包的闭包。它不接受任何实际参数，并且当它被调用时，它会返回内部打包的表达式的值。这个语法的好处在于通过写普通表达式代替显式闭包而使你省略包围函数形式参数的括号。
func getFirstPositive(_ v1: Int, _ v2: @autoclosure () -> Int) -> Int? {
    return v1 > 0 ? v1 : v2()
}
getFirstPositive(10, 20)

为了避免与期望冲突，使用了@autoclosure的地方最好明确注释清楚:这个值会被推迟执行
@autoclosure 会自动将 20 封装成闭包 { 20 }
@autoclosure 只支持 () -> T 格式的参数
@autoclosure 并非只支持最后1个参数
有@autoclosure、无@autoclosure，构成了函数重载

如果你想要自动闭包允许逃逸，就同时使用 @autoclosure 和 @escaping 标志。

```

#### 54、存储属性和计算属性的区别

```
存储属性(Stored Property)

类似于成员变量这个概念
存储在实例对象的内存中
结构体、类可以定义存储属性
枚举不可以定义存储属性

计算属性(Computed Property)

本质就是方法(函数)
不占用实例对象的内存
枚举、结构体、类都可以定义计算属性
```

#### 55、延迟存储属性(Lazy Stored Property)

```
使用lazy可以定义一个延迟存储属性，在第一次用到属性的时候才会进行初始化(类似OC中的懒加载)

lazy属性必须是var，不能是let
let必须在实例对象的初始化方法完成之前就拥有值

如果多条线程同时第一次访问lazy属性，无法保证属性只被初始化1次

class PhotoView {
    // 延迟存储属性
    lazy var image: Image = {
        let url = "https://...x.png"        
        let data = Data(url: url)
        return Image(data: data)
    }() 
} 
```

#### 55、属性观察器

```swift
可以为非lazy的var存储属性设置属性观察器,通过关键字willSet和didSet来监听属性变化

struct Circle {
    var radius: Double {
        willSet {
            print("willSet", newValue)
        } 
        didSet {
            print("didSet", oldValue, radius)
        }
    } 
    init() {
        self.radius = 1.0
        print("Circle init!")
    }
}
```
