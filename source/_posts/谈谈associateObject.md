---

toc: true
title: 从一次崩溃谈谈associateObject
date: 2019-10-23 21:06:22
categories: [iOS]
tags: [bugs]
---

## 前言

associateObject是运行时系统中提供一个非常有用的东西，可以在开发中提供大量的便利，但是如果使用不当反而会导致很多问题。

<!--more-->

## 崩溃分析

在项目中有一个单例模块用来管理全部的监听者，在添加监听时采用listenerWarpper来包装传入的监听者，数组持有这个listenerWarpper，listenerWarpper弱引用外部监听者，这样外部对象就不会被持有了。大致实现简化如下:

```objective-c
- (void)addListener:(id)listener {
dispatch_async(_queue, ^{
		ListenterWarpper *warpper = [ListenterWarpper alloc] initListener];
		[self.Listeners addObject:warpper]
});
}
```

外部用户通过调用removeListener来移除监听者，此时遍历监听者数组去掉监听者:

```objective-c
- (void)removeListener:(id)listener {
dispatch_async(_queue, ^{
  //for 循环找到需要移除的warpper ...
		[self.Listeners remove:warpper]
});
}
```

但是在实际运行中发现会出现野指针崩溃，外部调用removeListener的时候通常listener可能已经处于deallocting状态或者处于生命周期最后。此时listener指针并没有被释放，所以``_queue``持有了这个指针，但是当进入``_queue``的执行时，listener可能已经成了一个野指针了。这是一个典型的多线程问题。那么如何解决了？

问题的根源是获取listener被释放的时机，即捕获其deallocting的状态。我们知道当一个对象被释放时，弱引用表会把所有指向它的弱引用指针置空。那么只要能有一个对象保存一个指向listener的弱指针就可以了。假如采用一个全局表去做这样的事会引起查表的和维护表的成本。于是想到采用关联对象来做。

假如直接把弱指针关联，则关联对象直接就被释放了，于是想到用一个warpper来作为关联对象。

我们再注册Listener时添加了一个关联对象，关联对象持有弱引用的listenter，大致如下:

```objective-c
@interface WeakObjectContainer : NSObject
@property (nonatomic, readonly, weak) id object;
@end
```

```objective-c
WeakObjectContainer* weakObjc = [[WeakObjectContainer alloc] initWithObject:listener];
objc_setAssociatedObject(handlerTarget, @"kListenerWeakSelf", weakObjc, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
```

由于对象释放时首先会移除弱引用表中的弱指针，所以这里有两个判断来得知当前listener是否处于待释放状态:

```objective-c
WeakObjectContainer* weakObjc = objc_getAssociatedObject(target, @"kListenerWeakSelf");
    if (weakObjc == nil || weakObjc.object == nil) {
        return;
    }
```

但是在实际使用中还是发现还是会出现野指针问题，经过分析，发现问题出在设置关联的policy上。

## associateObject内存管理

associateObject的实现并不复杂，有一个全局的静态哈希表，以对象为key，value为另一个哈希表，这个哈希表的key则是关联的key，value就是运行时系统封装关联对象(ObjcAssociation)。添加关联对象就是添加到这个静态表。同事如果有旧的关联对象，调用封装的ReleaseValue函数释放旧的关联对象。而获取关联对象则就是从这个静态哈希表中取。

当对象被释放进入dealloc状态时，运行时系统会解除它的关联引用，并在合适的时候调用release。它的实现和使用都不复杂。但是值得关注的是传入时的policy参数，在runtime的实现中policy决定了关联对象的生命周期:

public的runtime头文件只有几种policy，分别对应到源码中的policy为:

```
OBJC_ASSOCIATION_ASSIGN = OBJC_ASSOCIATION_SETTER_ASSIGN //设置关联时不会做retain
OBJC_ASSOCIATION_RETAIN_NONATOMIC = OBJC_ASSOCIATION_SETTER_RETAIN//设置时会retain
OBJC_ASSOCIATION_COPY_NONATOMIC = OBJC_ASSOCIATION_SETTER_COPY //设置时会copy
```

这三种跟源码是一一对应的，但是剩下两种是融合的模式：

`OBJC_ASSOCIATION_RETAIN`包含以下：

```
OBJC_ASSOCIATION_SETTER_RETAIN | OBJC_ASSOCIATION_GETTER_RETAIN
```

`OBJC_ASSOCIATION_COPY`包含如下：

```
OBJC_ASSOCIATION_SETTER_RETAIN | OBJC_ASSOCIATION_SETTER_COPY | OBJC_ASSOCIATION_GETTER_RETAIN
```

Policy不同会影响对象的生命周期。如要表现在

set关联对象时首先调用一下函数:

```c
static id acquireValue(id value, uintptr_t policy) {
    switch (policy & 0xFF) {
    case OBJC_ASSOCIATION_SETTER_RETAIN:
        return objc_retain(value);
    case OBJC_ASSOCIATION_SETTER_COPY:
        return ((id(*)(id, SEL))objc_msgSend)(value, SEL_copy);
    }
    return value;
}
```

也就是如果policy为`OBJC_ASSOCIATION_SETTER_RETAIN`，引用计数会加1。如果是`OBJC_ASSOCIATION_SETTER_COPY`，会调用关联对象的copy方法。

而释放旧的关联对象时，如果policy为`OBJC_ASSOCIATION_SETTER_RETAIN`，引用计数会减1，来确保对象正常释放：

```c
static void releaseValue(id value, uintptr_t policy) {
    if (policy & OBJC_ASSOCIATION_SETTER_RETAIN) {
        return objc_release(value);
    }
}
```

也就是只要设置关联时传入的policy不是`OBJC_ASSOCIATION_SETTER_ASSIGN`。关联对象再关联期间都是保活的。<b>所以`OBJC_ASSOCIATION_SETTER_ASSIGN`要谨慎使用除非你能确保他的生命周期</b>。

而在Get时，如果policy为`OBJC_ASSOCIATION_GETTER_RETAIN`，关联对象引用计数也会加1：

```c
if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) {
     objc_retain(value);
}
```

即传入的Polciy为`OBJC_ASSOCIATION_RETAIN`或者`OBJC_ASSOCIATION_COPY`时关联对象被取出来引用计数也会加1。

而传入的是`OBJC_ASSOCIATION_RETAIN_NONATOMIC`时get时并不会做retain操作。所以当切换到另一个线程的时候，这个associateObject可能已经被释放了，而导致崩溃。

那么一定要使用`OBJC_ASSOCIATION_RETAIN`吗，如果不是有多线程的情况需要延迟对象的生命周期，采用`OBJC_ASSOCIATION_RETAIN_NONATOMIC`反而会提高效率，因为objc_retain做的事情还真滴不少。
