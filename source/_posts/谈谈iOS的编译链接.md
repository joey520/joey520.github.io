---

toc: true
title: 谈谈iOS的链接问题
date: 2019-11-23 17:29:33
categories: iOS
tags: [LLVM,LINKER]
---

## 前言

首先抛出几个个问题：

1.什么是`header serach path`,什么又是`framework search path`？[答案](#answer1)

2.`#import`,` #include`, `@class`和`@import`的区别是什么。循环`#import`一定会导致编译失败吗？[答案](#answer1)

3.可以同时通过`#import <>`和使用header seach path来`#import ""`动态库里的文件吗？[答案](#answer3)

4.如果编译时主工程（注意不是来自外部link的库）中有个Class的.m文件没有被加入到指定的target的complie files里会导致什么问题，如果是一个Category的文件呢？[答案](#answer4)

5.如果一个类里的方法只有声明没有定义，则编译时会出现`undefinde symbol`吗, 为什么？[答案](#answer4)

6.如果一个静态库里面有两个相同的类会出现`duplictate symbols`错误吗？如果是动态库呢，如果是可执行文件呢？[答案](#answer6)

7.如果主项目和连接的静态库有相同的类和相同的符号时会出现什么问题？动态库也会这样吗？[答案](#answer7)

8.动态库和静态能呈现多层的链式链接关系吗？在链式链接的关系中能否跨层级直接调用到链接链中任意库文件中的方法吗？可以从链接链后面的库文件直接调用前面的库文件中的方法吗？[答案](#answer8)

9.什么`Other Link Flag`,什么是`_all-load`, 什么是`-Objc`, 什么是`-force-load`，什么是`-dead_strip`?[答案](#answer9)

<!--more-->

## 什么是链接

如果有注意过xcode的编译输出可以看到，其实是把每一个.m文件编译成.o文件再通过ld工具链接成`mach-O`文件。那么合成`mach-O`文件这一步即是链接的过程。

还是以一个小Demo说明，首先我们创建一个main.c:

```c
extern int shared;
static int age = 27;
int main() {
	int a = 100;
	swap( &a, &shared);
	return 0;
}
```

然后再创建一个test.c:

```c
int shared  = 1;
void swap (int *a, int *b) {
	*a ^= *b ^= *a ^= *b; 
}
```

然后通过clang分别编译出对应的目标文件：

```shell
$ clang -c main.c test.c
main.c:5:2: warning: implicit declaration of function 'swap' is invalid in C99
      [-Wimplicit-function-declaration]
        swap( &a, &shared);
        ^
1 warning generated.
```

会出现warnig警告，主要是因为编译器swap方法是test.c内部的，而没有进行显式声明导致的，如果是显式的对方法进行了定义，并且进行include则不会提示，当然我们也可以通过加入`-Wno`来屏蔽掉这个警告:

```shell
$ clang -Wno-implicit-function-declaration -c main.c
```

这个时候编译器已经把源代码信息存储在目标文件`main.o`和`test.o`中了，并且不同的信息存储在不同段中了。我们利用otool查看分别查看两个目标文件的`load commands`:

```shell
$ otool -r main.o
$ otool -r test.o
```

只截取`LC_SEGMENT_64`这个command，对于`Load command`不了解的可以查看[这里]()。可以看到结果如下:

```shell
##main.o的`LC_SEGMENT_64`cmd输出
   vmaddr 0x0000000000000000
   vmsize 0x0000000000000098
  fileoff 472
 filesize 152
 Section
  sectname __text
   segname __TEXT
      addr 0x0000000000000000
      size 0x0000000000000035
    offset 472
     align 2^4 (16)
    reloff 624
    nreloc 2
     flags 0x80000400
##test.o的`LC_SEGMENT_64`cmd输出
   vmaddr 0x0000000000000000
   vmsize 0x0000000000000090
  fileoff 552
 filesize 144
 Section
  sectname __text
   segname __TEXT
      addr 0x0000000000000000
      size 0x000000000000002c
    offset 552
     align 2^4 (16)
    reloff 0
    nreloc 0
     flags 0x80000400
 reserved1 0
 reserved2 0
Section
```

main.o表示的是物理地址为页中偏移472，大小为152。 虚拟地址为0和大小为152。test.0表示的是物理地址为页中偏移552，大小为144。 虚拟地址为0和大小为144。 而且代码段总是在地址起始位置。

### 链接过程

我们都知道程序在启动时需要被load到内存中，而虚拟地址是从0开始的，因此如果现在`main.o`和`test.o`链接到一起，则不可能同时存在两个从虚拟地址0位开始的。可见链接时会对段进行合并与重定位。

LLVM的做法是把相似的段合并到一起，把两者的`text`段，`data`段，符号表等等分别进行合并。利用LD工具可以把目标文件进行链接：

```shell
$ LD main.o test.o -e _main -o main.out -lSystem /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/clang/11.0.0/lib/darwin/libclang_rt.osx.a
```

`-e`指定了程序起始的符号，即main函数。同时链接还依赖libclang_rt.osx.a这个库。

这么简单的一行语句实际需要经过以下步骤：

1.取出所有目标文件的符号表合成全局符号表

为了验证我们在`main.c`文件的main函数之前插入几句代码,并在main函数用使用printf标准函数来输出：

```c
  2 void test() {
  3
  4 }
  5 static void test1() {
  6 }
  7 static int age = 27;
  8 int age1 = 30;
  9 int age2;
```

重新编译得到main1.o，并查看其符号，此时编译器已经给符号附上了不同的链接属性，并且赋值为对应段中偏移：

```shell
$ clang -c main.c -o main1.o
$ nm -nm main1.o
                 (undefined) external _printf
                 (undefined) external _shared
                 (undefined) external _swap
0000000000000000 (__TEXT,__text) external _test
0000000000000004 (common) (alignment 2^2) external _age2
0000000000000010 (__TEXT,__text) external _main
0000000000000048 (__DATA,__data) external _age1
```

由于这shared和swap这两个符号是未能找到定义的因此在目标文件中修饰为undefined，并且不会被放入当前目标文件的段表中，因为它们既没有代码，又没值。 而`test()`,`age1`由于可以被外部可见被修饰为external。<b>age2虽然没有加`external`修饰符，但是是一个external的符号，common修饰的一般是未定义的值，它的值表示它需要的字节数，没有包含在section中，在链接的过程中才会进行分配</b>。而`test1()`,`age1`由于不被外部可见，直接连符号都省了。符号是链接时查找到指定代码的依据，因此为了合并目标文件，第一步必须把符号表整合起来。而符号名是符号的唯一标志，因此必须不能冲突，也不缺失。

作为对比，我们查看一下生成的Maco-O文件的符号表:

```shell
$ nm -nm main.out
                 (undefined) external _printf (from libSystem)
                 (undefined) external dyld_stub_binder (from libSystem)
0000000100000000 (__TEXT,__text) [referenced dynamically] external __mh_execute_header
0000000100000ed0 (__TEXT,__text) external _test
0000000100000ee0 (__TEXT,__text) external _main
0000000100000f50 (__TEXT,__text) external _test1
0000000100000f60 (__TEXT,__text) external _swap
0000000100002008 (__DATA,__data) non-external __dyld_private
0000000100002010 (__DATA,__data) external _age1
0000000100002014 (__DATA,__data) external _shared
0000000100002018 (__DATA,__data) external _age2
```

链接之后，undefined符号终于找到了自己的归属。。。age2也找到了自己的归属。而prinft也找到，只是标记了需要从`libSystem`标准库中去找。这样符号表就组合到了一起。

---

2.合并section进行地址重定位

在链接的过程会地址进行重定位，如上可以看到的符号对应的地址已经发生了改变我们通过size工具可以查看到此时各个segment的情况:

```shell
$ size -x -l -m main.out
Segment __PAGEZERO: 0x100000000 (vmaddr 0x0 fileoff 0)
Segment __TEXT: 0x1000 (vmaddr 0x100000000 fileoff 0)
	Section __text: 0xbc (addr 0x100000ed0 offset 3792)
	Section __stubs: 0x6 (addr 0x100000f8c offset 3980)
	Section __stub_helper: 0x1a (addr 0x100000f94 offset 3988)
	Section __cstring: 0xa (addr 0x100000fae offset 4014)
	Section __unwind_info: 0x48 (addr 0x100000fb8 offset 4024)
	total 0x12e
Segment __DATA_CONST: 0x1000 (vmaddr 0x100001000 fileoff 4096)
	Section __got: 0x8 (addr 0x100001000 offset 4096)
	total 0x8
Segment __DATA: 0x1000 (vmaddr 0x100002000 fileoff 8192)
	Section __la_symbol_ptr: 0x8 (addr 0x100002000 offset 8192)
	Section __data: 0x14 (addr 0x100002008 offset 8200)
	total 0x1c
Segment __LINKEDIT: 0x1000 (vmaddr 0x100003000 fileoff 12288)
total 0x100004000
```

可以看到代码段已经变成从逻辑地址0x10000000开始了。原来是一个叫做`_PAGEZERO`的segement占了，这一段主要是系统的区域，因此不允许访问。它的地址是0x0，这也是为啥我们访问空指针是会提示`EXC_BAD_ACCESS 0x0`这样类似的错误。动态连接器调用相关的代码，代码中的一些字符串常量`__cstring`,数据常量区，数据区都被插入进来了也被插入进来。

需要注意的是`__la_symbol_pt`表示的是延迟符号指针，可以用于调用一些可执行文件中没有定义的函数，例如前面的`printf`，它可以允许动态连接器进行延迟链接。 

### 小结

简单说链接器就是把每个单独的目标文件进行重装，包括了符号表合并，段合并与地址重定位，动态链接器的调用等等。

那么得找的到对应的目标文件，也得找得到对应的依赖库文件，对应的符号，且不能冲突。解决这些问题，基本就不会出现什么链接问题了。

## iOS的编译

#### 头文件引用

<span id="answer1">clang在进行预处理时会把import的头文件引入并展开，这样就可以在当前文件中使用这些声明的方法了。而import的时候只是传入了一个字符串，编译器是如何找到对应的头文件的呢，那就得依靠搜索路径。</span>

Clang编译器具有和GCC相同[搜索指令](http://gcc.gnu.org/onlinedocs/cpp/Search-Path.html)。对于`#include ""`修饰首先在当前文件夹内查找，然后在自定义的路径中进行搜索。对于`#include <>`修饰的首先在系统标准库中进行搜索，然后是自定义的路径。

但是在使用iOS工程时我们发现只要是在工程的根目录文件下的文件都可以直接通过`#include ""`到，而不需要像C，C++里面一下指定详细的相对路径。这是因为<b>hmap</b>这个东西。默认xcode的build setting里有一个选项是`Uses Headers Map`打开的。我们build一下当前的工程并导出编译log可以看到:

![image-20191127111720240](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/谈谈iOS的编译链接/image1.png?raw=true)

在编译源文件之前，先会创建一些列hmap文件到derivedData中，我们先关注这个工程名+-project-headers.hmap`的文件中。

```shell
$ hexdump -C Build/Intermediates.noindex/TestLink.build/Debug-iphonesimulator/TestLink.build/TestLink-project-headers.hmap
```

我们只截取后面一部分字符串部分如下:

```shell
000000d0  00 00 00 00 00 00 00 00  00 54 65 73 74 53 75 62  |.........TestSub|
000000e0  53 75 62 44 69 72 2e 68  00 2f 55 73 65 72 73 2f  |SubDir.h./Users/|
000000f0  6a 6f 65 79 2e 63 61 6f  2f 44 65 73 6b 74 6f 70  |joey.cao/Desktop|
00000100  2f 4c 65 61 72 6e 69 6e  67 2f 4c 4c 56 4d 2f 64  |/Learning/LLVM/d|
00000110  79 6c 64 2f 44 65 6d 6f  32 2f 54 65 73 74 4c 69  |yld/Demo2/TestLi|
00000120  6e 6b 2f 54 65 73 74 4c  69 6e 6b 2f 53 75 62 64  |nk/TestLink/Subd|
00000130  69 72 2f 53 75 62 53 75  62 44 69 72 2f 00 54 65  |ir/SubSubDir/.Te|
00000140  73 74 53 75 62 53 75 62  44 69 72 31 31 31 2e 68  |stSubSubDir111.h|
00000150  00 54 65 73 74 53 75 62  44 69 72 2e 68 00 2f 55  |.TestSubDir.h./U|
00000160  73 65 72 73 2f 6a 6f 65  79 2e 63 61 6f 2f 44 65  |sers/joey.cao/De|
00000170  73 6b 74 6f 70 2f 4c 65  61 72 6e 69 6e 67 2f 4c  |sktop/Learning/L|
00000180  4c 56 4d 2f 64 79 6c 64  2f 44 65 6d 6f 32 2f 54  |LVM/dyld/Demo2/T|
00000190  65 73 74 4c 69 6e 6b 2f  54 65 73 74 4c 69 6e 6b  |estLink/TestLink|
000001a0  2f 53 75 62 64 69 72 2f  00 41 70 70 44 65 6c 65  |/Subdir/.AppDele|
000001b0  67 61 74 65 2e 68 00 2f  55 73 65 72 73 2f 6a 6f  |gate.h./Users/jo|
000001c0  65 79 2e 63 61 6f 2f 44  65 73 6b 74 6f 70 2f 4c  |ey.cao/Desktop/L|
000001d0  65 61 72 6e 69 6e 67 2f  4c 4c 56 4d 2f 64 79 6c  |earning/LLVM/dyl|
000001e0  64 2f 44 65 6d 6f 32 2f  54 65 73 74 4c 69 6e 6b  |d/Demo2/TestLink|
000001f0  2f 54 65 73 74 4c 69 6e  6b 2f 00 53 63 65 6e 65  |/TestLink/.Scene|
00000200  44 65 6c 65 67 61 74 65  2e 68 00 56 69 65 77 43  |Delegate.h.ViewC|
00000210  6f 6e 74 72 6f 6c 6c 65  72 2e 68 00              |ontroller.h.|
```

可以看到这里把所有的头文件，和它所属的文件夹全部列了出来。 并且这里还做了一些优化，用一个头文件后面来接它所属的文件夹层级，其他的同级文件直接跟在文件夹目录后面，这样同级的文件就不用再加文件夹路径了。否则如果每个头文件都接一个对应路径这个hmap文件会爆炸，而且非常不利于路径的查找。

<b>由于这个hmap，只要是工程根目录的头文件都可以通过hmap查找到指定的路径，因此我们再编写iOS代码，只需要直接`#import ""`即可直接引入对应的头文件。关于Header map的实现可以参考Clang[官方文档](https://clang.llvm.org/doxygen/classclang_1_1HeaderMap.html)。</b>

有意思的是，如果我们仅仅在主工程添加另一个Header File的引用，同样会在Hmap中添加相应的搜索路径。因此我们可以把在工程中嵌套工程，并通过直接拉文件引用的方式，来直接调用另一个工程的功能。但是这样有两个问题：1.必须保证两一个工程产生是的Mach-O文件link到主工程，否则会找不到符号。2.这样会导致工程及其混乱，一层套一层，而且由于import是一个递归的过程，因此直接引用的头文件不能import其它不可见的头文件，否则仍然编译不过。

---

@class 

另一种引用方式是`@class XXX.h`。这种引用不会把头文件引入，它只是一种前向声明，只是告诉编译器有这么一个类，然后就可以在代码中使用引用这个类了，但是能知道的仅仅是这个类名。更神奇的是由于只是作为一个符号引用，编译器根本不会对它做任何检查，即使是一个根本不存在类一样可以。 

这种引入的好处：1是可以避免头文件引入混乱，比如A，B两个文件互相引用了对方，并在代码中使用到了对方，则会导致编译器提示某个文件不存在，因为这时候互相依赖导致编译器不知道要先编译哪个对象，此时两个源文件都没有目标文件。2：减少预处理时间，比如A.h中有一个`#import B.h`，则如果C.h文件import了A.h就会潜在的把B.h也import进来，而@class 仍然只是插入一个前向声明而已。 

<b>这里涉及到我们的一个编程规范，尽量减少暴露头文件，尽量在头文件中使用@class，而在.m中才#import头文件。对于有些不想Public出去的属性和方法，可以利用extension来处理，例如拉出一个A_Private.h来提供某些类使用</b>>

---

@import Framework

还有一种比较少见的引用方式为`@import framework`。首先可以首先可以看下`Build Setting`里有个`Link`Frameworks Automatically 默认是Yes的，这也是为啥我们把一个framework导入工程时会自动在`Build Phases`中进行Link，而`Enable Modules`选项可以允许我们通过@import来引入framework。

![image-20191127200736984](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/谈谈iOS的编译链接/image2.png?raw=true)

根据[WWDC2013](https://developer.apple.com/videos/wwdc2013/)的描述，利用这种比`#import`更加安全效率更高，因为对于import仍然还是简单的递归的拷贝头文件，而`@import`使用把framework作为modules的方式进行自动链接，仅仅在代码中真正使用了对应的`framework`中的文件时才会进行import。而且对于第三方也可以通过这种方式进行引入，当前仅仅引入才会触发自动动态链接。

关于Modules是什么可以通过[这篇文章](https://samsymons.com/blog/understanding-objective-c-modules/)进行了解。我们可以在动态库的build文件夹中找到`module.modulemap`。在通过Xcode构建动态库时会自动打开`Defines Module`,自动生成modulemap文件，描述动态库的结构和一个`umbrella header`,只需要在`umbrella header`中import其他公开头文件即可，并且如果有遗漏的，编译器还会提示warning。可见苹果推荐的还是在`umbrella header`。

```shell
$ cat /Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Intermediates.noindex/Function1.build/Debug-iphonesimulator/Function1.build/module.modulemap
framework module Function1 {
  umbrella header "Function1.h"

  export *
  module * { export * }
}
```

关于它是怎么实现的，仍然是类似`Hmap`一样的优化，通过特定的文件把把所有的framework的头文件分成一个个module写入ModuleCache.noidex文件下。打开DerivedData文件下`ModuleCache.noindex`。这里存放都是预编译module文件，可以看到工程中用到的系统的标准库文件，包括引入DJISDK.framework也有一个对应的.pcm文件。虽然它们是data文件，比较幸运的是，我们用`hexdump`竟然可以看到其中的一些字符：

```shell
$ hexdump -C /DerivedData/ModuleCache.noindex/3SRZO6DXX9MMT/DJISDK-LFXKH73HE9T0.pcm
//只截取部分显示
001485a0  00 00 00 00 00 6c 44 01  00 c4 a5 d0 18 04 00 16  |.....lD.........|
001485b0  00 64 69 64 55 70 64 61  74 65 54 61 70 5a 6f 6f  |.didUpdateTapZoo|
001485c0  6d 53 74 61 74 65 00 80  98 01 00 c4 e5 ff b4 04  |mState..........|
001485d0  00 2b 00 67 65 74 4d 75  6c 74 69 73 70 65 63 74  |.+.getMultispect|
001485e0  72 61 6c 53 68 75 74 74  65 72 53 70 65 65 64 57  |ralShutterSpeedW|
001485f0  69 74 68 43 6f 6d 70 6c  65 74 69 6f 6e 00 38 9b  |ithCompletion.8.|
00148600  01 00 01 00 c8 e5 32 04  04 00 0a 00 69 6e 70 75  |......2.....inpu|
00148610  74 48 69 6e 74 00 fe 87  01 00 01 00 c9 e5 6a 4c  |tHint.........jL|
00148620  0c 00 11 00 4e 53 4d 75  74 61 62 6c 65 43 6f 70  |....NSMutableCop|
00148630  79 69 6e 67 00 a7 ab 00  00 00 00 00 00 b5 66 00  |ying..........f.|
00148640  00 01 00 ca 25 a6 02 0c  00 1b 00 44 4a 49 4c 69  |....%......DJILi|
00148650  67 68 74 62 72 69 64 67  65 4c 69 6e 6b 44 65 6c  |ghtbridgeLinkDel|
00148660  65 67 61 74 65 00 d3 87  01 00 00 00 00 00 58 38  |egate.........X8|
```

至少可以看到它内部存储了Framework的所有public头文件，后面应该是Framework的二进制文件。那么@import module时就可以找到指定的头文件。而且由于这些文件放在Derived文件下，因此多次编译都会公用这些.pcm文件，而提高预编译的效率。同样在`Intermediates.noindex`中也会存放一些编译的中间产物，比如hmap，目标文件文件，目标文件的clang诊断信息等以提供重复使用。

<b>正是由于编译cache，clang才能做到快速的增量编译，仅仅编译发生修改的文件，但是当一个头文件被修改时所有import它或者间接import它的文件都需要重新编译。因此尽量减少在头文件进行修改，尽可能少的在头文件暴露信息，或是利用extension创建Private头文件减少import的数量，可以有效减少编译时间。</b>

#### 库文件引用

我们在工程中创建一个Frameworks文件，然后拖入一个framework。然后可以发现`Link Binary With Libraries`中多出了这个framework，`Frameworks Search Paths`中也自动添加了这个Frameworks目录的路径。这时我们就可以直接引用库里的头文件。但是当我们想运行时，咔，会报错`image not found`

```shell
Referenced from: /Users/joey.cao/Library/Developer/CoreSimulator/Devices/235EABB7-445A-4D9E-A268-EDB7ADD28846/data/Containers/Bundle/Application/5B95AE34-9A8D-4DB8-97F8-A4579D71759B/TestLink.app/TestLink
  Reason: image not found
```

原因看起来很简单找不到引入的DJISDK这个库文件，但是我们命名已经Link了这个动态库啊。由于DJISDK.framework是动态库，只有在代码中使用到它才会在运行时通过动态链接器dyld进行动态链接。我们打开生成程序bundle，看到它的内部并没有DJISDK.framework，而运行时程序访问的只有xxx.app这个bundle里的资源和系统标准库中的资源，所以由于动态库根本没有拷贝进程序bundle,所以查找不到。此时只需要把第三方引入的动态库选中`Embed and Sign`即可。<b>这是个很重要的点，尤其是在使用Cmake等工具进行OC工程的构建时一定要注意</b>>

当我们Embed之后，再次build可以看到，首先会在.app这个bundle下构建一个frameworks的文件，在编译完成之后会把embed的库文件拷贝过去，并进行重签名，正是由于签名机制导致不能在运行时动态的加载包含代码的动态库，但是可以加载只含有资源的bundle文件。动态库在运行时使用到动态库时才进行链接，因此不会出现编译时的符号冲突。<b>由于动态库不参与编译，所以有改动时不需要引用的工程重新编译。当然缺点就是可能会产生可怕的运行时崩溃，因此一定要控制源代码和依赖的动态库的库版本对齐</b>。

库文件的搜索路径在`Frameworks Search Paths`中，当出现库文件找不到时，可以检查是不是路径没有添加。

当提示库文件里`Undefined Symbols`问题，如果确保库文件已经embed，应该检查一下库文件的`Fat file`或者`thin file`是否当前运行环境的架构。

如果库文件工程源码在同一个workspace下,或者嵌在同一个工程中，我们甚至还可以通过修改`header search path`或者直接添加一个头文件或头文件引用的方式直接调用库里没有Public的代码。

<span id="answer3">现在回到问题3：</span>

如果同时使用`#import <XXXFramework/AAA.h>`和`#import BBB.h`会导致问题吗？答案是，会导致重复定义。

样例如下，Function1和Method1都是Function1这个framework下的文件，通过直接拉头文件引用来`#import Method1.h `,通过库引用得到方式来引用Function1:

![image-20191201230430748](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/谈谈iOS的编译链接/image3.png?raw=true)

结果编译会出错，重复定义了Method1这个类，为了避免因为`Function1.h`是`Umbrella header`的原因，我们改成另一个header，发现依然还是一样的问题：

![image-20191201230650759](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/谈谈iOS的编译链接/image4.png?raw=true)

首先这种引用方式当然是不合理的，<b>建议要么都用 "", 要么都用<></b>。但是奇怪的是如果我们把`#import Method1.h`这一句放在上面就不会报错，这里我们首先要理解在`#import`的时候Clang做了什么，由于Function1是一个动态库，所以在构建时创建了Modules。当我们通过`import <Function1/xxx.h>`方式import时就会直接把module引入。而在`#import "Method1.h"`只是添加hmap并查找这个符号。作为证据我们删除掉DerviedData，并删掉`import <Function1/xxx.h>`这一句。重新编译，可以看到ModuleCache里已经没有Function1了。

![image-20191201234223020](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/谈谈iOS的编译链接/image5.png?raw=true)

而当我们把`import <Function1/xxx.h>`加上时在编译：

![image-20191201234346827](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/谈谈iOS的编译链接/image6.png?raw=true)

可见只有`#import <>`的方式才会产生modulecache。因此当再次`import ""`时，由于modulecache中已经把Framework的二进制文件缓存起来了，因此提示重复定义。而为什么`import ""`放在前面时不会提示这个错误，这个我目前还不知道，只能猜测ModuleCache在链入时跟静态库链接一样，出现重复强符号，只采用首个出现的。

经过分析其实是ModuleCache导致的，所以把`Defines Modules`关掉就可以解决，但是这并不是真正的解决之道。

这一切得原因都是import方式不规范导致，只要规范下import方式即可解决。

### iOS的链接

#### 库文件的编译

当一个静态库有两个相同命名的类是，是不会出现`duplicate symbol`的，我们创建一个`Method3`文件和一个`Method3_Copy`文件，两者内容是一模一样的，看一下静态库文件编译过程，如下，可以发现它只是通过`libtool`把`Link file list`中的文件进行了拷贝进行了生成.a文件：

![image-20191201150831370](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/谈谈iOS的编译链接/image7.png?raw=true)

我们通过ar工具查看生成的静态库的成员:

```shell
$ ar -v -t /Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Products/Debug-iphonesimulator/libFunction3.a
rw-r--r--       0/0           208 Jan  1 08:00 1970 __.SYMDEF
rw-r--r--       0/0          5064 Jan  1 08:00 1970 Method3.o
rw-r--r--       0/0          5072 Jan  1 08:00 1970 Method3_Copy.o
rw-r--r--       0/0          5112 Jan  1 08:00 1970 Function3.o
```

可以看到内部仅仅只有一个符号表和直接拷贝进来的源码编译出的目标文件。然后我们查看.a文件的符号如下：

```shell
nm -nm /Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Products/Debug-iphonesimulator/libFunction3.a

/Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Products/Debug-iphonesimulator/libFunction3.a(Method3.o):
                 (undefined) external _NSLog
                 (undefined) external _OBJC_CLASS_$_NSObject
                 (undefined) external _OBJC_METACLASS_$_NSObject
                 (undefined) external ___CFConstantStringClassReference
                 (undefined) external __objc_empty_cache
0000000000000000 (__TEXT,__text) non-external +[Method3 log]
0000000000000080 (__DATA,__objc_const) non-external l_OBJC_$_CLASS_METHODS_Method3
00000000000000a0 (__DATA,__objc_const) non-external l_OBJC_METACLASS_RO_$_Method3
00000000000000e8 (__DATA,__objc_const) non-external l_OBJC_CLASS_RO_$_Method3
0000000000000130 (__DATA,__objc_data) external _OBJC_METACLASS_$_Method3
0000000000000158 (__DATA,__objc_data) external _OBJC_CLASS_$_Method3

/Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Products/Debug-iphonesimulator/libFunction3.a(Method3_Copy.o):
                 (undefined) external _NSLog
                 (undefined) external _OBJC_CLASS_$_NSObject
                 (undefined) external _OBJC_METACLASS_$_NSObject
                 (undefined) external ___CFConstantStringClassReference
                 (undefined) external __objc_empty_cache
0000000000000000 (__TEXT,__text) non-external +[Method3 log]
0000000000000080 (__DATA,__objc_const) non-external l_OBJC_$_CLASS_METHODS_Method3
00000000000000a0 (__DATA,__objc_const) non-external l_OBJC_METACLASS_RO_$_Method3
00000000000000e8 (__DATA,__objc_const) non-external l_OBJC_CLASS_RO_$_Method3
0000000000000130 (__DATA,__objc_data) external _OBJC_METACLASS_$_Method3
0000000000000158 (__DATA,__objc_data) external _OBJC_CLASS_$_Method3

/Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Products/Debug-iphonesimulator/libFunction3.a(Function3.o):
                 (undefined) external _NSLog
                 (undefined) external _OBJC_CLASS_$_NSObject
                 (undefined) external _OBJC_METACLASS_$_NSObject
                 (undefined) external ___CFConstantStringClassReference
                 (undefined) external __objc_empty_cache
0000000000000000 (__TEXT,__text) non-external +[Function3 method3]
0000000000000090 (__DATA,__objc_const) non-external l_OBJC_$_CLASS_METHODS_Function3
00000000000000b0 (__DATA,__objc_const) non-external l_OBJC_METACLASS_RO_$_Function3
00000000000000f8 (__DATA,__objc_const) non-external l_OBJC_CLASS_RO_$_Function3
0000000000000140 (__DATA,__objc_data) external _OBJC_METACLASS_$_Function3
0000000000000168 (__DATA,__objc_data) external _OBJC_CLASS_$_Function3
```

可以看到符号表也是分开的，即没有进行链接操作，所以不会提示`duplicate symbols`问题。当它被link到一个项目时，会重新拷贝内部这些目标文件并进行链接，可以理解把它再内嵌到目标项目中的。所以一旦静态库发生改变，目标项目也必须重新进行编译。同样的由于静态库不会进行链接，所以引用一些标准库等时还需要主项目手动添加这些库的连接，否则会导致`undefined symbols`。

而动态库的编译过程如下,可以看到它确实对所有目标文件进行了链接，在符号表重组时，直接发现了`duplicate symbols`的问题:

![image-20191201151150923](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/谈谈iOS的编译链接/image8.png?raw=true)

所以动态库其实是对内部的目标文件进行了链接，形成了一个整体在运行时通过动态连接器链接到目标项目中的，因此动态库重新编译并不会导致目标项目重新编译。

那么当我们把静态库链接到可执行文件中时会出现`duplicate symbols`错误码，答案仍然是不会。并且发现我们再调用`Mehtod3`的方法时，执行的是`Method3`这个类里实现的方法。但是当我们调整一下`Function3`这个静态库里编译文件的顺序，把`Method3_Copy`放到前面时，可以看到执行的变成了`Method3_Copy`里的方法。此时我们查看一下`Function3`这个镜头库的构成:

```shell
$ nm -nm /Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Products/Debug-iphonesimulator/libFunction3.a

/Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Products/Debug-iphonesimulator/libFunction3.a(Method3_Copy.o):
...

/Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Products/Debug-iphonesimulator/libFunction3.a(Method3.o):
...

/Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Products/Debug-iphonesimulator/libFunction3.a(Function3.o):
...
```

所以可以推断，编译文件时会按顺序写入`LinkFileList`，而静态库依据`LinkFileList`把目标文件拷贝进静态库，再进行和主项目的链接。而出现重复符号时只会依据第一个强符号来作为唯一符号进行处理。所以才会出现这样的问题。

#### 库文件的链接

<span id="answer6">现在回到问题6:</span>

根据上面的结论可以知道当Clang在链接库文件时遇到主项目和库文件符号出现重复或者库本身有重复符号时，不会出现`duplicate symbols`错误，而是根据遇到的第一个强符号来处理，并且由于优先编译主工程内部文件，因此主工程中的符号将会优先被加入到全局符号表中，因此执行出来的反而是主工程的方法。

我们在主项目中创建一个链接的静态库有相同类`Method3`,然后运行，结果正常运行，但是在通过输出可以看到，正在执行的却是我们新添加的`Method3`中的方法。

```shell
objc[9953]: Class Method1 is implemented in both /Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/DerivedData/MainProject/Build/Products/Debug-iphonesimulator/Function1.framework/Function1 (0x1077fd100) and /Users/joey.cao/Library/Developer/CoreSimulator/Devices/235EABB7-445A-4D9E-A268-EDB7ADD28846/data/Containers/Bundle/Application/073E9CF2-4B5C-4C71-819C-B11196B76B09/TestLink.app/TestLink (0x1074e38b8). One of the two will be used. Which one is undefined.
2019-12-01 16:03:19.616457+0800 TestLink[9953:9185455] 主项目的 sum
2019-12-01 16:03:19.616564+0800 TestLink[9953:9185455] ---+[Function3 method3]
2019-12-01 16:03:19.616667+0800 TestLink[9953:9185455] 主工程 method3
```

好在Xcode帮我们指出了有重复方法声明，但是可惜的是通过它并不能定位出到底是哪个方法发生重复了。更可怕的是当库文件本身发生符号冲突时，十一点警告都没有。那么当我们引用的第三方库中符号发生冲突，则可能导致方法执行错误而我们不自知的情况，尤其是不同的第三方库使用同一个库的不同版本，可能导致运行时奇怪的崩溃，甚至完全代码跑偏。

如果这些库文件都是开源的，那么还可以通过脚本对依赖的库进行比对，从而找到不同的地方，并根据实际情况进行选择。比如之前MSDK在进行跨平台开发时，发现有一些文件在Linux系统和安卓系统上时不一样的，这是只能通过预编译宏进行控制，更麻烦的可能还需要手动添加一些实现来弥补平台的缺失。

而对于闭源的第三方库，我们知道.a只是一个目标文件集合，因此我们可以通过ar把指定的不兼容的目标文件删除掉即可。具体实现可以参照[这篇文章](http://atnan.com/blog/2012/01/12/avoiding-duplicate-symbol-errors-during-linking-by-removing-classes-from-static-libraries)。

为了避免这些问题，所以作为SDK开发时，或者封装库文件时一定要指定好依赖的第三方库文件的版本。或是通过给给文件加前缀来避免符号重复，就好像加了命名空间一样。当然还可以把依赖的库文件分离出来，让用户进行引入，例如`DJIWidget`中使用`FFMpeg`，避免多份库文件的存在。

#### 库文件多层链接

<span id="answer8">回到问题8：</span>

库文件可以多层嵌套吗，答案是可以的。

我们测试如下，主工程Link一个动/静态库，静态库再Link一个静态库，静态库再Link一个静态库。然后依次链式调用下一级静态库里的方法，可以发现可以正常调用。

由于的工程是4个工程平铺的，甚至可以在任意一个层级的工程中拉一个另一个低层级库中头文件的引用，以实现直接跨层级的调用。

```shell
2019-12-01 20:42:24.097954+0800 TestLink[43537:9645427] __+[Method1 sum:b:]
2019-12-01 20:42:24.098097+0800 TestLink[43537:9645427] __+[Method2 log:]
2019-12-01 20:42:24.098210+0800 TestLink[43537:9645427] Method3--在function2 中使用 function3
2019-12-01 20:42:24.098316+0800 TestLink[43537:9645427] --+[Function4 method4]
2019-12-01 20:42:24.098420+0800 TestLink[43537:9645427] 在层级1的动态库中直接调用层级3的静态库中的方法
2019-12-01 20:42:24.098520+0800 TestLink[43537:9645427] Method3--11
2019-12-01 20:42:24.098601+0800 TestLink[43537:9645427] --+[Function4 method4]
```

上面在中文log之前，是在主工程依次按层级调用到了四层Link的库。之后是在第一层的动态库直接调用第3层的动态库。

<b>所以无论link多少层静态库，本质上都是在运行时把所有的目标文件的符号整合到一起，通过符号找到对应的机器指令，因此只要符号找得到就能调用到。</b>

但是问题又来了，能否从层级3或者4调用到层级2中方法？其实是不可以的，因为这样会导致互相依赖。

![image-20191201205329591](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/谈谈iOS的编译链接/image9.png?raw=true)

#### 链接选项

<span id="answer9">回到问题9：</span>

在`Build Setting`中有一个选项为`Other Link Flag`,很多人碰到过引用第三方库中有Category时发生`Unrecongnized Selector`崩溃，都知道可以通过添加`-Objc`来进行解决，那么这里到底是干了什么呢？

查看编译log可以看到，添加`-Objc`之后会在为Clang添加`-Objc`这个编译选项，同理添加`-all_load`, `-force_load`都只是链接器添加一个选项而已。因为编译器为了避免生成的可执行文件太大，默认只连接Class，c, C++相关的目标文件，而Category和一些未使用的文件符号都会被Strip掉。因此第三方库虽然存在Category的符号，但是链接时被Strip掉了，因此会出现`Unrecongnized Selector`。而`-all_load`则是把库文件所有的目标文件以及所有依赖的第三方库全部链接，比较常见的场景是使用静态库时，发现找不到符号，需要添加一些第三方库，网上有些回答添加`-all_load`。当然这样可以解决，但是带来的问题也很明显，它会导致所有的库文件都执行这样的操作，会导致可执行文件的增大，并且如果两个静态库中的目标文件有相同的符号就会导致重复符号错误。另一种解决方法就是`-force_load`，指定某一个第三方库进行all_load。 <b>这里还有一个重要的应用场景是， 动态库会丢弃内部没有使用的其它静态库文件或者是静态库文件的Object对象，可能会导致运行时崩溃，为了解决就得使其all_load</b>

还有一个选项`-dead_strip`，它可以把使用不到的代码，block，以及目标文件进行strip，可以参见苹果的[release note](https://opensource.apple.com/source/cctools/cctools-622.5.1/RelNotes/CompilerTools.html?txt)。

#### 模块化管理工程

工程变庞大之后，可以考虑按功能模块，或者架构层级把子模块编译成库文件，然后让主工程依赖它，以根据需求按需加载模块。而且可以针对子模块的库文件构建独立的单元测试项或是测试界面。

一种解决方式是通过脚本来进行管理工程，通过脚本独立的仓库依次拉下来，然后构建一个workspace,再手动去添加依赖关系。然后在主工程中有一个位置存放各个版本依赖的各个仓库的分支或者是标签。只需要根据切换指定版本之后，在依据保存的此时的各个仓库情况利用脚本一次仓库切换分支即可实现模块化的管理。

另一种方式是利用cocoapods进行维护。cocoapods可以帮我们拉取代码，并创建workspace，只需要构建合适的Podfile即可利用cocoapods帮我们管理依赖关系。

## OC类与分类的编译

<span id="answer4">这里我们先回到问题4, 5：</span>

1.如果编译时一个Class的文件没有被加入到指定的target，则只要这个文件被别的地方引用了一定会导致编译不过，因为符号找不到。但是如果是Category的话则可以正常编过。

2.如果一个方法没有被定义，即使被被其他地方调用，仍然可以正常编译，只是会在运行时崩溃。

我们以一个例子来分析为什么：

 首先我们只为test1添加定义。build一下之后去derivedData中`/Build/Intermediates.noindex/TestLink.build/Debug-iphonesimulator/TestLink.build/Objects-normal/x86_64`文件就下寻找编译的中间产物：

首先我们查看Test.o的符号：

```shell
$ nm -nm /Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/TestLink/DerivedData/TestLink/Build/Intermediates.noindex/TestLink.build/Debug-iphonesimulator/TestLink.build/Objects-normal/x86_64/Test.o
                 (undefined) external _NSLog
                 (undefined) external _OBJC_CLASS_$_NSObject
                 (undefined) external _OBJC_METACLASS_$_NSObject
                 (undefined) external ___CFConstantStringClassReference
                 (undefined) external __objc_empty_cache
                 (undefined) external _objc_storeStrong
0000000000000000 (__TEXT,__text) non-external +[Test test1]
0000000000000030 (__TEXT,__text) non-external -[Test name]
0000000000000050 (__TEXT,__text) non-external -[Test setName:]
0000000000000090 (__TEXT,__text) non-external -[Test age]
00000000000000b0 (__TEXT,__text) non-external -[Test setAge:]
00000000000000d0 (__TEXT,__text) non-external -[Test .cxx_destruct]
00000000000001e0 (__DATA,__objc_const) non-external l_OBJC_$_CLASS_METHODS_Test
0000000000000200 (__DATA,__objc_const) non-external l_OBJC_METACLASS_RO_$_Test
0000000000000248 (__DATA,__objc_const) non-external l_OBJC_$_INSTANCE_METHODS_Test
00000000000002c8 (__DATA,__objc_const) non-external l_OBJC_$_INSTANCE_VARIABLES_Test
0000000000000310 (__DATA,__objc_const) non-external l_OBJC_$_PROP_LIST_Test
0000000000000338 (__DATA,__objc_const) non-external l_OBJC_CLASS_RO_$_Test
0000000000000380 (__DATA,__objc_data) external _OBJC_METACLASS_$_Test
00000000000003a8 (__DATA,__objc_data) external _OBJC_CLASS_$_Test
00000000000003d0 (__DATA,__objc_ivar) private external _OBJC_IVAR_$_Test._age
00000000000003d8 (__DATA,__objc_ivar) private external _OBJC_IVAR_$_Test._name
```

不意外数据段存放了class,metaclass和私有的Ivar。 而程序段也有Property自动生成的getter和setter。<b>但是需要注意的是：这些方法并不是external的，前面我们说了external表示的是其可见性，这里表明了这些方法其实外部是并不可见的。说明在链接的时候并不会去检查符号表，因此即使没有为方法添加定义仍然可以正常编译</b>。

但是class文件仍然是external的，因此如果被外部引用到了，但是没有参与编译直接会报错 `Undefined symbols _OBJC_CLASS_$_Test`。而我们查看一下Test的分类的符号：

```shell
$ nm -nm /Users/joey.cao/Desktop/Learning/LLVM/dyld/Demo2/TestLink/DerivedData/TestLink/Build/Intermediates.noindex/TestLink.build/Debug-iphonesimulator/TestLink.build/Objects-normal/x86_64/Test+test.o
                 (undefined) external _NSLog
                 (undefined) external _OBJC_CLASS_$_Test
                 (undefined) external ___CFConstantStringClassReference
0000000000000000 (__TEXT,__text) non-external +[Test(test) test3]
00000000000000a0 (__DATA,__objc_const) non-external l_OBJC_$_CATEGORY_CLASS_METHODS_Test_$_test
00000000000000c0 (__DATA,__objc_const) non-external l_OBJC_$_PROP_LIST_Test_$_test
00000000000000e8 (__DATA,__objc_const) non-external l_OBJC_$_CATEGORY_Test_$_test
```

可以惊奇的发现它是没有类符号的，分类并不是类，所以即使import了Category， 并调用了其中的方法，也不会导致编译不过，因为他没有一个符号是外部可见的。同样可以看到添加的属性都生成了Properlist，但是却没有生成对应Ivars, 也没有getter和setter。因此我们可以分类添加的property属性列表里看到有这些属性，但是无法get, set。

而这一切都是由于OC的Runtime机制，它只会对类名符号进行链接，而所有的method都是`non-external`的，只是保存在`method_list`中，通过`msg_send`进行调用。

关于编译出的Mach-O文件具体代表什么，可以参考这边[文章](https://satanwoo.github.io/2017/06/29/Macho-2/)。

## 参考资料

1.程序员的自我修养（https://item.jd.com/10067200.html）

2.https://opensource.apple.com/source/cctools/cctools-622.5.1/RelNotes/CompilerTools.html?txt

3.https://samsymons.com/blog/understanding-objective-c-modules/

4.https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/XcodeBuildSettingRef/1-Build_Setting_Reference/build_setting_ref.html#//apple_ref/doc/uid/TP40003931-CH3-SW61

5.https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/DynamicLibraries/000-Introduction/Introduction.html#//apple_ref/doc/uid/TP40001869

6.https://forums.developer.apple.com/thread/98369