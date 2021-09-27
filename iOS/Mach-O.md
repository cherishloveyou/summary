## Mach-O

> Mach-O(Mach Object)是macOS、iOS、iPadOS存储程序和库的文件格式。对应系统通过应用二进制接口(application binary interface，缩写为 ABI)来运行该格式的文件。
>
> Mach-O格式用来替代BSD系统的a.out格式。Mach-O文件格式保存了在编译过程和链接过程中产生的机器代码和数据，从而为静态链接和动态链接的代码提供了单一文件格式。

## Mach-O格式常见文件

- 目标文件`.o`
- 库文件
  - `.a`
  - `.dylib`
  - `Framework`
- 可执行文件
- `dyld`
- `.dsym`

可在终端通过`file`命令查看文件类型 ![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/65bbb899bb904d8298f7083005e0ac87~tplv-k3u1fbpfcp-watermark.image) 可看到当前`Demo`文件是一个`Mach-O`类型的64位`x86_64`架构的可执行文件

## Mach-O文件结构

在`MachOView`工具上查看一个Mach-O文件 ![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/26ed1aaecad44fcdbd5aa85ed742226d~tplv-k3u1fbpfcp-watermark.image)

能够看到`Mach-O`文件格式由`Header`、`Load Commands`、`__TEXT代码`、`__DATA代码`、`符号表`和以及一些其它信息构成。`dyld`会根据`Load Commands`保存的信息找到具体的代码

`Mach-O`可理解为文件配置加上二进制代码,即：

`Mach-O` = 文件配置 + 二进制代码

用苹果官方图概括即：

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/166321d829e1498a9621812cdd69a030~tplv-k3u1fbpfcp-watermark.image)

`Mach-O`的组成结构如图所示包括了：

- ```
  Header
  ```

   包含该⼆进制⽂件的⼀般信息

  - 字节顺序、架构类型、加载指令的数量量等
  - 使得可以快速确认⼀些信息，⽐如当前⽂件⽤于32位还是64位，对应的处理器是什么、文件类型是什么

- ```
  Load commands
  ```

   ⼀张包含很多内容的表

  - 内容包括区域的位置、符号表、动态符号表等

- `Data` 通常是对象文件中最大的部分 -包含`Segement`的具体数据

### Header

64位的`Header`结构

```c
struct mach_header_64 {
	uint32_t	magic;		/* 魔术，快速定位属于64还是32位 */
	cpu_type_t	cputype;	/* cpu类型，如ARM */
	cpu_subtype_t	cpusubtype;	/* cpu具体类型，如arm64/armv7 */
	uint32_t	filetype;	/* 文件类型 */
	uint32_t	ncmds;		/*loadCommands数量*/
	uint32_t	sizeofcmds;	/*loadCommands大小 */
	uint32_t	flags;		/* 标志位标识二进制文件支持功能。主要和系统加载、连接有关 */
	uint32_t	reserved;	/* reserved */
};

复制代码
```

在`MachOView`工具上查看Header ![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5a4fbdfcdb5b4962a5091e78a0125853~tplv-k3u1fbpfcp-watermark.image)

使用`objdump --macho -private-header`命令查看`Mach-O`文件，输出对应`Header`数据结构，与上面对应

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f72c038f56ac4c52aabc9cf4fe834989~tplv-k3u1fbpfcp-watermark.image)

### LoadCommands

64位的`load_command`结构

```c
struct segment_command_64 { /* for 64-bit architectures */
	uint32_t	cmd;		/* command的类型 LC_SEGMENT_64 */
	uint32_t	cmdsize;	/* section_64大小 */
	char		segname[16];	/* 段名称 */
	uint64_t	vmaddr;		/* 段的虚拟内存地址 */
	uint64_t	vmsize;		/* 段的虚拟内存大小 */
	uint64_t	fileoff;	/* 段的文件偏移量 */
	uint64_t	filesize;	/* 段在文件中的大小 */
	vm_prot_t	maxprot;	/* 最大的虚拟机保护 */
	vm_prot_t	initprot;	/* 最初的虚拟保护 */
	uint32_t	nsects;		/* 段中的section数 */
	uint32_t	flags;		/* 标记 */
};
复制代码
```

![img](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a9b75d94c6f948f0abb439b34c0d12c1~tplv-k3u1fbpfcp-watermark.image)

其它的一些`load_commands`：

|                       |                                              |
| --------------------- | -------------------------------------------- |
| LC_SEGMENT_64         | 将⽂件中(32位或64位)的段映射到进程地址空间中 |
| LC_DYLD_INFO_ONLY     | 动态链接相关信息                             |
| LC_SYMTAB             | 符号地址                                     |
| LC_DYSYMTAB           | 动态符号表地址                               |
| LC_LOAD_DYLINKER      | 使⽤谁加载，我们使用dyld                     |
| LC_UUID               | ⽂件的UUID                                   |
| LC_VERSION_MIN_MACOSX | 支持最低的操作系统版本                       |
| LC_SOURCE_VERSION     | 源代码版本                                   |
| LC_MAIN               | 设置程序主线程的⼊口地址和栈⼤小             |
| LC_ENCRYPTION_INFO_64 | 获取加密信息                                 |
| LC_LOAD_DYLIB         | 依赖库的路径，包含三方库                     |
| LC_FUNCTION_STARTS    | 函数起始地址表                               |
| LC_DATA_IN_CODE       | 定义在代码段内的非指令的表                   |
| LC_CODE_SIGNATURE     | 代码签名                                     |

### Data

存放数据：代码，字符常量，类，方法等

- 代码段（__TEXT）

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5696a3f463ee45599b20bb6618683d66~tplv-k3u1fbpfcp-watermark.image)

代码段开始地址是0，所以读内存当中的`MachO`开始的位置从代码段开始读的

![img](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c8b742dfebb04817b34c54aba8e1e51c~tplv-k3u1fbpfcp-watermark.image)

- 数据段（__DATA）

![img](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/493e94412c9a4ba790051ec62de585f4~tplv-k3u1fbpfcp-watermark.image)

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/008aadbe30f64f5a938499b71532eb5c~tplv-k3u1fbpfcp-watermark.image)

```
__objc_selrefs:记录了几乎所有被调用的方法

__objc_classrefs和__objc_superrefs：记录了几乎所有使用的类

__objc_classlist:工程里所有的类的地址
```

删除无用的类

MachO文件中`__objc_classrefs`段记录里了引用类的地址`，__objc_classlist`段记录了所有类的地址，我们可以认为取两者的差值就可以获得未使用类的地址，然后进行符号化，就可以取得未使用类的信息。

大家可以使用[classunref](https://github.com/xuezhulian/classunref)这个工具实现未使用类的查找。

删除未使用的方法

当我们将无用的类删除完毕之后，在已经使用的类里边很有可能依然会有未使用的方法。

前边我们已经提到过LinkMap中保存了工程的信息，而我们所有已经被包含到项目中的方法可以通过LinkMap获取。

在[classunref](https://github.com/xuezhulian/classunref)的启发下,笔者利用python实现了未使用方法的自动化方式

因为py实在不太熟悉加上笔者比较懒，请大家忽略一些语法、接口设计不规范等等的问题

1.使用指令`grep '[+|-]\[.*\s.*\]' xxx-linkMap.txt`指令我们得到所有被包含到工程到项目中的代码

```
// 获取所有的方法
def method_readRealization_pointers(linkMapPath,path):
# all method
lines = os.popen("grep '[+|-]\[.*\s.*\]' %s" % linkMapPath).readlines()
// 需要忽略的方法
lines = method_ignore(lines,path);
pointers = set()
for line in lines:
    line = line.split('-')[-1].split('+')[-1].replace("\n","")
    line = line.split(']')[0]
    line = str("%s]"%line)
    pointers.add(line)
if len(pointers) == 0:
    exit('Finish:method_readRealization_pointers null')
print("Get all method linkMap pointers...%d"% len(pointers))
return pointers
```

2.考虑到大家项目中使用了大量的三方库，而三方库的方法有许多并未使用,所以通过`method_ignore`方法进行忽略,这样获取的差值的集合中就不会包括三方库的未使用方法

```
def method_ignore(lines,path):
  print("Get method_ignore...")
  effective_symbols = set()
  // 获取所有需要忽略类名的前缀(例如YYModel 会以前缀YY的方式作出忽略)
  prefixtul = tuple(class_allIgnore_Prefix(path,'',''))
  getPointer = set()
  
  // 此处是为了忽略Setter Getter方法
  for line in lines:
      classLine = line.split('[')[-1].upper()
      methodLine = line.split(' ')[-1].upper()
      if methodLine.startswith('SET'):
         endLine = methodLine.replace("SET","").replace("]","").replace("\n","").replace(":","")
         print("methodLine:%s endLine:%s"%(methodLine.lower(),endLine.lower()))
         if len(endLine) != 0:
            getPointer.add(endLine)
  getPointer = list(set(getPointer))
  getterTul = tuple(getPointer)
  
  for line in lines:
      classLine = line.split('[')[-1].upper()
      methodLine = line.split(' ')[-1].upper()
      if (classLine.startswith(prefixtul)or methodLine.startswith(prefixtul)  or methodLine.startswith('SET') or methodLine.startswith(getterTul)):
          continue
      effective_symbols.add(line)
  
  if len(effective_symbols) == 0:
      exit('Finish:method_ignore null')
  return effective_symbols;
```

3.使用指令`otool -v -s __DATA__objc_selrefs`指令我们可以得到所有已经被实现的方法

```
def method_selrefs_pointers(path):
  # all use methods
  lines = os.popen('/usr/bin/otool -v -s __DATA __objc_selrefs %s' % path).readlines()
  pointers = set()
  for line in lines:
       line = line.split('__TEXT:__objc_methname:')[-1].replace("\n","").replace("_block_invoke","")
       pointers.add(line)
  print("Get use method selrefs pointers...%d"% len(pointers))
  return pointers
```

3.利用步骤1和步骤2的差值可以获取到还未使用的方法。

```
def method_remove_Realization(selrefsPointers,readRealizationPointers):
  if len(selrefsPointers) == 0:
     return readRealizationPointers
  if len(readRealizationPointers) == 0:
     return null
  methodPointers = set()
  for readRealizationPointer in readRealizationPointers:
      newReadRealizationPointer = readRealizationPointer.split(' ')[-1].replace("]","")
      methodPointers.add(newReadRealizationPointer)
  unUsePointers = methodPointers - selrefsPointers;

  dict = {}
  for unUsePointer in unUsePointers:
      dict[unUsePointer] = unUsePointer
  
  for readRealizationPointer in readRealizationPointers:
      newReadRealizationPointer = readRealizationPointer.split(' ')[-1].replace("]","")
      if dict.has_key(newReadRealizationPointer):
          dict[newReadRealizationPointer] = readRealizationPointer
          str = dict[newReadRealizationPointer]
  
  return list(dict.values())
```

感兴趣的同学可以在这里下载到修改版的[ ONLClassMethodUnref](https://github.com/LessonFirst/ONLClassMethodUnref)

无用类检测中使用了capstone，通过反汇编来识别函数指令中类的调用情况

[**App瘦身**](https://juejin.cn/post/6844904130205466638)

##### [**mach-o文件如何分析多余的类和方法**](https://mp.weixin.qq.com/s/z0eK4cOfvYWFhHl16rGnEg)

[Lazy/Non-lazy Binding](https://juejin.cn/post/7001842254495268877)

[iOS程序员的自我修养](https://juejin.cn/post/6844903912143585288)

