---
toc: true
title: 深入理解Mach内核堆栈
date: 2020-02-28 02:17:40
categories:
tags:
---

## 前言

在捕获`Crash`时我们通常采用`backtrace`来获取堆栈，但是往往只有少数几个关键调用帧的信息。但是在`LLDB`中通过`backtrace`或者是第三方`crash`统计中总是能获取相当详细的堆栈信息，借着逆向[Bugly]()的机会，深入学习一下`Mach`内核堆栈相关知识。

## 内核

虽然苹果老被黑系统封闭，但是其实除了`GUI`等等，它的内核和运行时系统都是开源的。内核的源码可以在苹果的[网站](https://opensource.apple.com)和[github](https://github.com/apple/darwin-xnu)上拿到，并且还会更新，说一句良心不能为过把，虽然我看不懂。。。

不过看不懂还是得硬啃一下，搭配上苹果官网[Kernel Programming Guide](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/About/About.html#//apple_ref/doc/uid/TP30000905-CH204-TPXREF101)还要软饭硬吃一下的。

![img](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/art/osxarchitecture.gif)

这张图盗用苹果官方文档。可以看到内核主要分为两个部分`mach`和`BSD`。

`mach`主要工作是虚拟内存管理，模块架构以及进程间通信(IPC)，程序间调度等等。

`BSD`主要工作是管理文件系统，网络服务（非硬件级别的），系统调用支持（还记得信号机制学习中的`syscall`吗），`BSD`进程模型管理和信号管理，`Free BSD APIs`， `POSIX APIs`，线程管理等等。

### Mach

`mach`相关的接口在`<mach/mach.h>`下可以看到

### 线程管理

`Mach`的任务调度是基于一个运行的队列控制系统，具有4个优先级:

| Priority Band        | Characteristics                                              |
| :------------------- | :----------------------------------------------------------- |
| Normal               | normal application thread priorities                         |
| System high priority | threads whose priority has been raised above normal threads  |
| Kernel mode only     | reserved for threads created inside the kernel that need to run at a higher priority than all user space threads (I/O Kit workloops, for example) |
| Real-time threads    | threads whose priority is based on getting a well-defined fraction of total clock cycles, regardless of other activity (in an audio player application, for example). |



## 参考资料

https://opensource.apple.com

https://github.com/apple/darwin-xnu

https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/About/About.html#//apple_ref/doc/uid/TP30000905-CH204-TPXREF101

