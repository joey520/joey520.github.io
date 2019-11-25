---

toc: true
title: 谈谈iOS的链接问题
date: 2019-11-23 17:29:33
categories: iOS
tags: [LLVM]
---

## 前言

首先抛出几个个问题：

1.如果编译时一个Class的文件没有被加入到指定的target会导致什么问题，如果是一个Category的文件呢？

2.如果一个类的方法只有声明没有定义，则编译时会出现问题吗？

2.如果编译的文件Import了一个非Public的framework会怎么样？

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

### iOS的链接

这里我们先回到文章开头的问题：

1.如果编译时一个Class的文件没有被加入到指定的target，则只要这个文件被别的地方引用了一定会导致编译不过，因为符号找不到。但是如果是Category的话则可以正常编过。

2.如果一个方法没有被定义，即使被被其他地方调用，仍然可以正常编译，只是会在运行时崩溃。

我们以一个例子来分析为什么：

首先我们分别创建一个文件main.m和一个Test.h和Test.m：

```objective-c
#import <Foundation/Foundation.h>
#import "Test.h"
int main(int argc, char * argv[]) {
    NSString * appDelegateClassName;
    @autoreleasepool {
        [Test test];
    }
    return 0;
}
```

```objective-c
#import <Foundation/Foundation.h>
@interface Test : NSObject
+ (void)test;
@end

#import "Test.h"

@implementation Test
+ (void)test {
    NSLog(@"test");
}
@end
```

然后我们尝试编译Test.m文件如下，指定编译文件为OC文件，优化等级为0：

```shell
$ clang -x objective-c -O0 -c Test.m -o Test.o
```

然后查看Test.o的符号:

```shell
$ nm -nm Test.o
                 (undefined) external _NSLog
                 (undefined) external _OBJC_CLASS_$_NSObject
                 (undefined) external _OBJC_METACLASS_$_NSObject
                 (undefined) external ___CFConstantStringClassReference
                 (undefined) external __objc_empty_cache
0000000000000000 (__TEXT,__text) non-external +[Test test]
0000000000000068 (__DATA,__objc_const) non-external l_OBJC_$_CLASS_METHODS_Test
0000000000000088 (__DATA,__objc_const) non-external l_OBJC_METACLASS_RO_$_Test
00000000000000d0 (__DATA,__objc_const) non-external l_OBJC_CLASS_RO_$_Test
0000000000000118 (__DATA,__objc_data) external _OBJC_METACLASS_$_Test
0000000000000140 (__DATA,__objc_data) external _OBJC_CLASS_$_Test
```





