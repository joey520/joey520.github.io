---
toc: true
title: 深入理解Dyld
date: 2020-02-01 02:48:30
categories:
tags:
---


## 加载

[dyld](http://opensource.apple.com/tarballs/dyld)是苹果出的动态库动态加载器，在启动程序时用于加载共享库和动态库。苹果为了进行优化把一些公共的系统库制作成了共享缓存。这些公共的动态库按不同的架构被放在一个缓存文件中成为`dyld_shared_cache`文件。

[dyld_cache_extract](https://github.com/macmade/dyld_cache_extract)是一个可以解析``dyld_shared_cache``的GUI工具。

通过xcode堆栈可以看到在调用`main`函数之前是调用了`dyld_start`:

打开`dyldStartup.s`可以查看到`__dyld_star`在不同架构下的汇编实现，这里截取`x86_64`下的实现：

```assembly
__dyld_start:
	popq	%rdi		# param1 = mh of app
	pushq	$0		# push a zero for debugger end of frames marker
	movq	%rsp,%rbp	# pointer to base of kernel frame
	andq    $-16,%rsp       # force SSE alignment
	subq	$16,%rsp	# room for local variables

	# call dyldbootstrap::start(app_mh, argc, argv, slide, dyld_mh, &startGlue)
	movl	8(%rbp),%esi	# param2 = argc into %esi
	leaq	16(%rbp),%rdx	# param3 = &argv[0] into %rdx
	movq	__dyld_start_static(%rip), %r8
	leaq	__dyld_start(%rip), %rcx
	subq	 %r8, %rcx	# param4 = slide into %rcx
	leaq	___dso_handle(%rip),%r8 # param5 = dyldsMachHeader
	leaq	-8(%rbp),%r9
	call	__ZN13dyldbootstrap5startEPK12macho_headeriPPKclS2_Pm
	movq	-8(%rbp),%rdi
	cmpq	$0,%rdi
	jne	Lnew

    	# clean up stack and jump to "start" in main executable
	movq	%rbp,%rsp	# restore the unaligned stack pointer
	addq	$8,%rsp 	# remove the mh argument, and debugger end frame marker
	movq	$0,%rbp		# restore ebp back to zero
	jmp	*%rax		# jump to the entry point
```

根据`call`的`C++`符号可以看到可以看到调用了`dyldbootstrap::start(const struct machoHeader*, int ...)这个方法:

```
uintptr_t start(const struct macho_header* appsMachHeader, int argc, const char* argv[], 
				intptr_t slide, const struct macho_header* dyldsMachHeader,
				uintptr_t* startGlue)
{
	// if kernel had to slide dyld, we need to fix up load sensitive locations
	// we have to do this before using any global variables
    //获取当前image，dyldsMachHeader的slide，其实是拿当前dyldsMachHeader地址减去第一个文件，通常为__Text的地址。
    //获取基地址偏移
    slide = slideOfMainExecutable(dyldsMachHeader);
    bool shouldRebase = slide != 0;
#if __has_feature(ptrauth_calls)
    shouldRebase = true;
#endif
    if ( shouldRebase ) {
        //重定位
        rebaseDyld(dyldsMachHeader, slide);
    }

	// allow dyld to use mach messaging
    //是dyld可以调用mach系统调用
	mach_init();

	// kernel sets up env pointer to be just past end of agv array
	const char** envp = &argv[argc+1];
	
	// kernel sets up apple pointer to be just past end of envp array
	const char** apple = envp;
	while(*apple != NULL) { ++apple; }
	++apple;

	// set up random value for stack canary
    //避免栈溢出
	__guard_setup(apple);

#if DYLD_INITIALIZER_SUPPORT
	// run all C++ initializers inside dyld
	runDyldInitializers(dyldsMachHeader, slide, argc, argv, envp, apple);
#endif

	// now that we are done bootstrapping dyld, call dyld's main
    //获取main executeable的 slide
	uintptr_t appsSlide = slideOfMainExecutable(appsMachHeader);
    //调用dyld的mian方法
	return dyld::_main(appsMachHeader, appsSlide, argc, argv, envp, apple, startGlue);
}
```

`_main`方法是`dyld`的入口。内核在加载`dyld`并且调用`__dyld_start`时会传入几个寄存器（就是上面方法中的一些变量）。并且在最后调用`_main`进入`dyld`的处理流程。`_main`函数非常长，简单概括主要做了一下几件事：

```mermaid
graph LR
A[内核] --> B[dyld start] 
B --> C[配置环境变量]
C --> D[加载共享缓存]
D --> E[实例化主程序]
E --> F[插入动态库]
F --> G[链接主程序]
G --> H[链接动态库]
H --> I[绑定弱符号]
I --> J[调用所有初始化方法]
J --> K[找到主程序main入口并返回]
K --> L[内核调用main]
```

在`executeable`类型的`mach-o`中专门有一个`LoadCommand`为`main`，记录了主程序入口的函数信息，通过`entry offser`+`slide`可以获取到函数地址，dyld找到`main`函数z指针之后返回，内核从主线程启动mian`函数:

![image-20200203162232907](/Users/joey.cao/Library/Application Support/typora-user-images/image-20200203162232907.png)

具体详细每一步的分析可以查看这位专门做逆向的[大佬的博客](https://www.dllhook.com/post/238.html)。

## 动态绑定

符号是如何进行调用的，在`main`函数中添加两个`printf`函数，并分别打上断点然后运行。程序进程在第一个断点处挂起，利用`LLDB`的`disassemble`指令可以打印当前栈帧的反汇编代码，可以看到调用`printf`实际上`callq`了``0x100000f82`。通过助记符可以猜到是调用到了`dyld`的`stub helper`:

```assembly
dis
TestDyld`main:
    0x100000f40 <+0>:  pushq  %rbp
    0x100000f41 <+1>:  movq   %rsp, %rbp
    0x100000f44 <+4>:  subq   $0x20, %rsp
    0x100000f48 <+8>:  movl   $0x0, -0x4(%rbp)
    0x100000f4f <+15>: movl   %edi, -0x8(%rbp)
    0x100000f52 <+18>: movq   %rsi, -0x10(%rbp)
    0x100000f56 <+22>: leaq   0x45(%rip), %rdi          ; "Hello, World!\n"
    0x100000f5d <+29>: movb   $0x0, %al
->  0x100000f5f <+31>: callq  0x100000f82               ; symbol stub for: printf
    0x100000f64 <+36>: leaq   0x37(%rip), %rdi          ; "Hello, World!\n"
    0x100000f6b <+43>: movl   %eax, -0x14(%rbp)
    0x100000f6e <+46>: movb   $0x0, %al
    0x100000f70 <+48>: callq  0x100000f82               ; symbol stub 
```

然后调用我们进入`0x100000f82`这个地址的栈帧:

```assembly
dis -start-address 0x100000f82
TestDyld`printf:
    0x100000f82 <+0>: jmpq   *0x88(%rip)               ; (void *)0x0000000100000f98
    0x100000f88:      leaq   0x71(%rip), %r11          ; (void *)0x0000000000000000
    0x100000f8f:      pushq  %r11
    0x100000f91:      jmpq   *0x71(%rip)               ; (void *)0x00007fff703ae214: dyld_stub_binder
    0x100000f97:      nop    
    0x100000f98:      pushq  $0x0
    0x100000f9d:      jmp    0x100000f88
```

可以看到这里`jmp`到`0x0000000100000f98，查看`mach-O`可以看到如下

![image-20200201025828020](/Users/joey.cao/Library/Application Support/typora-user-images/image-20200201025828020.png)

恰好就是`Lazy Symbol Pointers`的`data`，实际上指向的`stub_helper`中的代码:

![image-20200201030642625](/Users/joey.cao/Library/Application Support/typora-user-images/image-20200201030642625.png)

我们知道编译器会同时生成`Lazy Symbol Pointers`和`stub`。这里是直接`callq`到`stub_helper`中了的指定行了，注意`stub_helper`也是由编译器创建用于解析`stub`，此时代码跳转到`100000F98`这一句，可以看到显示入栈`0x0，这个`0x0`是编译时给的编号，然后跳转到了`10000F88`其实就是`stub_helper`中的函数，这里传入的参数`r11`实际是通过位移拿到的符号的偏移`0x100001000`恰好是`Lazy Symbol Pointers`的起始位置, 进行偏移找到指定的符号的偏移地址，然后调用`stub binder`获取函数的实际地址。 我们查看`dyld`源码进行分析查看是如何绑定地址的：

![image-20200201031453139](/Users/joey.cao/Library/Application Support/typora-user-images/image-20200201031453139.png)

调用了`fastBindLazySymbol`这个函数，这里做到了汇编层的干涉，查看`dyld_stub_binder.s`可以看到各个架构下的汇编源码:

```assembly
dyld_stub_binder_:
	movl		MH_LOCAL(%esp),%eax	# call dyld::fastBindLazySymbol(loadercache, lazyinfo)
	movl		%eax,MH_PARAM_OUT(%esp)
	movl		LP_LOCAL(%esp),%eax
	movl		%eax,LP_PARAM_OUT(%esp)
	call		__Z21_dyld_fast_stub_entryPvl
	movdqa		XMMM0_SAVE(%esp),%xmm0	# restore registers
	movdqa		XMMM1_SAVE(%esp),%xmm1
	movdqa		XMMM2_SAVE(%esp),%xmm2
	movdqa		XMMM3_SAVE(%esp),%xmm3
	movl		ECX_SAVE(%esp),%ecx
	movl		EDX_SAVE(%esp),%edx
	movl		%eax,%ebp		# move target address to epb
	movl		EAX_SAVE(%esp),%eax	# restore eax
	addl		$STACK_SIZE+4,%esp	# cut back stack
	xchg		%ebp, (%esp)		# restore ebp and set target to top of stack
	ret					# jump to target
```

这里可以看到第5行调用`C++`函数`_dyld_fast_stub_entry`，参数为`ImageLoader`和延迟加载的信息指针，我们关注下这个方法，它其实在找`dyld`内的`__dyld_fast_stub_entry`这个函数指针，然后调用它，这里`dyld`采用了`dyld_funcs`这样一张map表来实现字符串和函数的映射，实际调用为`fastBindLazySymbol`，这里我们简要分析下这个方法的流程：

这里需要解析指定的`image`即符号定义所在的`module`。实际上`imageLoader`这个类已经解析好了。找到`Indirect symbol table`，找到`S_LAZY_SYMBOL_POINTERS`，，这里的操作和`FishHook`非常相似。

```c++
const struct macho_segment_command* seg = (struct macho_segment_command*)cmd;
					const struct macho_section* const sectionsStart = (struct macho_section*)((char*)seg + sizeof(struct macho_segment_command));
					const struct macho_section* const sectionsEnd = &sectionsStart[seg->nsects];
					for (const struct macho_section* sect=sectionsStart; sect < sectionsEnd; ++sect) {
						const uint8_t type = sect->flags & SECTION_TYPE;
						uint32_t symbolIndex = INDIRECT_SYMBOL_LOCAL;
						//找到lazy pointers LoadCommand
						if ( type == S_LAZY_SYMBOL_POINTERS ) {
							//计算有几个指针
							const size_t pointerCount = sect->size / sizeof(uintptr_t);
							//找到lazy pointers section。 symbolPointers即第一个函数指针
							uintptr_t* const symbolPointers = (uintptr_t*)(sect->addr + fSlide);
							//按道理来讲lazyPointer应该大于symbolPointers, 并且小于最后一个。
							//这里是为了安全防止数组越界
							if ( (lazyPointer >= symbolPointers) && (lazyPointer < &symbolPointers[pointerCount]) ) {
								//找到indirect tables中的位置
								const uint32_t indirectTableOffset = sect->reserved1;
								//算出往后偏移几个
								const size_t lazyIndex = lazyPointer - symbolPointers;
								//取出符号的index
								symbolIndex = indirectTable[indirectTableOffset + lazyIndex];
							}
              。。。。
if ( symbolIndex != INDIRECT_SYMBOL_ABS && symbolIndex != INDIRECT_SYMBOL_LOCAL ) {
							//找到符号，并取出符号名在String table中的位置
							const char* symbolName = &fStrings[fSymbolTable[symbolIndex].n_un.n_strx];
							const ImageLoader* image = NULL;
							//在image中找到符号的地址
							uintptr_t symbolAddr = this->resolveUndefined(context, &fSymbolTable[symbolIndex], twoLevel, false, true, &image);
							//这里会把lazyPointer进行绑定
							symbolAddr = this->bindIndirectSymbol(lazyPointer, sect, symbolName, symbolAddr, image,  context);
							++fgTotalLazyBindFixups;
							return symbolAddr;
						}
```



找到符号，则会调用`resolveUndefined`当前的image中查找到符号的地址，并调用`bindIndirectSymbol`进行重新绑定，然后赋值给原来的符号表中的符号的address字段。这样就可以真正调用到对应的函数了，并且此时符号的地址已经是真正函数的地址了。

我们放开断点执行到第二个断点，然后直接查看第二次调用`printf`的堆栈:

```assembly
dis -start-address 0x100000f82
TestDyld`printf:
    0x100000f82 <+0>: jmpq   *0x88(%rip)               ; (void *)0x00007fff7044cec4: printf
```

可以看到此时调用已经变了不再需要通过`stub`去查找函数地址了，而是直接访问了`0x00007fff7044cec4`这个地址，助记符可以看到是`printf`函数，我们查看`0x00007fff7044cec4`的栈帧如下:

```assembly
dis -start-address 0x00007fff7044cec4
libsystem_c.dylib`printf:
    0x7fff7044cec4 <+0>:  pushq  %rbp
    0x7fff7044cec5 <+1>:  movq   %rsp, %rbp
    0x7fff7044cec8 <+4>:  subq   $0xd0, %rsp
    0x7fff7044cecf <+11>: movq   %rdi, %r10
    0x7fff7044ced2 <+14>: testb  %al, %al
    0x7fff7044ced4 <+16>: je     0x7fff7044cefc            ; <+56>
    0x7fff7044ced6 <+18>: movaps %xmm0, -0xa0(%rbp)
    0x7fff7044cedd <+25>: movaps %xmm1, -0x90(%rbp)
```

恰好就是`printf`的实现了。可见第二次调用时直接就访问了函数的地址，只有第一次调用时才需要通过`dyld`进行地址绑定。

这里需要注意：

1.符号动态绑定需要依赖`dyld_stub_binder`。而它也是一个外部符号，那么谁来绑定它呢，查看`got`可以看到``dyld_stub_binder``并不是一个延迟加载的符号，同样的`objc_msgSend`，`objc_release`这种比较重要的依赖函数都不能是延迟加载的。



## 参考资料

http://opensource.apple.com/tarballs/dyld

https://www.dllhook.com/post/238.html （一位主攻逆向的大佬的博客）