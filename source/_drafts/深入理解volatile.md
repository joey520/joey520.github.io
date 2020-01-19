---
toc: true
title: 深入理解voliate
date: 2020-01-11 22:08:30
categories: 多线程
tags:
---

## 前言



### 函数的跳转

在`setjmp.h`中定义了几个函数用于实现跨函数的跳转，它的效果和goto类似可以跳转到指定的地方，不同的是goto只能在跳转到函数内部的`label，`而`setjmp`则可以实现函数间的跳转。

我们知道函数跳转时是一个入栈的过程，每一个函数都是一栈帧，栈帧中存储了自动变量。那么当调用多层函数之后就成了多层的栈嵌套，假如此时需要从当前被调用的栈回到某个指定的栈帧时，只有两条路:依次出栈返回或者是利用`setjmp`直接返回。

首先我们可以看到`setjmp.h`这个头文件非常简单，可以说只有两个函数:

```c
__BEGIN_DECLS
extern int	setjmp(jmp_buf);
extern void longjmp(jmp_buf, int) __dead2;

#ifndef _ANSI_SOURCE
int	_setjmp(jmp_buf);
void	_longjmp(jmp_buf, int) __dead2;
int	sigsetjmp(sigjmp_buf, int);
void	siglongjmp(sigjmp_buf, int) __dead2;
#endif /* _ANSI_SOURCE  */

#if !defined(_ANSI_SOURCE) && (!defined(_POSIX_C_SOURCE) || defined(_DARWIN_C_SOURCE))
void	longjmperror(void);
#endif /* neither ANSI nor POSIX */
__END_DECLS
```

首先是为了`C++`因此符号错误添加`extern "C"`。利用`extern`修饰`setjmp`符号为外链接。

而下面则是为了适配不同平台的`source`进行了写预编译处理，关于`source`可以参考[GUN官方文档](https://www.gnu.org/software/libc/manual/html_node/Feature-Test-Macros.html)。

关于`setjmp`等函数解释在[苹果的手册](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/longjmperror.3.html)也可以找到。 说白了`setjmp`，`sigsetjmp`,`_setjmp`会把当前的调用环境保存在`jmp_buf`这个参数中，然后在调用`longjmp`时返回最近的注册的`setjmp`处继续执行

`jmp_buf`是一个`int`型数组，长度等同与当前架构所支持的寄存器数量，因此在不同的结构上也是不同，例如`x86_64`:

```c
#if defined(__x86_64__)
/*
 * _JBLEN is number of ints required to save the following:
 * rflags, rip, rbp, rsp, rbx, r12, r13, r14, r15... these are 8 bytes each
 * mxcsr, fp control word, sigmask... these are 4 bytes each
 * add 16 ints for future expansion needs...
 */
#define _JBLEN ((9 * 2) + 3 + 16)
typedef int jmp_buf[_JBLEN];
typedef int sigjmp_buf[_JBLEN + 1];
```

很明确了就是保存了一些运行堆栈必要的寄存器的值。

我们来实践一下这俩函数:

```objective-c
- (void)testSetjmp {
    int i = 10;
    NSLog(@"before jmp");
    if (setjmp(jmpb) != 0) {
        NSLog(@"after jmp");
        NSLog(@"--%d", i);
        NSLog(@"after exit");
        return;
    }
    
    if (setjmp(jmpb) != 0) {
        NSLog(@"after jmp 1");
        NSLog(@"--%d", i);
        NSLog(@"after exit 1");
        return;
    }
    i = 30;
    [self func:i];
    NSLog(@"func exit");
    exit(0);
}

- (void)func:(int)a {
    NSLog(@"func--%d", a);
    longjmp(jmpb, 1);
}
```

运行一下结果如下:

```c
2020-01-12 21:51:31.936757+0800 LearnRuntime[18886:3277707] before jmp
2020-01-12 21:51:31.936913+0800 LearnRuntime[18886:3277707] func--30
2020-01-12 21:51:31.937036+0800 LearnRuntime[18886:3277707] after jmp 1
2020-01-12 21:51:31.937147+0800 LearnRuntime[18886:3277707] --30
2020-01-12 21:51:31.937235+0800 LearnRuntime[18886:3277707] after exit 1
```

有几点需要注意。`longjmp`只会跳转到注册的最近的那一次`setjmp`。并且跳转是完全从`setjmp`处正常继续执行，如果去掉`setjmp`中的`return`这段代码就是一个死循环。还有跳转回来只会打印的`i`值变为了30。

此时如果我们打开buildSetting把优化打开，默认在`DEBUG`模式下，编译优化为`-O0`。此时我们改成`-O1`，再次编译运行：

```c
2020-01-12 13:38:54.560095+0800 LearnRuntime[4458:2419320] before jmp
2020-01-12 13:38:54.560244+0800 LearnRuntime[4458:2419320] func--30
2020-01-12 13:38:54.560370+0800 LearnRuntime[4458:2419320] after jmp
2020-01-12 13:38:54.560468+0800 LearnRuntime[4458:2419320] --10
2020-01-12 13:38:54.560566+0800 LearnRuntime[4458:2419320] after exit
```

可以看到`setjmp`里的打印出来的值变成了`setjmp`之前的最后状态。

这是为什么呢，因为在优化之后会多利用寄存器来存储参数的值，因为寄存器的操作效率更高，而`setjmp`会把寄存器的值保存下来，因此返回时再次恢复的是保存的寄存器里的值。通过[libc源码](http://git.musl-libc.org/cgit/musl/tree/src/setjmp/i386/setjmp.s)可以看到`setjmp`的实现如下:

```assembly
.global ___setjmp
.hidden ___setjmp
.global __setjmp
.global _setjmp
.global setjmp
.type __setjmp,@function
.type _setjmp,@function
.type setjmp,@function
___setjmp:
__setjmp:
_setjmp:
setjmp:
	mov 4(%esp), %eax  //把esp向上便宜4个字节保存到eax，注意这里的eax即为jmp_buf的地址
	mov    %ebx, (%eax)  //把ebx寄存器保存到jmp_buf[1]
	mov    %esi, 4(%eax)	//把esi寄存器保存到jmp_buf[2]
	mov    %edi, 8(%eax) //把edi寄存器保存到jmp_buf[3]
	mov    %ebp, 12(%eax) //把ebp寄存器保存到jmp_buf[4]
	lea 4(%esp), %ecx		//把栈顶寄存器esp前一个地址保存到ecx
	mov    %ecx, 16(%eax) //把exc拷贝到jmp_buf[5]
	mov  (%esp), %ecx  //把栈顶寄存器esp拷贝到ecx
	mov    %ecx, 20(%eax) //保存ecx到jmp_buf[6]
	xor    %eax, %eax //利用异或清零
	ret
```

可以清楚的看到`setjmp`保存寄存器的过程。关于常见这些寄存器的作用可以参考下[这篇博客](https://www.cnblogs.com/qq78292959/archive/2012/07/20/2600865.html)。

如果变量为`static`或者`globe`或者`volatile`修饰的，则不会被保存在寄存器中，因此恢复时是直接从内存中读取的真实的值。因此技术进行了编译优化拿到的仍然是修改之后的真实的值，做个测试我们分别添加几种类型的变量，并且在`setjmp`和`longjmp`之间修改其值，并且打开编译器优化如下:

```objective-c
static int a = 10;
int b = 20;
- (void)testSetjmp {
    int i = 10;
    volatile int c = 30;
    NSLog(@"before jmp");
    if (setjmp(jmpb) != 0) {
        NSLog(@"after jmp");
        NSLog(@"--a:%d, b:%d, c:%d, i:%d", a, b, c, i);
        NSLog(@"after exit");
        return;
    } else {
        NSLog(@"setjmp failed");
    }
    i = 30;
    a = 20;
    b = 30;
    c = 40;
    [self func:i];
    NSLog(@"func exit");
    exit(0);
}

- (void)func:(int)a {
    NSLog(@"func--%d", a);
    longjmp(jmpb, 1);
}
```

运行之后结果如下:

```c
2020-01-12 23:52:38.593172+0800 LearnRuntime[24899:3525907] before jmp
2020-01-12 23:52:38.593319+0800 LearnRuntime[24899:3525907] setjmp failed
2020-01-12 23:52:38.593448+0800 LearnRuntime[24899:3525907] func--30
2020-01-12 23:52:38.593552+0800 LearnRuntime[24899:3525907] after jmp
2020-01-12 23:52:38.593618+0800 LearnRuntime[24899:3525907] --a:20, b:20, c:40, i:10
2020-01-12 23:52:38.593699+0800 LearnRuntime[24899:3525907] after exit
```

只有自动变量i的值仍然是保持在寄存器里的值，而其它三个变量都保持了真实的值。 为了验证我们做个简单的试验如下:

```c
#include <stdio.h>
int main(int argv, char **argc) {
    int i = 20;
    printf("%d\n", i);
    return 0;
}
```

写一个简单的c程序，然后看一下在不优化和开启优化下汇编出来的机器指令是如何处理`int i = 20`这一句的：

```shel
clang hello.c -S -o hello.s
clang -Os hello.c -S -o helloOs.s
```

然后我们只挑出需要的几行:

```assembly
 cat hello.s
	movl	$20, -20(%rbp)
  
cat helloOs.s
	movl	$20, %esi
```

​	可以看到在没优化时20是保持在`rbp`寄存器向前偏移20字节地址处。`rbp`表示的是栈基址寄存器，表示当前栈的地址，所以这个20其实保持在栈内存中。而在开启优化之后，20保持在`esi`寄存器中，`esi`则是通用寄存器的一个，因为操作寄存器远比访问地址快，所以优化时几乎都采用寄存器代替重复的地址访问。所以在优化编译下，`setjmp`就把这些寄存器保存下来了，因此恢复到`setjmp`就恢复到了之前设置的值。

但是如果我们采用`volatile`修饰再次采用`-Os`进行优化如下:

```assembly
	cat helloVolatile.s
	movl	$20, -4(%rbp)
```

可以看到仍然是直接加载到内存上的



## 参考资料

Unix环境高级编程

https://www.gnu.org/software/libc/manual/html_node/Feature-Test-Macros.html

https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man3/longjmperror.3.html

http://git.musl-libc.org/cgit/musl/tree/src/setjmp/i386/setjmp.s

https://www.cnblogs.com/qq78292959/archive/2012/07/20/2600865.html