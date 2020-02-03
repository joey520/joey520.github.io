---
toc: true
title: 深入理解MachO结构
date: 2020-01-30 17:36:37
categories:
tags:
---

## 前言

趁着这个假期好好补一下基础知识，之前虽然在学习`Bitcode`时了解了`Mach-O`的知识，但是还是模棱两可，这一次参考`mach-o view`源码一步步学习。但是这个[源码](https://github.com/gdbinit/MachOView)可以看到最后更新时间还是2015年,不仅下下来各种无法编译，并且很多`mach-o`信息也无法解析，于是我对它进行一些修改，可以直接拿我我fork过来的[工程](https://github.com/joey520/MachOView) ，已经兼容了苹果最新的`mach-o`结构。

## 结构分析

首先大致描述下`Mach-O`文件的结构：

```mermaid
graph LR
A[header] --> B[LoadCommands]
B --> C[segments]
C --> D[sections]
D --> E[Data]
```



### headers

`Mach-O`的`headers`对文件进行了简要的描述，结构如下:

```c
/*
 * The 32-bit mach header appears at the very beginning of the object file for
 * 32-bit architectures.
 */
struct mach_header {
	uint32_t	magic;		/* mach magic number identifier */
	cpu_type_t	cputype;	/* cpu specifier */
	cpu_subtype_t	cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
};
/* Constant for the magic field of the mach_header (32-bit architectures) */
#define	MH_MAGIC	0xfeedface	/* the mach magic number */
#define MH_CIGAM	0xcefaedfe	/* NXSwapInt(MH_MAGIC) */

/*
 * The 64-bit mach header appears at the very beginning of object files for
 * 64-bit architectures.
 */
struct mach_header_64 {
	uint32_t	magic;		/* mach magic number identifier */
	cpu_type_t	cputype;	/* cpu specifier */
	cpu_subtype_t	cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
	uint32_t	reserved;	/* reserved */
};

/* Constant for the magic field of the mach_header_64 (64-bit architectures) */
#define MH_MAGIC_64 0xfeedfacf /* the 64-bit mach magic number */
#define MH_CIGAM_64 0xcffaedfe /* NXSwapInt(MH_MAGIC_64) */
```

定义了`Mach-O`文件的格式以及文件的`magic number`。

```shell
otool -h /Users/joey.cao/Desktop/Learning/MyBlog/LocalDemos/LearnHook/DerivedData/LearnHook/Build/Products/Debug-iphonesimulator/LearnHook.app/LearnHook
Mach header
      magic cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf 16777223          3  0x00           2    22       3104 0x00200085
```

使用了`magic`用于标志`Mach-O`文件，架构和文件类型已经`LoadCommands`数量，这样依次加载`LoadCommands`。`flags`按位描述了`Mach-O`文件的特性。

### LoadCommands

`header`中标注了`loadcommand`个数。然后动态连接器就根据`loadcommands`依次加载。对于不同的`loadcommand`有不同的结构体进行解析，通常我们只需要关注以下几种:

### 符号绑定

符号绑定是把`undefined external symbols`绑定到指定函数。当程序开始加载的时候，动态连接器会把动态库加载到程序的内存空间，然后把程序中的`undefined external symbols`替换成动态库中的函数定义的地址。

动态链接器可以在加载时或者是运行时绑定程序，具体取决于编译时的选项：

`just in time binding`（`lazy binding`）表示动态链接器将会在初次使用符号引用时进行绑定。动态链接器加载动态库依赖于程序加载的时机，并且动态库的符号直到被使用时才被绑定。

`load time binding`表示动态链接器在加载时即绑定所有的符号，使用`ld`的`bind_at_load`选项可以在加载时时绑定所有的外部符号。如果不设置该选项默认为`just in time binding`（`lazy binding`）`。

在预绑定时，符号会被预绑定到一个默认的地址，静态链接器会给每一个`undefined external symbol`设置默认地址来使用。在运行时，动态链接器只会验证这些默认地址没有因为编译或者重计算而改变，如果发生了改变则动态链接器会清除掉`undefined external symbol`预绑定的地址，然后变成`just in time binding`方式进行符号的绑定。

预绑定需要每一个`framework`指明它需要的虚拟内存地址空间，所有预绑定的地址不会发生重复。通过设置`LD`的`prebind`选项可以开启预绑定。

弱引用的符号动态链接器如果找不到相应的定义则会设置为`NULL`然后继续加载程序，程序可以运行检查一个引用是否为`NULL`,如果是则不会处理。

### 符号查找

符号代表了一个函数，数据变量，常量在可执行文件中的地址。OSX v10.1中引入了`two-level symbol`命名空间规则。第一级的命名指的是包含这个符号的库名，第二级就是符号名。通过两级命名规则，当静态链接器记录到了外部引用的符号时，同时记录库名和符号名。两级命名法不仅提高了查找效率，可以指定到对应的库查找符号，也解决了不同库文件符号冲突问题。

实践一下，在`main.c`中引用`hello.c`中的`hello`方法查看符号如下:

```shell
nm -nm hello.o
                 (undefined) external _printf
0000000000000000 (__TEXT,__text) external _hello
nm -nm a.out
                 (undefined) external _printf (from libSystem)
                 (undefined) external dyld_stub_binder (from libSystem)
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
0000000100000f20 (__TEXT,__text) external _main
0000000100000f60 (__TEXT,__text) external _hello
```

可以看到由于是直接静态链接到一起的，则符号已经保存在最终的可执行文件`a.out`中了。

然后我们把`hello.c`制作成动态库再尝试和`main.c`进行链接如下:

```shell
clang -dynamiclib hello.c -o hellodynamic
clang hellodynamic main.c -o main.out
```

此时生成的`main.out`和`a.out`运行结果相同，不同的时一个是`main.out`是运行时动态绑定的符号。查看符号进行对比查看，首先是`hello.o`和`hellodynamic`:

```objective-c
nm -nm hello.o
                 (undefined) external _printf
0000000000000000 (__TEXT,__text) external _hello
nm -nm hellodynamic
                 (undefined) external _printf (from libSystem)
                 (undefined) external dyld_stub_binder (from libSystem)
0000000000000f60 (__TEXT,__text) external _hello
```

由于`hello.o`只参与编译，对于未知的符号`_printf`只需要标记出来即可，只有链接时才知道符号需要到哪个库里去找，而对于`hellodynamic`，由于动态库已经是编译并链接好的，因此已经指定了链接信息，所以`_printf`已经指定需要到`libSystem`中查找，并且动态库的链接需要用用到`dyld_stub_binder`。

然后我们对比下`a.out`和`main.out`：

```objective-c
nm -nm a.out
                 (undefined) external _printf (from libSystem)
                 (undefined) external dyld_stub_binder (from libSystem)
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
0000000100000f20 (__TEXT,__text) external _main
0000000100000f60 (__TEXT,__text) external _hello

nm -nm main.out
                 (undefined) external _hello (from hellodynamic)
                 (undefined) external _printf (from libSystem)
                 (undefined) external dyld_stub_binder (from libSystem)
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
0000000100000f40 (__TEXT,__text) external _main
```

由于`a.out`是源码编译，因此符号都合并到`a.out`的符号表中了，因此可以看出程序段的`_hello`。而`main.out`是动态链接的，只指定`_hello`符号需要到`hellodynamic`中查找。

### 动态绑定

当定义在其它文件的符号被使用时会创建符号引用，符号引用分为`lazy`和`no lazy`。

`no lazy symbols`会在模块加载时就被动态链接器进行绑定，本质上就是一个符号指针，指向一个指针大小的数据（函数指针），编译器会为数据符号和函数符号创建`no lazy symbols`（也就是访问内部的符号）。

`lazy symbols`仅仅在符号第一次被使用时才进行绑定。之后的对符号的使用直接`jmp`到符号的定义处。`lazy symbols`由符号指针和`symbol stub`共同组成，少量的代码直接解引用并且通过符号指针跳转。编译器在访问一个由其他文件定义的符号时才会生成`lazy symbols`。

`OSX x86_64`使用`mach-O`文件代替`ELF`文件。静态链接器负责生成所有的`all stub functions, stub helper functions, lazy and non-lazy pointers, as well as the indirect symbol table needed by the dynamic loader (`dyld`).`

`lazy symbol pointer`即在初始化时赋给胶水代码的一个地址调用链接器的胶水代码`dyld_stub_binding_helper`，这个胶水代码会调用动态链接器的方法来进行实际的`stub`绑定工作。`dyld_stub_binding_helper`会返回外部方法的实际地址。上面我们也看的了进行连接后多出来的`dyld_stub_binder`符号。

链接器还会进行一些优化，去掉同一个module内的`symbol stubs`，修改方法调用，直接调到最终的需要调的地方。去掉重复的`symbol stub`。

## 放置独立的代码

`Position-independent code or PIC`即通过动态链接器加载另一份未经过虚拟内存处理的代码的技术（加载动态库等）。`mach-O`进行`PIC`依赖于`__DATA segment`总是相对于`__TEXT segment`固定的偏移。所以动态加载时绝对不会相对`__DATA segment`移动它的`__TEXT segment`。因此任何方法都可以通过当前地址加一定的偏移来访问想要访问的数据。`mach-O`文件进行`PIC`和`GOT`（`global offser table`）很相似。不过的是`mach-O`文件的地址偏移都是直接偏移，而`ELF`文件是通过`GOT`。

在`zGCC3.1`中引入了新的选项`-mdynamic-no-pic`减少可执行文件代码的大小并通过消除独立地址代码的引用来提高性能，而是把它们转换为`Undifined symbols`。`Xcode`进行编译时默认是打开这个选项的。

在汇编器处理重定位时，如果当前的`label`是一个局部的，先前的非局部`label`将会作为外部重定位。，则直接使用。

## X86_64代码模型

`x86_64`在`OSX`中只有一种用户空间代码模型。<b>在`mach-O`中所有的静态初始化存储必须填满4GB</b>，填充的数据必须为0。

## 参考资料

https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/MachOTopics/0-Introduction/introduction.html#//apple_ref/doc/uid/TP40001519

https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/dyld.3.html