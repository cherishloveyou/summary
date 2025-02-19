# **Mixin**

### 一，概述

- **继承**（关键字 extends）
- **混入** mixins （关键字 with）
- **接口实现**（关键字 implements）

这三种关系可以同时存在，但是有前后顺序：

```
extends -> mixins -> implements
```

**extens**在前，**mixins**在中间，**implements**最后

### 二，继承(extends)

- extends是继承可以继承父类的特性。

  - Dart中继承是单继承.
  - 构造函数或者析构函数不能继承.
  - 子类重写父类方法要在方法前加@override.
  - 子类调用父类的方法用super.
  - Dart中子类可以访问父类的所有变量和方法.

  

  ```
  class Person {
    //公有变量
    String name; 
    num age; 
    //私有变量
    String _gender;
    //类名构造函数
    Person(this.name,this.age);
    //公有的方法
    void printInfo() {
      print("${this.name}---${this.age}");  
    }
  
    work(){
      print("${this.name}在工作...");
    }
  }
  
  class Web extends Person{
  　Web(String name, num age) : super(name, age);
    run(){
      print('run');
      super.work();  //自类调用父类的方法
    }
  
    //覆写父类的方法
    @override       //可以写也可以不写  建议在覆写父类方法的时候加上 @override 
    void printInfo(){
       print("姓名：${this.name}---年龄：${this.age}"); 
    }
  }
  
  main(){ 
    Web w=new Web('李四',20);
    // w.printInfo();
    w.run();
  }
  ```

### 二, 混合 mixins (with)

　　**mixins**的中文意思是**混入**，就是在类中**混入**其他功能。在Dart中可以使用**mixins**实现类似多继承的功能因为**mixins**使用的条件，随着Dart版本一直在变，这里说的是Dart2.x中使用**mixins**的条件：

- (1) 作为**mixins**的类只能继承自Object，不能继承其他类

- (2) 作为**mixins**的类不能有构造函数

- (3) 一个类可以**mixins**多个**mixins**类

- (4) mixins绝不是继承，也不是接口，而是一种全新的特性

  

  ```
  class Person{
    String name;
    num age;
    Person(this.name,this.age);
    printInfo(){
      print('${this.name}----${this.age}');
    }
    void run(){
      print("Person Run");
    }
  }
  
  class A {
    String info="this is A";
    void printA(){
      print("A");
    }
    void run(){
      print("A Run");
    }
  }
  
  class B {  
    void printB(){
      print("B");
    }
    void run(){
      print("B Run");
    }
  }
  
  class C extends Person with B,A{
    C(String name, num age) : super(name, age);
  }
  
  void main(){  
    var c=new C('张三',20);  
    c.printInfo();
    // c.printB();
    // print(c.info);
    c.run();
  }
  ```
  
  

### 3.接口实现(implements)

　　Flutter是没有interface的，但是Flutter中的每个类都是一个隐式的接口，这个接口包含类里的所有成员变量，以及定义的方法。如果有一个类 A,你想让类B拥有A的API，但又不想拥有A里的实现，那么你就应该把A当做接口，类B implements 类A.
　　所以在Flutter中:class 就是 interface

- 当class被当做interface用时，class中的方法就是接口的方法，需要在子类里重新实现，在子类实现的时候要加@override

- 当class被当做interface用时，class中的成员变量也需要在子类里重新实现。在成员变量前加@override

  ```
  /*
  Dart中一个类实现多个接口：
  */
  
  abstract class A{
    String name;
    printA();
  }
  
  abstract class B{
    printB();
  }
  
  class C implements A,B{  
    @override
    String name;  
    @override
    printA() {
      print('printA');
    }
    @override
    printB() {
      // TODO: implement printB
      return null;
    }
  }
  
  void main(){
    C c=new C();
    c.printA();
  }
  ```



class 定义一个类，常规的类，定义一些方法、属性。可以被继承
 mixin 定义一个类，定义一些方法、属性。 不可以被实例化。可以被混入
 abstract class 定义一个抽象类，可以包含抽象方法和具体方法。可以被继承
 abstract mixin class 当一个类既需要被继承（作为基类）又需要作为 mixin 提供功能时。 可以被继承和混入\



dart没有多继承，一个类只能extends另外一个类。通过with，可以混入多个mixin类。

1、mixin不能被实例化。
 2、mixin不允许存储状态。mixin的如果有属性只能是getter和setter。或者是final属性，在声明的时候就初始化了，不能再修改。这种不允许存储状态的设计，目的也是为了避免对混入类产生副作用。
 3、mixin的属性和方法，也不是通过super调用的，直接通过this调用。
 4、mixin可以限制哪些类可以混入它，on后面指定类或它的子类才可以使用。



“mixin 不存储状态” 的真正含义是什么？

“mixin 不存储状态”的说法主要强调：

- 不直接管理类的核心状态：

mixin 不应强加类的主要状态管理逻辑（例如复杂的初始化、生命周期管理）。
 它只应提供功能扩展，而不会对类的全局状态产生较大副作用。

- 避免破坏类的初始化逻辑：

由于 mixin 不能定义构造函数，因此不会与混入类的初始化逻辑冲突。



| 类型            | 解决什么问题                     | 使用场景                         | 限制                           |
| --------------- | -------------------------------- | -------------------------------- | ------------------------------ |
| extends         | 子类继承                         | 子类继承父类                     | 可以有构造方法和实例变量       |
| Mixin（with）   | 实现类似多继承，能力集           | 不通过继承，获取一个类的能力     | 不能有构造方法，可以有实例变量 |
| Extension（on） | 给系统类【例如String类添加功能】 | 在无法修改被扩展类源码的情况下   | 不能有构造方法和实例变量       |
| Implement       | 声明和实现的解耦                 | 模版方法的实现【设计模式的一种】 | 暂无                           |



**抽象类 和 继承**

在面向对象的语言中任何一个抽象类都是不可以直接实例化的，只能用来继承。在 Dart 中可以使用 ‘***abstract***’ 关键字定义一个抽象类，并且抽象类中的方法可以省略方法体：

当你使用非抽象类类继承一个抽象类时，必须使用 ***@override*** 覆写抽象类中的方法（不必覆写所有）并保证方法名称一致。

**继承 ---- extends** 

- 典型的面向对象的继承，用于扩展父类；
- **class B extends A { }** , 不强制覆写每一个父类中的方法(getter,setter 必须覆写)；
- 在 Dart 只能继承一个类

**接口 ---- interface**

- 当你不想提供方法的实现而只想提供它们的 API 时，使用接口；
- **class C implements B { }**，你必须覆写B中所有的方法；
- implements 可以扩展到多个类

**混入 ---- mixin**

- 共享相同逻辑的代码段；
- **class B with A { }**, 在B中你可以使用A的所有方法，可以通过 on 关键字限制 mixin 的使用范围；
- with 可以扩展到多个 mixin
