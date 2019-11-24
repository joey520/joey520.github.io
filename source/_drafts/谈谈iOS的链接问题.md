---
toc: true
title: 谈谈iOS的链接问题
date: 2019-11-23 17:29:33
categories:
tags:
---

## 前言

在MSDK的工程中是拥有多个子工程嵌套组成的而不是利用workspace进行平铺，虽然这些都是历史问题，但是



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

