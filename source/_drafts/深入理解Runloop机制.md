---
toc: true
title: 深入理解Runloop机制
date: 2020-03-03 16:48:59
categories:
tags:
---

runloop是一个事件处理循环，用于调用和协调未来的事件。它可以在事件到来是催动线程工作，并且在空闲时使线程睡眠。也就是`runloop`是用于管理线程工作的。

主线程自带一个runloop。新建的线程需要手动创建runloop。

runloop等同于有一个while或者for循环，保持一致运行接收事件。事件有两种：

1. 输入源(input sources), 传递异步事件。通常由另一个线程发送过来的消息。
2. 时钟源(Timer sources)。

每种事件都有一个指定的handler进行处理。

盗图官方文档的图片，非常形象哈，runloop就像一个传送带，然后把新输入的事件放在传送带上。

![Structure of a run loop and its sources](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/Art/runloop.jpg)

为了处理输入事件，runloop也会在收到时发送通知，通过对runloop添加观察者可以在runloop接收到这些事件时收到通知。不过只是CoreFoundation框架提供的接口:

```objective-c
    //获得当前线程的runloop
    NSRunLoop *runnloop = [NSRunLoop currentRunLoop];
    //获取CFRunloop
    CFRunLoopRef cfRunLoop = [runnloop getCFRunLoop];
    //创建context
    CFRunLoopObserverContext ctx;
    //创建观察者，传入的activity是要观察的事件类型。 函数指针是收到事件的回调
    CFRunLoopObserverRef observer = CFRunLoopObserverCreate(CFAllocatorGetDefault(), kCFRunLoopAllActivities, true, 0, myCallBack, &ctx);
    //注册观察者，并觉得运行模式
    CFRunLoopAddObserver(cfRunLoop, observer, kCFRunLoopDefaultMode);
```

CFRunLoopObserverContext的结构定义如下，

```c
typedef struct {
    CFIndex	version;
    void *	info;
    const void *(*retain)(const void *info);
    void	(*release)(const void *info);
    CFStringRef	(*copyDescription)(const void *info);
} CFRunLoopObserverContext;
```

runloop观察者的事件类型如下:

```c
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0), //进入runloop
    kCFRunLoopBeforeTimers = (1UL << 1), //Timer信号
    kCFRunLoopBeforeSources = (1UL << 2), //事件处理之前
    kCFRunLoopBeforeWaiting = (1UL << 5), //进入休眠之前
    kCFRunLoopAfterWaiting = (1UL << 6), //唤醒之前
    kCFRunLoopExit = (1UL << 7), //退出runloop
    kCFRunLoopAllActivities = 0x0FFFFFFFU //全部事件
};
```

通常主线程runloop在进入之后会执行一段时间的Timer信号。 如果无任何输入则进入waiting。直到再次被唤醒进入afterWaiting（例如点击主屏幕）。

### Runloop的模式

runloop的有几种模式，runloop在初始化时可以设置模式指定运行在那种模式下。它只会把在这些模式的输入事件和时钟事件传递到线程执行。以及在收到这个模式的事件时才会通知注册在这个模式下的观察者。

runlopp的模式是以字符串进行标志的，系统提供了两种模式kCFRunLoopDefaultMode和kCFRunLoopCommonModes。通常runloop初始化时如果未指定模式则被设置为kCFRunLoopDefaultMode。

<b>模式可以理解为标签，每个事件都被打上了标签，runloop在接收到事件会判断它的标签。</b>

所以你可以自定义这个标签来添加你的模式。

### 输入源

输入事件源指异步的传递信息到指定的线程。输入的事件源取决于输入是事件源的类型。通常由两类：

1. 基于端口输入的事件，监测Mach端口。
2. 自定义输入事件源。

runloop运行时不会在意输入源是自定义的还是mach端口来的。唯一的区别是，端口源通过内核信号发送过来，而自定义输入源则是从其它线程发送的信号。

#### 基于端口的输入源

基于端口的输入源

cocoa和CoreFoundation提供了内置方法来创建基于端口的输入源。通过NSProt可以把一个port添加到runloop.

#### 自定义输入源

[CFRunLoopSourceRef](https://developer.apple.com/documentation/corefoundation/cfrunloopsource) 是CoreFoundation提供的接口用于创建输入源。

#### 时钟源

时钟源在一个指定的时间异步的传输事件到你的线程。也是一种通知线程做某些事的方式。runloop不运行则timer也不会运行（直接在新建的线程中运行timer并不会执行，因为它是基于runloop的）。如果runloop的模式不是timer支持的模式也不会fire。

有两种方式，一种是为线程创建runloop。为线程创建runloop直接调用[NSRunLoop currentRunLoop]即可，如果有runloop会返回，没有的话会为线程重新创建。然后将timer添加到runloop上即可:

```objective-c
_thread = [[NSThread alloc] initWithBlock:^{
        @autoreleasepool {
            NSRunLoop *runloop = [NSRunLoop currentRunLoop];
            NSTimer *timer = [NSTimer timerWithTimeInterval:1
                                             repeats:YES
                                               block:^(NSTimer *_Nonnull timer) {
                                                   NSLog(@"111");
                                               }];
          //需要把timer加入runloop
            [runloop addTimer:timer forMode:NSRunLoopCommonModes];
            [runloop run];
          //直接scheduled并不能作用在此
            [NSTimer scheduledTimerWithTimeInterval:1 repeats:YES block:^(NSTimer * _Nonnull timer) {
                NSLog(@"222");
            }];
        }
    }];
    [_thread start];
```

也可以采用GCD源：

```c
  dispatch_queue_attr_t attr = dispatch_queue_attr_make_with_qos_class(DISPATCH_QUEUE_SERIAL, QOS_CLASS_DEFAULT, 0);
    dispatch_queue_t queue = dispatch_queue_create("joey", attr);
    _timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, _queue);
    dispatch_source_set_timer(_timer, DISPATCH_TIME_NOW, 1 * NSEC_PER_SEC, 1 * NSEC_PER_SEC);
    dispatch_source_set_event_handler(_timer, ^{
        NSLog(@"3333");
    });
    dispatch_resume(_timer);
```

无论哪一种都必须保存引用计数。比如第一种NSThread持有runloop。runloop持有timer。所以self必须NSThread才能保证都不被释放。

而在GCD中，则是source持有queue。所以必须保证self持有source。

如果Timer错过了一次执行，会等到下一次周期到的时候再执行。



处理系统默认的

## 参考资料

https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html#//apple_ref/doc/uid/10000057i-CH16-SW1

