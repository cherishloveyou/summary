# attribute__

1、通知编译器在目标文件中保留一个静态函数，即使它没有被引用。

2、标记为attribute__((used))的函数被标记在目标文件中，以避免链接器删除未使用的节。

3、静态变量也可以标记为used，方法是使用 __attribute__((used))。

```
static int lose_this(int);
static int keep_this(int) __attribute__((used)); // retained in object file
static int keep_this_too(int) __attribute__((used)); // retained in object file
```

1、section 函数属性使您能够将代码放置在图像的不同部分中。

2、这个函数属性是ARM编译器支持的GNU编译器扩展。

3、例程

在下面的示例中，Function_Attributes_section_0 被放置到RO section new_section 中，而不是 .text 中。

```c
void Function_Attributes_section_0 (void) __attribute__((section ("new_section")));
 
void Function_Attributes_section_0 (void) {
   static int aStatic =0;
   aStatic++;
}
```

在下面的例子中，section 函数属性覆盖了 #pragma arm section 设置。

```c
#pragma arm section code="foo"
int f2() {
  return 1;
} // into the 'foo' area

__attribute__((section ("bar"))) int f3(){
   return 1;
} // into the 'bar' area

int f4(){
  return 1;
} // into the 'foo' area

#pragma arm section
```

section关键字可以将变量定义到指定的输入段中，下面以具体的例子来讲解section的使用方法.

```c
#define SECTION(level)  __attribute__((used,__section__(".fn_cmd."level)))
#define CMD_START_EXPORT(func,func_s) 、

const struct CMD_LIST cmd_fn_##func SECTION("0.end") = {func,func_s}
CMD_START_EXPORT (start_fun,"start_fun");
```

首先来看SECTION这个宏定义，这个宏可以将变量添加到某个输入段中。例如

int a __attribute__((section(“list”))) = 0;
这句话的意思是将一个int型的变量a放到名为list的输入段中。

level可以理解为这个输入段的后缀，所以这个输入段最终的名字取决于level，如果这里level = "0.end",那么将这个宏展开得到

 __attribute__((used,__section__(".fn_cmd.""0.end")))
这个时候查看.map文件就可以发现文件里面多了一个名为.fn_cmd.0.end的输入段，可见level确实就是这个输入段的后半部分。

好了，讲解完SECTION这个宏我们对CMD_START_EXPORT（start_fun,"start_fun"）;这个宏进行初步展开可以得到下面的表达式，可以看到就是定义了一个struct CMD_LIST类型的变量并且使用了section对其属性进行了修饰。

const struct CMD_LIST cmd_fn_start_fun SECTION("0.end") = {start_fun,"start_fun"};
这其中有一个地方要注意一下，就是宏表达式中的##符号，这个符号的作用就是将两个字符串进行拼接，例如

\#define DEF_INT(a,b) int a##b = 0
DEF_INT(a,b);
那么最终这个宏实现的意思就是定义一个int型的变量，变量的名字叫ab，将这个宏展开就是

int ab = 0；
好了知道了##的意思那么就明白了为什么宏展开的结果是定义了一个名为cmd_fn_start_fun的变量了。然后现在在进一步进行宏展开，将其中的SECTION展开得到如下表达式。

const struct CMD_LIST cmd_fn_start_fun __attribute__((used,__section__(".fn_cmd.""0.end"))) = {start_fun,"start_fun"};
这个时候再来看会发现其实就是定义了一个struct CMD_LIST 类型的变量，变量的名字是cmd_fn_start_fun，并且这个变量被放到了我们所希望的一个输入段.fn_cmd.0.end中了。



那么问题来了，使用section将变量放到我们自定义的输入段中有什么意义呢？
我们知道在传统的C语言编程中程序结构是这样的。

```c
int main() {
  init_xx();
  init_xx();
  ...
  while(1) {
    ...
  }
}
```

先进行若干个初始化程序，然后在循环的执行一段代码。这样开发固然可以，但是这样有一个让人非常不爽的地方，就是每写一个初始化函数都要在main函数中调用，非常的不方便。但是如果使用section先事先将所有的初始化函数加入到我们自己定义的输入段中，然后再在main函数中将这个输入段中初始化函数依次取出，这样就可在不修改main函数的前提下完成对系统的初始化了。

那么section是怎么将这些初始化函数放入输入段中，并且系统还可以获取这些初始化函数的地址呢？在说明之前我先将下面会用到的几个宏定义进行说明一下。

```c++
#define SECTION(level)  __attribute__((used,__section__(".fn_cmd."level)))
\#define CMD_START_EXPORT(func,func_s) const struct CMD_LIST cmd_fn_##func SECTION("0.end") = {func,func_s}
\#define CMD_EXPORT(func,func_s) const struct CMD_LIST cmd_fn_##func SECTION("1") = {func,func_s}
\#define CMD_END_EXPORT(func,func_s) const struct CMD_LIST cmd_fn_##func SECTION("1.end") = {func,func_s}
```


当这几个宏被调用时将会产生名为cmd_fn_xx的变量，并且这个变量根据被调用的宏来把这个变量放到相应的输入段。例如CMD_START_EXPORT这个宏，这个宏其实上面已经讲过了，调用这个宏的时候会产生一个名为cmd_fn_xx的变量，并且把这个变量放到了我们自定义的输入段.fn_cmd.0.end中了。其他两个宏的其实也是差不多的，不同之处就是输入段有些区别。

言归正传，现在继续来讲如何使用section将不同的函数放到我们想要的输入段中，并且获得他们的起始地址和结束地址。

我们可以在每个XXX_Init函数后面都调用宏CMD_EXPORT，在调用这个宏时编译器会将XXX_init这个函数加入到输入段中，由于变量在输入段中的地址是连续的，并且顺序先按 section 名 01234排一遍，section 内再按函数名称排。所以可以按照输入段中顺序来逐个调用这些初始化函数来完成系统的初始化。

具体实现我会根据我的一个自定义命令行的应用来进行部分的说明。

```c++
/*命令函数段起始位置*/
int cmd_start(void) {
   return 0;
}
CMD_START_EXPORT(cmd_start,"int cmd_start(void)");

/*命令函数段结束位置*/
int cmd_end(void) {
   return 0;
}
CMD_END_EXPORT(cmd_end,"int cmd_end(void)");

void test(void) {
   printf("hello world\r\n");
}
CMD_EXPORT(test,"void test(void)");

void demo(void) {
    printf("hello world\r\n");
}
CMD_EXPORT(demo,"void demo(void)");
```

先定义start和end函数并且分别使用CMD_START_EXPORT和CMD_END_EXPORT来将其放到输入段.fn_cmd.0.end和.fn_cmd.1.end中，按照上面的说明输入段.fn_cmd.0.end是排在输入段.fn_cmd.1.end前面的，而使用的CMD_EXPORT这个宏对应的输入段.fn_cmd.1是排在.fn_cmd.0.end和.fn_cmd.1.end之间的，这里可以看一下编译产生的.map会更加的形象一些。具体在MAP文件的位置如下所示

 cmd_fn_cmd_start             0x080042f0  Data      8  serialcmd.o(.fn_cmd.0.end)
 cmd_fn_test                        0x080042f8  Data      8  application.o(.fn_cmd.1)
 cmd_fn_demo                    0x08004300  Data      8  application.o(.fn_cmd.1)
 cmd_fn_cmd_end              0x08004308  Data      8  serialcmd.o(.fn_cmd.1.end)
可以看到输入段.fn_cmd.0.end的地址确实是在最靠前的，而.fn_cmd.1.end的地址确实是排在最后面的。在输入段.fn_cmd.1中的那些数据就是我们要用到的数据。 由于我所放到输入段中的变量是struct CMD_LIST类型的，占8个字节，定义如下

typedef void (*fun)();
struct CMD_LIST {
    fun funs;
    const INT8 *cmd;
};
所以每个变量的大小是8，也就是每个变量在内存中的偏移是8，所以就明白了为什么变量的地址是每次递增8个字节了。

下面这个函数就是从输入段中获取数据，并进行相应处理的函数。

```c++
const static struct CMD_LIST *CmdList;
static UINT8 CmdSize = 0;

/*命令函数初始化*/
void SerialCmdInit(void) {
   const struct CMD_LIST *cmd_ptr;
   CmdList = &cmd_fn_cmd_start;
   CmdList++;
   for (cmd_ptr = CmdList; cmd_ptr < &cmd_fn_cmd_end;cmd_ptr++) {
    /*这里如果用于初始化的话可以使用下面这种方式来执行初始化函数，因为我的应用并不是用于初始       
     化，所以就没有进行函数调用。
     （*cmd_ptr->fun）();
     */
     CmdSize++;
   }
}
```

其中cmd_fn_cmd_start是在.fn_cmd.0.end这个输入段中的，而各个要执行的函数是在.fn_cmd.1这个输入段中的，cmd_fn_cmd_end作为结束的标志在.fn_cmd.1.end输入段中。其余的就不做过多讲解了，从上面的.map文件中的地址很容易就可以看出在得到了起始地址和结束地址之后怎样一个个遍历这些函数。

最后再说一下之前__attribute__((used,__section__(".fn_cmd."level)))中used的意思。
unused：表示该函数或变量可能不使用，这个属性可以避免编译器产生警告信息。
used： 向编译器说明这段代码有用，即使在没有用到的情况下编译器也不会警告。



[**iOS冷启动优化之模块启动项自注册实现**](https://www.jianshu.com/p/0e46b744bee0?utm_campaign=hugo)
