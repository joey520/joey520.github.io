---
toc: true
title: 检验mach-O中未使用的类
date: 2020-02-04 14:47:26
categories: [iOS]
tags: [包体积]
---

## 前言

查找未使用的类有两种方式，一种是基于代码的静态扫描，但是非常耗时，而且逻辑复杂。而`mach-O`中通过`classlist`和`classrefs`差集可以快速的定位出哪些未在代码中显式被调用的类。而且速度非常快，所以优先采用这种方式实现，具体脚本见[github](https://github.com/joey520/Blogs/tree/master/包体积优化/未使用类检测)。

<!--more-->

## Ctypes

要在`python`中解析`mach-O`文件就依赖大量的结构体，而`ctypes`提供了一种类似结构体的序列化数据的方式。`ctypes`的使用并不复杂。只需要定义一个类并继承自`structure`,并在``_fieds_``中定义好结构即可。

``_fieds_``可以理解为就是一个结构体的定义。需要把成员变量名和类型一一对应。这样`struct`就会根据``_fieds_``进行内存布局。

`ctypes`提供的方法`string_at`可以把当前结构体类数据转化成二进制数据。因此我们对结构体类进行封装，构建了一个基类的`encode`和`decode`方法，进行二进制数据的序列化和反序列化：

```python
class StructBase(Structure):
    def encode(self):
        return string_at(addressof(self), sizeof(self))

    def decode(self, data):
        memmove(addressof(self), data, sizeof(self))
        return len(data)
```

下面以一个实例来试验一下:

```python
class mach_header(StructBase):
		_fields_ = [
		('magic', c_uint32),
		('cputype', c_int32),
		('cpusubtype', c_int32),
		('filetype', c_uint32),
		('ncmds', c_uint32),
		('sizeofcmds', c_uint32),
		('flags', c_uint32),
	]
```

这个类是在64位架构下`mach-o`的头结构体的实现。此时我们已经可以读取`mach-o`的头部了:

```python
path = sys.argv[1]
    with open(path, 'rb') as f:
        data = f.read()
        header = mach_header()
        header.decode(data)
        print(header.magic, header.cputype, header.filetype, header.ncmds)
        
输出:
4277009103 16777223 2 23
```

但是这里有一个问题是，c结构体转ctypes结构体类很麻烦，必须要手动一个个写。

而对于`mach-o`的loader的结构体定义，估计手写完我会疯掉。于是想到写一个脚本来做结构体的解释器，把c结构体转换为ctype的结构体，虽然不能完美转换，但是至少可以省去大部分工作，而且以后还有类似需求可以再拿出来使用。

## 结构体解释脚本

### 解析文本

首先读入文本文件，然后为了避免干扰去掉注释的部分，采用正则表达式把注释去掉

```python
def __removeAnnotation(self, content):
        blockPattern = "/\\*[\\s\\S]*?\\*/"
        linePatter = "//.*?\\n"
        ret = re.sub(re.compile(blockPattern, re.S), "", content)
        ret = re.sub(re.compile(linePatter, re.S), "", ret)
        return ret
```

为了提高代码复用性。抽象出`struct`类和`union`类，它的结构很简单，保存了名字和相应的成员变量，成员变量封装成一个`VarPair`的类。

之后开始解析。按行读取文件，按空格将文件分解成tokens流。

### 解析宏命令

宏命令是需要解析的，因为我们需要这些大量的宏定义。因此如果当前行里有`#define`这个token，表明进入了宏命令，由于C语言宏命令具有多行的特性，灵活性太高，此时只处理单行的简单宏定义，对于多行宏命令，直接打印出warning进行手动适配。

### 解析块域

C语言的定义是按块来的，所以我们可以在遍历中把一个块的数据收集起来，当块结束时再根据类型进行解析。所以我们定义了快类型的枚举:

```python
class StructureType():
    unkonwn = 0
    struct = 1
    union = 2
```

通常我们按行进行遍历，碰到一个`struct`符号，并且此时没有还没有处于一个块域之中，表明进入了`stuct`的域（需要特别考虑剔除当前处于`typedef`的情况）。则设置当前的块域类型，并开始搜集数据。此时需要进行花括号的匹配，每次遇到一个左花括号计数加1，遇到一个右花括号计数减1。一旦计数减一之后归零了，表示块域结束了。

此时根据块域的类型调用不同的解析，对于结构体，依然是把当前块域按行分解，然后解析tokens流，把结构体名放入，然后每匹配到一个成员变量，把名字和类型的`pair`加入到结构体中。如果碰到内部有结构体或者是`union`，取出命名和类型加入。

如果碰到内容不符合规则，直接设置`type`为`UnknowType`，并把整行数据保存作为变量名，然后特别打印出来，方便后续手动修改。

### 解析数据

经过前面的处理，最终得到三个数组，宏定义的类数组，`union`的类数组和`struct`的类数组。

优先遍历宏定义数组，把宏的`pair`转换成`python`的全局变量

在遍历`unions`数组，由于c类型与`ctypes`类型不一致，需要有一个映射表表把类型进行映射。并且转换成`python`文本时一定要注意缩进等级。遍历`structs`数组，同理。

过程中碰到无法匹配的类型，需要打印出出现的结构体名或者是联合体名，已经不匹配的类型，方便手动进行修改。

最终将文本文件写入本地文件。

脚本虽然并不能完美解析，主要是由于import，typedef以及一些宏命令等特定类型导致，最好是解析不了时全部保存，然后打印出来，手动检查修改。

[脚本文件](https://github.com/joey520/Blogs/blob/master/包体积优化/未使用类检测/CStructToCYpes.py)，最终终于把<mach-o/loader.h>中的结构体全部转换成了[ctypes文件](https://github.com/joey520/Blogs/blob/master/包体积优化/未使用类检测/MachODefines.py)。

最后手动看一下结构有没有问题。以及未能识别的`type`，需要手动进行下处理。

## mach-O解析

有了相应的`ctypes`结构体，就可以开始解析`mach-O`文件了，由于我们的目的很明确，所以暂时不需要把整个文件都解析了，这样可以提高脚本的执行效率。 只关注`classlist`和`classref`:

1.首先读入文件，解析首部。

2.依次遍历`loadcommand`，把符号表，`indirect table`和所有的`LC_SEGMENT_64`的`section`单独保存下来。

3.通过`sectionname`找到``__objc_classlist``和``__objc_classrefs``。两者保存都是`class`对象数据段的指针。

4.取两者差集，然后开始按定义的`class`的数据结构和`class_ro`的数据结构解析数据。把转换成封装的`class`对象保存下来。

<b>5.由于mach-O中没有被显示调用的类，比如superclass这种并不会存在于classrefs，所以需要遍历去掉是superclass的类</b>。

6.根据类白名单掉白名单中的类。

由于我们只解析了`classlist`和`classrefs`，脚本的运行非常快速，可以随时随意的跑脚本检测。

<b>由于可执行文件具有`PAGEZERO`因此它的虚拟地址的其实和动态库等不一样，需要特别在解析时注意。</b>

## 动态检测类是否无用

在上一步中已经可以通过脚本拿到未使用的类，虽然我们过滤了仅仅作为superclass这样情况，但是还有其它的比如是IBuild构建的类等等，通常需要手动过滤一遍。而我再MSDK的工程中直接扫出来100多个，手动过滤是很麻烦的。于是想到利用脚本来实现。一个取巧的办法是把这些类移除buildFile然后编译来判断。相当于删除了，虽然头文件import可能还在，但至少可以帮助结局一大部分情况。

这些重复的事情可以由脚本来完成，脚本的实现需要借助于pbxproj这个python开源库，主要工作有一下两步:

1.遍历未使用的类，遍历buildTarget列表，把相应的buildFile去掉。

2.尝试编译，如果成功则表明该文件确实无用。如果失败，表明还有隐式的使用。

这里有有一点需要注意，类与buildFile的命名不一定一致，应该可能找不到当前类的buildFile。

所以单纯依靠类名来查找文件不是一个好办法，好在`linkmap`在会把变异的符号和文件对应起来：

例如：

```
[  2] /Users/joey.cao/Desktop/MSDK/ios-objctest/Demo/DJISdkDemo/DerivedData/DJISdkDemo/Build/Intermediates.noindex/DJISdkDemo.build/Debug-iphonesimulator/SDK QA.build/Objects-normal/x86_64/DJIFlyLimitCircle.o

...
0x100001B80	0x000000F0	[  2] -[DJIFlyLimitCircle description]
0x100001C70	0x00000040	[  2] -[DJIFlyLimitCircle realCoordinate]
0x100001CB0	0x00000040	[  2] -[DJIFlyLimitCircle setRealCoordinate:]
...
```

那么通过查找linkMap就可以找到某个类对应的buildFile。然后再把该buildFile从target的` compile Files`中移除，然后再进行build就可以知道是不是真的不需要这个类。

当然这一种方式不一定完全可靠，但是可以大大减少了手工检查的消耗。

## 参考资料

https://docs.python.org/3.8/library/ctypes.html?highlight=ctypes#module-ctypes

https://www.ibm.com/developerworks/cn/linux/l-cn-pythonandc/

