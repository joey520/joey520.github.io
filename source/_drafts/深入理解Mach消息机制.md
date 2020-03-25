---
toc: true
title: 深入理解Mach消息机制
date: 2020-03-25 10:51:30
categories:
tags:
---

## Mach架构



![img](https://developer.apple.com/library/archive/documentation/Darwin/Conceptual/KernelProgramming/art/osxarchitecture.gif)

盗用苹果官方文档中关于XNU的架构的图如上。它是整个Darwin系统的核心。可以看到最底层是mach微内核，主要负责最底层的服务，如CPU调度和进程间通信和与硬件驱动程序的通信（I/O Kit）。 Mach内核对底层的硬件架构进行封装，提供了一些接口来与无论是x86，还是arm64，还是原来的Power PC的通信。

上面一层是BSD层，对Mach的更高抽象，提供了POSIX（一套标准的接口原型）的兼容。主要工作包括对进程/线程模型的抽象，网络协议栈，文件系统的抽象等等。在之前学习Socket编程时知道，系统调用都是以0x2000000地址开始，这是BSD的起始地址，就是通过BSD对Mach进行访问。

### 内核启动流程

内核启动包含了Mach启动，BSDinit等等，thread_bootstrap等等。

### Mach消息传递

要理解Mach的工作原理，需要先了解几个概念:

1.消息mach_msg

在mach中一切操作都是通过发送信息来实现的。在`<mach/mach.h>`头文件可以看到很多与mach_msg相关的接口。这些接口就提供了通过mach_msg和Mach核心通信的途径。Mach内核做了一个极大的优化是，在消息传递时传递的不是消息数据，而是消息指针，因此整个传递过程不会出现内存拷贝的消耗（这里猜测一下是否会有传递过程中出现指针内存被改变的风险呢？）。

2.端口mach_port

既然通过消息传递，必然需要由端口来接收消息。在Mach内核中通mach_port_t来抽象端口。实际上是一个`unsigned int`类型的整形:

```c
typedef unsigned int            __darwin_natural_t;
```

所有对Mach源生对象的访问都需要通过端口，即通过这个对象端口的句柄。

消息传递通过的是`mach_msg`函数(同样消息的接收也是通过这个接口):

```c
extern mach_msg_return_t        mach_msg(
	mach_msg_header_t *msg,
	mach_msg_option_t option,
	mach_msg_size_t send_size,
	mach_msg_size_t rcv_size,
	mach_port_name_t rcv_name,
	mach_msg_timeout_t timeout,
	mach_port_name_t notify);
```

这个函数会调用到`mach_msg_trap()`，这个函数会从用户态进入调内核态，然后调用到系统调用`mach_msg_overwrite_trap( )`。

在XNU源码的`mach_msg.c`中可以找到这个函数的定义，无论是发送消息和接收消息都需要通过这个函数:

<details>
  <summary> mach_msg_overwrite_trap函数定义：</summary>


  ```c
  mach_msg_return_t
mach_msg_overwrite_trap(
	struct mach_msg_overwrite_trap_args *args)
{
  	mach_vm_address_t	msg_addr = args->msg;
	mach_msg_option_t	option = args->option;
	mach_msg_size_t		send_size = args->send_size;
	mach_msg_size_t		rcv_size = args->rcv_size;
	mach_port_name_t	rcv_name = args->rcv_name;
	mach_msg_timeout_t	msg_timeout = args->timeout;
	mach_msg_priority_t override = args->override;
	mach_vm_address_t	rcv_msg_addr = args->rcv_msg;
	__unused mach_port_seqno_t temp_seqno = 0;

	mach_msg_return_t  mr = MACH_MSG_SUCCESS;
	//获取VM空间
	vm_map_t map = current_map();
	
	/* Only accept options allowed by the user */
	option &= MACH_MSG_OPTION_USER;
	//检验消息
	if (option & MACH_SEND_MSG) {
		ipc_space_t space = current_space();
		ipc_kmsg_t kmsg;
	
		KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_START);
		//获取消息数据到kmsg中
		mr = ipc_kmsg_get(msg_addr, send_size, &kmsg);
	
		if (mr != MACH_MSG_SUCCESS) {
			KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_END, mr);
			return mr;
		}
	
		KERNEL_DEBUG_CONSTANT(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_LINK) | DBG_FUNC_NONE,
				      (uintptr_t)msg_addr,
				      VM_KERNEL_ADDRPERM((uintptr_t)kmsg),
				      0, 0,
				      0);
		//拷贝消息
		mr = ipc_kmsg_copyin(kmsg, space, map, override, &option);
		//如果拷贝失败，释放kmsg
		if (mr != MACH_MSG_SUCCESS) {
			ipc_kmsg_free(kmsg);
			KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_END, mr);
			return mr;
		}
	 //发送msg
		mr = ipc_kmsg_send(kmsg, option, msg_timeout);
	
		if (mr != MACH_MSG_SUCCESS) {
      //尝试重发
			mr |= ipc_kmsg_copyout_pseudo(kmsg, space, map, MACH_MSG_BODY_NULL);
			(void) ipc_kmsg_put(kmsg, option, msg_addr, send_size, 0, NULL);
			KDBG(MACHDBG_CODE(DBG_MACH_IPC,MACH_IPC_KMSG_INFO) | DBG_FUNC_END, mr);
			return mr;
		}
	
	}
	//如果是接收到的内核消息
	if (option & MACH_RCV_MSG) {
		thread_t self = current_thread();
		ipc_space_t space = current_space();
		ipc_object_t object;
		ipc_mqueue_t mqueue;
	//获取IPC队列
		mr = ipc_mqueue_copyin(space, rcv_name, &mqueue, &object);
		if (mr != MACH_MSG_SUCCESS) {
			mach_port_guard_exception(rcv_name, 0, 0, kGUARD_EXC_RCV_INVALID_NAME);
			return mr;
		}
		/* hold ref for object */
	
		if ((option & MACH_RCV_SYNC_WAIT) && !(option & MACH_SEND_SYNC_OVERRIDE)) {
			ipc_port_t special_reply_port;
			__IGNORE_WCASTALIGN(special_reply_port = (ipc_port_t) object);
			/* link the special reply port to the destination */
			mr = mach_msg_rcv_link_special_reply_port(special_reply_port,
					(mach_port_name_t)override);
			if (mr != MACH_MSG_SUCCESS) {
				io_release(object);
				return mr;
			}
		}
	//获取消息的信息
		if (rcv_msg_addr != (mach_vm_address_t)0)
			self->ith_msg_addr = rcv_msg_addr;
		else
			self->ith_msg_addr = msg_addr;
		self->ith_object = object;
		self->ith_rsize = rcv_size;
		self->ith_msize = 0;
		self->ith_option = option;
		self->ith_receiver_name = MACH_PORT_NULL;
		self->ith_continuation = thread_syscall_return;
		self->ith_knote = ITH_KNOTE_NULL;
	//调用从IPC队列中取出信息
		ipc_mqueue_receive(mqueue, option, rcv_size, msg_timeout, THREAD_ABORTSAFE);
		if ((option & MACH_RCV_TIMEOUT) && msg_timeout == 0)
			thread_poll_yield(self);
		return mach_msg_receive_results(NULL);
	}
	
	return MACH_MSG_SUCCESS;
}
  ```

</details>

可以发现发送消息最后调到的是`ipc_mqueue_send,`接受消息最后调到的是`ipc_mqueue_receive`。这个queue就是一个内核的消息队列，两端通过它进行消息的传递。

### 消息的结构

一般消息的结构主要有2部分组成:

```c
typedef struct{
	mach_msg_header_t       header;
	mach_msg_body_t         body;
} mach_msg_base_t;
```

必选的`mach_msg_header_t`，保存消息的基本信息。`mach_msg_header_t`的定义如下:

```c
typedef struct{
	mach_msg_bits_t       msgh_bits; //标志位。标志是合种类型的消息
	mach_msg_size_t       msgh_size; //消息大小，这里包含了body（可选）的长度
	mach_port_t           msgh_remote_port; //接收端口
	mach_port_t           msgh_local_port; //发送断
	mach_port_name_t      msgh_voucher_port; //
	mach_msg_id_t         msgh_id; //消息的唯一ID
} mach_msg_header_t;
```

可选的`mach_msg_body_t`，描述了包含了多少附件信息。实际就是一个`unsigned int`数据,结构定义如下:

```c
typedef struct{
	mach_msg_size_t msgh_descriptor_count;
} mach_msg_body_t;
```

通常只有复杂消息才会存在附件。当消息为复杂消息时它首部的bits为`MACH_MSGH_BITS_COMPLEX`。它的body中包含了descriptor的个数，以及多个具体的descriptor数据。每个descriptor都有一个字段type标志当前是合种类型，这样就形成了一段TV格式的数据，也就可以解析消息数据了。

```c
typedef unsigned int mach_msg_descriptor_type_t;

#define MACH_MSG_PORT_DESCRIPTOR                0
#define MACH_MSG_OOL_DESCRIPTOR                 1
#define MACH_MSG_OOL_PORTS_DESCRIPTOR           2
#define MACH_MSG_OOL_VOLATILE_DESCRIPTOR        3
#define MACH_MSG_GUARDED_PORT_DESCRIPTOR        4
```

关于各种descriptor的定义结构可以参见<mach/messge.h>。 <b>特别要注意的这里采用的4字节对齐`#pragma pack(push, 4)`</b>



## Mach信息的应用

Mach信息不止在内核中使用，同样还会传递上上层，比如`Core Foundation`和`Foundation`中的runloop。比如信号传递。

## Mach信号

在之前我们分析过信号在类Unix系统中的传递机制。然而在iOS中，最底层来自于Mach异常，然后才转换成Unix信号。举个例子开发中经常碰到的`EXC_BAD_ACCESS`。当初发生在访问到应用虚拟地址空间之外的内存，此时Mach内核触发异常，如果没有在Mach层捕获这个异常，Mach内核就会把它转换成SIGSEGV信号。 这也就是我们通常在崩溃的日志中同时看到这两个信息`EXC_BAD_ACCESS (SIGSEGV)`的原因。

也就是说其实我们可以直接在Mach层就捕获到崩溃异常，这样可以更快的处理崩溃信息。但是Mach只是Darwin系统特有的东西，而Unix信号则是POSIX标准，更加通用，因此通常都是等转换成Unix信号才去处理。否则容易有兼容性问题，以及如果有另一个用户想要捕获信号就捕获不到了。

不过如果对系统的运行敏感度很高，而且不考虑其它因素，可以直接在Mach层捕获异常:

```

```

