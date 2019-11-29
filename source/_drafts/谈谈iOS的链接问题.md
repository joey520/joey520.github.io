---

toc: true
title: 谈谈iOS的链接问题
date: 2019-11-23 17:29:33
categories: iOS
tags: [LLVM]
---

## 前言

首先抛出几个个问题：

1.什么是`header serach path`,什么又是`framework search path`？

2.#import, #include, 和@class的区别是什么。循环#import一定会导致编译失败吗？

3.如果编译时一个Class的文件没有被加入到指定的target的complie files里会导致什么问题，如果是一个Category的文件呢？

4.如果一个类里的方法只有声明没有定义，则编译时会出现`undefinde symbol`吗, 为什么？

### 什么是链接

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

----

### iOS的编译

### 搜索路径

#### 头文件引用

clang在进行预处理时会把import的头文件引入并展开，这样就可以在当前文件中使用这些声明的方法了。而import的时候只是传入了一个字符串，编译器是如何找到对应的头文件的呢，那就得依靠搜索路径。

Clang编译器具有和GCC相同[搜索指令](http://gcc.gnu.org/onlinedocs/cpp/Search-Path.html)。对于`#include ""`修饰首先在当前文件夹内查找，然后在自定义的路径中进行搜索。对于`#include <>`修饰的首先在系统标准库中进行搜索，然后是自定义的路径。

但是在使用iOS工程时我们发现只要是在工程的根目录文件下的文件都可以直接通过`#include ""`到，而不需要像C，C++里面一下指定详细的相对路径。这是因为<b>hmap</b>这个东西。默认xcode的build setting里有一个选项是`Uses Headers Map`打开的。我们build一下当前的工程并导出编译log可以看到:

![image-20191127111720240](/Users/joey.cao/Library/Application Support/typora-user-images/image-20191127111720240.png)

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



但是即使有Hmap进行快速的搜索，在递归查找时

---

@class 

另一种引用方式是`@class XXX.h`。这种引用不会把头文件引入，它只是一种前向声明，只是告诉编译器有这么一个类，然后就可以在代码中使用引用这个类了，但是能知道的仅仅是这个类名。更神奇的是由于只是作为一个符号引用，编译器根本不会对它做任何检查，即使是一个根本不存在类一样可以。 

这种引入的好处：1是可以避免头文件引入混乱，比如A，B两个文件互相引用了对方，并在代码中使用到了对方，则会导致编译器提示某个文件不存在，因为这时候互相依赖导致编译器不知道要先编译哪个对象，此时两个源文件都没有目标文件。2：减少预处理时间，比如A.h中有一个`#import B.h`，则如果C.h文件import了A.h就会潜在的把B.h也import进来，而@class 仍然只是插入一个前向声明而已。 

<b>这里涉及到我们的一个编程规范，尽量减少暴露头文件，尽量在头文件中使用@class，而在.m中才#import头文件。对于有些不想Public出去的属性和方法，可以利用extension来处理，例如拉出一个A_Private.h来提供某些类使用</b>>

---

@import Framework

还有一种比较少见的引用方式为`@import framework`。首先可以首先可以看下`Build Setting`里有个`Link`Frameworks Automatically 默认是Yes的，这也是为啥我们把一个framework导入工程时会自动在`Build Phases`中进行Link，而`Enable Modules`选项可以允许我们通过@import来引入framework。

![image-20191127200736984](/Users/joey.cao/Library/Application Support/typora-user-images/image-20191127200736984.png)

根据[WWDC2013](https://developer.apple.com/videos/wwdc2013/)的描述，利用这种比`#import`更加安全效率更高，因为对于import仍然还是简单的递归的拷贝头文件，而`@import`使用把framework作为modules的方式进行自动链接，仅仅在代码中真正使用了对应的`framework`中的文件时才会进行import。而且对于第三方也可以通过这种方式进行引入，当前仅仅引入才会触发自动动态链接。

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

至少可以看到它内部存储了Framework的所有public头文件，那么@import module时就可以找到指定的头文件。而且由于这些文件放在Derived文件下，因此多次编译都会公用这些.pcm文件，而提高预编译的效率。同样在`Intermediates.noindex`中也会存放一些编译的中间产物，比如hmap，目标文件文件，目标文件的clang诊断信息等以提供重复使用。

<b>正是由于编译cache，clang才能做到快速的增量编译，仅仅编译发生修改的文件，但是当一个头文件被修改时所有import它或者间接import它的文件都需要重新编译。因此尽量减少在头文件进行修改，尽可能少的在头文件暴露信息，或是利用extension创建Private头文件减少import的数量，可以有效减少编译时间，提高工作效率。</b>



#### 库文件引用

我们在工程中创建一个Frameworks文件，然后拖入一个framework。然后可以发现`Link Binary With Libraries`中多出了这个framework，`Frameworks Search Paths`中也自动添加了这个Frameworks目录的路径。这时我们就可以直接引用库里的头文件。但是当我们想运行时，咔，会报错`image not found`

```shell
Referenced from: /Users/joey.cao/Library/Developer/CoreSimulator/Devices/235EABB7-445A-4D9E-A268-EDB7ADD28846/data/Containers/Bundle/Application/5B95AE34-9A8D-4DB8-97F8-A4579D71759B/TestLink.app/TestLink
  Reason: image not found
```

原因看起来很简单找不到引入的DJISDK这个库文件，但是我们命名已经Link了这个动态库啊。由于DJISDK.framework是动态库，只有在代码中使用到它才会在运行时通过动态链接器dyld进行动态链接。我们打开生成程序bundle，看到它的内部并没有DJISDK.framework，而运行时程序访问的只有xxx.app这个bundle里的资源和系统标准库中的资源，所以由于动态库根本没有拷贝进程序bundle,所以查找不到。此时只需要把第三方引入的动态库选中`Embed and Sign`即可。<b>这是个很重要的点，尤其是在使用Cmake等工具进行OC工程的构建时一定要注意</b>>

当我们Embed之后，再次build可以看到，首先会在.app这个bundle下构建一个frameworks的文件，在编译完成之后会把embed的库文件拷贝过去，并进行重签名，正是由于签名机制导致不能在运行时动态的加载包含代码的动态库，但是可以加载只含有资源的bundle文件。动态库在运行时使用到动态库时才进行链接，因此不会出现链接时的符号冲突。<b>由于动态库不参与编译，所以有改动时不需要引用的工程重新编译。当然缺点就是可能会产生可怕的运行时崩溃，因此一定要控制源代码和依赖的动态库的库版本对齐</b>。

库文件的搜索路径在`Frameworks Search Paths`中，当出现库文件找不到时，可以检查是不是路径没有添加。

当提示库文件里`Undefined Symbols`问题，首先应该检查一下库文件的`Fat file`或者`thin file`是否当前运行环境的架构。

我们可以在工程中link另一个工程产生的库文件，但是要注意添加依赖关系，保证库文件先被生产出来。

我们还可以考虑通过静态库来隐藏代码，比如把某些不需要暴露的静态库内嵌在其他库文件提供给用户。

#### 模块化管理工程

工程变庞大之后，可以考虑按功能模块，或者架构层级把子模块编译成库文件，然后让主工程依赖它，以根据需求按需加载模块。而且可以针对子模块的库文件构建独立的单元测试项或是测试界面。

一种解决方式是通过脚本来进行管理工程，通过脚本独立的仓库依次拉下来，然后构建一个workspace,再手动去添加依赖关系。然后在主工程中有一个位置存放各个版本依赖的各个仓库的分支或者是标签。只需要根据切换指定版本之后，在依据保存的此时的各个仓库情况利用脚本一次仓库切换分支即可实现模块化的管理。

另一种方式是利用cocoapods进行维护。cocoapods可以帮我们拉取代码，并创建workspace，只需要构建合适的Podfile即可利用cocoapods帮我们管理依赖关系。

### 类与分类

这里我们先回到文章开头的问题：

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

不意外数据段存放了class,metaclass和私有的Ivar。 而程序段也有Property自动生成的getter和setter。<b>但是需要注意的是：这些方法并不是external的，前面我们说了external表示的是其可见性，这里表明了这些方法其实外部是并不可见的。说明在链接的时候并不会去检查符号表，因此即使没有为方法添加定义仍然可以正常编译</b>b>。

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

可以惊奇的发现它是没有类符号的，分类并不是类，所以即使import了Category， 并调用了其中的方法，也不会导致编译不过，因为他没有一个符号是外部可见的。

同样可以看到添加的属性都生成了Properlist，但是却没有生成对应Ivars, 也没有getter和setter。因此我们可以看到分类添加的property属性列表里有这些属性，但是无法获取。

为了更方便查看我们可以采用[MachOView](https://sourceforge.net/projects/machoview/)这个软件，它其实是把objdump的输出转成了图形界面显示。首先打开Test.o:

发现runtime里所讲的那些概念全部在Mach-O文件中找到了对应的section。我们只关注几个section：

1.类数据段__objc_data

![image-20191126230101279](/Users/joey.cao/Library/Application Support/typora-user-images/image-20191126230101279.png)

对Class进行了描述，可以看到ISA指针的指向的继承链。以及Class数据指向R0寄存器。

2.