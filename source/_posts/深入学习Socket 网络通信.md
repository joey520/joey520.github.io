---

toc: true
title: iOS程序员必备的Socket知识
date: 2019-12-12 17:04:24
categories: iOS
tags: socket
---

## 前言

最近工作和`socket`打了不少交道，对于`socket`也是一知半解，导致遇到不少问题，于是抽时间好好学习了一下`socket`相关的知识。本文主要对`socket`关键字进行详细分析，深入理解了不同关键字在不同情况下的行为表现，一些相关的工作原理，最后以一个简易的一对多客户端和服务端通信[Demo](https://github.com/joey520/Blogs/tree/master/TestSocket)来实地分析和学习`socket`工作原理。

<!--more-->

## 什么是socket

`Unix高级编程`书中这么描述`socket`:不同计算机进程间可以通过网络通信。而`socket`（套接字）即通信的一个载体，用专业数据讲就是<b>套接字是通信端点的抽象</b>。`socket`在系统标准库定义一套接口，需要使用必须`include<sys/socket.h>`。

其实在开始时我一直不理解`socket`是什么，当接受了`Unix`系统一切皆文件这个概念之后，大概明白了文件描述符的意思，`socket`即表示用于I/O的文件描述符。也就是返回这个socket数字是一个文件的描述，通过这个`socket`可以对文件进行读写。说白了`socket`就是对于网络端口的封装，发送数据即用户进程把数据写入到系统进程缓冲区，然后通过网络链路发送到对方的系统进程的缓存区。而取数据就是用户数据去系统进程的缓冲区去取。因此数据传输受系统进程缓冲区的限制，比如常见的粘包问题，和发送读取数据不全很可能是缓冲区受限。
`socket`这个函数类似于`open`文件，打开的是一个用于通信的文件，它需要传入的不是文件路径，而是指定通信协议和数据传输方式。

```c
include<sys/socket.h>
int socket = socket(int domain, int type, int protocol);
```

* domain确定通信的特性，包括格式地址（各个域通常以AF_开头，意指地址族）

|    域     |    描述    |
| :-------: | :--------: |
|  AF_INET  | IPV4英特网 |
| AF_INET6  | IPV6英特网 |
|  AF_UNIX  |   UNIX域   |
| AF_UNSPEC |   未指定   |

* type确定套接字的类型，进一步确定通信特征。

|      类型      |                             描述                             |
| :------------: | :----------------------------------------------------------: |
|   SOCK_DGRAM   |              长度固定的，无连接的不可靠报文传递              |
|    SOCK_RAW    | IP协议的数据报接口（POSIX.1中为可选）(直接访问IP，用于自定义通信协议) |
| SOCK_SEQPACKET |        长度固定、有序、可靠的面向连接<b>报文</b>传递         |
|  SOCK_STREAM   |         有序、可开、双向的面向连接<b>字节流</b>传递          |

* `protcol`通常是0，表示按给定的域和套接字类型选择默认协议。如果`domain`是`AF_INET`并且`type`是`SOCK_STREAM`则协议为`TCP`。如果`domain`是`AF_INET`并且`type`是`SOCK_DGRAM`则协议为`UDP`。
当然也可以进行指定。

### 字节序

字节序是一个处理器的架构特性，表示了字节排布的顺序。字节序有大端和小端之分，大端表示把数据的高字节保存在内存的低地址，小端表示把数据的高字节保存在内存的高地址。如果不确定数据的字节序读取的数据则完全是错的，因此数据在传输时要么确定自己的字节序不会变，例如网络中以大端传输，要么就得在数据中带上自己的字节序。

<b>这里有个有意思的事，很多经典书籍比如Unix环境高级编程在介绍字节序是会说Mac是大端的，然而我实测却是小端，搞的我怀疑人生。。后来经过查询资料，原来早年的苹果电脑是PowerPC架构是大端的，而如今的x86架构系统是小端的。所以万事还是得自己实测一下比较好。。。</b>

我们知道内存中地址顺序如下：

低地址------->高地址

对于数据`a = 0x04030201`,如果处理器支持大端字节序, 则内存为`04 03 02 01` ; 如果处理器支持小端字节序, 则内存为`01 02 03 04 `。以mac为例如下:

```shell
(lldb) x &a
0x7ffee01851bc: 01 02 03 04 
```

以下是两种判断字节序的方法:
1.将数字转成char *地址，取首字节的值进行判断
2.通过联合体进行判断

```c
union test {
    int a;
    char b;
};
int main(int argc, const char * argv[]) {
    test t;
    t.a = 1;
    if (t.b == 1) {
        cout << "小端" << endl;
    } else {
        cout << "大端" << endl;
    }
    return 0;
    uint32_t b = 0x04030201;
    char *ptr = (char *)&b;
    cout << ptr[0] << endl;
    if (ptr [0] == 0x04) {
        cout << "大端" << endl;
    } else {
        cout << "小端" << endl;
    }
    return 0;
}
```

由于网络通信中都是以大端形式，因此socket也提供了一些字节序转换函数:

```c
#include <arpa/inet.h>
//返回以网络字节序表示的32位整型数
htonl(x) 
//返回以网络字节序表示的16整型数
htons(x) 
//返回以主机字节序表示的32整型数
ntohl(x) 
//返回以主机字节序表示的16整型数
ntohs(x) 
```

关于字节序更多的内容，可以参考[阮一峰老师的博客](https://www.ruanyifeng.com/blog/2016/11/byte-order.html)。

### 地址格式分析

由于在C中没有类与对象，所以往往以结构体表示一类信息，在`socket`编程中常用一下几个结构体来表示地址信息：

#### `sockaddr_in`

`sockaddr_in`是一个用于表示`socket`地址的结构体，它可以传递IP和端口号，以及制定协议族。<b> 必须在初始化时全部通过`bzero`把填充字段置为0， 地址必须有效为当前机器的地址。端口号必须不小于1024。地址必须与创建套接字时所支持的格式匹配</b>。</b>

```c
struct sockaddr_in {
	__uint8_t       sin_len; //sin_len 默认为0
	sa_family_t     sin_family;//sin_family 协议族
	in_port_t       sin_port;//sin_port 端口
	struct  in_addr sin_addr;//sin_addr 网络地址, 一般用为`sin_addr.s_addr`
	char            sin_zero[8];//sin_zero 填充字段，需要置为0
};
    //设置一个服务器地址
    struct sockaddr_in server_addr;
    //初始化数据为0
    bzero(&server_addr, sizeof(server_addr));
    //IPV4网络
    server_addr.sin_family = AF_INET;
    //INADDR_ANY表示不需要绑定特定的IP，此时会使用本地主机localhost
    server_addr.sin_addr.s_addr = htons(INADDR_ANY);
		//绑定ip地址时使用inet_addr将字符串转换网络字节序的整形表示,
		//但是需要引入头文件#import <arpa/inet.h>
    //server_addr.sin_addr.s_addr = inet_addr("192.168.0.x")；
		//inet_ntoa可以把sin_addr地址转换成IP字符串
		//char *ip = inet_ntoa(server_addr.sin_addr);
    //设置端口号, htons转成网络字节序16位整形
    server_addr.sin_port = htons(8888);
```

#### `addrinfo`

`getaddrinfo`是`IPV6`上出的用户获取IP的函数，通过传入一个指定的域名或者IP获取到指定的地址信息。因此我们可以利用它对地址进行检查，也可以通过它获取`sock_addr`数据。

```c
    //use addrinfo
    struct addrinfo server_addr1;
    struct addrinfo *res; //返回的是一个addrinfo数组
    struct addrinfo *curr; //一个指针用于遍历
    char ipName[32];
    bzero(&server_addr1, sizeof(server_addr1));
    server_addr1.ai_flags = AI_PASSIVE; //作为服务端
    server_addr1.ai_family = AF_INET; //IPV4
    server_addr1.ai_socktype = SOCK_STREAM;//字节流
		//传入指定的域名，我们知道可以多个IP对应一个域名，因此结果保存在res这个数组中，如果成功返回0
    int ret1 = getaddrinfo("www.baidu.com", NULL, &server_addr1, &res);
		//说明传入的地址是错误的
    if (ret1 != 0) {
        SERVER1_LOG("invalid address ");
        return false;
    }
		//遍历数组，依次取出地址信息
    for (curr = res; curr != NULL; curr = curr->ai_next) {
        inet_ntop(AF_INET,&(((struct sockaddr_in *)(curr->ai_addr))->sin_addr), ipName, curr->ai_addr->sa_len);
        SERVER1_LOG("ip: %s", ipName);
    }
		//释放res
    freeaddrinfo(res);
//log
2019-12-14 01:33:35 [SERVER1]  ip: 14.215.177.39
2019-12-14 01:33:35 [SERVER1]  ip: 14.215.177.38
```

我们详细看一下`addrinfo`这个结构体，可以看到除了通用的一些设置，内部还有个指针指向了`sockaddr`,这个指针指向的便是地址信息。而由于一个域名可能对对应多个IP，这个结构体还是链式的，用`ai_next`指向下一个IP的信息。

```c
		struct addrinfo {
      //AI_PASSIVE：被动用于server bind
      //AI_CANONNAME：用于返回主机的名词
      //AI_NUMSOCK_STREAM：地址为字符串
        int    ai_flags;    /* AI_PASSIVE, AI_CANONNAME, AI_NUMSOCK_STREAM
        //设置IPV4还是IPV6协议族
        int    ai_family;    /* PF_xxx */
      //设置socket类型，例如SOCK_STREAM
        int    ai_socktype;    /* SOCK_xxx */
      //协议类型, 0表示根据其它参数来确定
        int    ai_protocol;    /* 0 or IPPROTO_xxx for IPv4 and IPv6 */
      //地址长度
        socklen_t ai_addrlen;    /* length of ai_addr */
      //主机名
        char    *ai_canonname;    /* canonical name for hostname */
      //存储地址
        struct    sockaddr *ai_addr;    /* binary address */
      //可能有多个IP对应了一个地址，因此这是一个链式的结构
        struct    addrinfo *ai_next;    /* next structure in linked list */
    };
		//sockaddr存储了地址信息
    struct sockaddr {
      //地址长度
        __uint8_t       sa_len;         /* total length */
      //地址协议族
        sa_family_t     sa_family;      /* [XSI] address family */
      //地址数据， inet_ntop方法就是把这个数据拷到字符串中的
        char            sa_data[14];    /* [XSI] addr value (actually larger) */
    };
```

#### `ifaddrs`

`ifaddrs`用于获取当前的接口信息，它的结构和`addrinfo`很相似，也是链式的，也包含有`sock_addr`指针来表示地址信息。 `getifaddrs`函数用于获取`ifaddrs`数组，表示当前设备的所有硬件接口信息（当然也可以拿到IP信息拉啦）：

```c
struct ifaddrs {
	struct ifaddrs  *ifa_next;
  //硬件接口名称，例如wifi
	char		*ifa_name;
  //接口标志位
	unsigned int	 ifa_flags;
  //详细地址信息
	struct sockaddr	*ifa_addr;
  //存储该接口的子网掩码
	struct sockaddr	*ifa_netmask;
	//表示连接的另一段的信息
	struct sockaddr	*ifa_dstaddr;
  //保存该协议族特殊信息，通常为NULL
	void		*ifa_data;
};
```

以一个例子来看下如何通过`getifaddrs`获取本机所有接口信息，假如我们对接口有要求，比如我们限制只能通过`WIFI`连接，则我们就可以过滤出接口名包含`en`的接口。然后获取其地址，这样就可以指定通过`WIFI`网卡的接口来进行通信了：

```c
    struct ifaddrs *addrs = 0, *firstAddr = 0;
    getifaddrs(&firstAddr);
    addrs = firstAddr;
    struct sockaddr *WIFI_addr = nil;
    while (addrs) {
        if (!addrs->ifa_flags
            || !addrs->ifa_addr
            || !addrs->ifa_name) {
            addrs = addrs->ifa_next;
            continue;
        }
        //我们只关注支持IPV4和IPV6协议族的接口
        if (addrs->ifa_addr->sa_family != AF_INET && addrs->ifa_addr->sa_family != AF_INET6) {
            addrs = addrs->ifa_next;
            continue;
        };
        SERVER1_LOG("ifa_name: %s", addrs->ifa_name);
      //判断是否为WIFI的接口
        if (strlen(addrs->ifa_name) >= 2 && addrs->ifa_name[0] == 'e' && addrs->ifa_name[1] == 'n') {
            SERVER1_LOG("find WIFI interface: %s", addrs->ifa_name);
            WIFI_addr = addrs->ifa_addr;
            break;
        }
        addrs = addrs->ifa_next;
    }
    freeifaddrs(firstAddr);

Logs：
2019-12-15 02:41:12 [SERVER1]  ifa_name: lo0
2019-12-15 02:41:12 [SERVER1]  ifa_name: lo0
2019-12-15 02:41:12 [SERVER1]  ifa_name: lo0
2019-12-15 02:41:12 [SERVER1]  ifa_name: en5
2019-12-15 02:41:12 [SERVER1]  find WIFI interface: en5
```

这些都是我mac电脑上的接口，接口顾名思义就是和外界通信的端口。由于我再查找`WIFI`之后中断了，更多的介绍可以参考`stackoverflow`上这个[回答](https://superuser.com/questions/267660/can-someone-please-explain-ifconfig-output-in-mac-os-x)。

### 关键字

#### 获取错误

万事开头难，无论学啥第一应该会的是看懂错误，否则出了问题会一脸懵逼。`socket`提供了一个`errno`宏命令用于传递错误码，实际它是一个指向`int *`指针函数的`int`型变量。当出现错误时，`socket`标注库会把错误传递进去，因此它只会记录最后一次错误。开发者可以通过`errno`获取到当前的错误码。同样`socket`标准库还提供了一些方法来打印错误：

```c
//将errno转换成字符串
strerror(errno);
//会打印 [socket]: xxx。 xxx表示具体的错误原因
perror('[socket]');
```

#### 绑定地址

对于服务器需要绑定一个特定的地址，以使客户端可以通过该地址与之通信。地址必须包含指定的目标地址，目标端口号，如果成功会返回0。否则可以打印`errno`获取失败原因。

```c
//将套接字与地址绑定,成功返回0,否则-1. 端口必须不小于1024
bind(int, const struct sockaddr *, socklen_t)
//获取socket绑定的地址
getsockname(int, struct sockaddr *restrict, socklen_t *restrict)
//如果socket已经和对方连接，调用下面来获取对方地址
getpeername(int, struct sockaddr *restrict, socklen_t *restrict)
```

#### 监听连接请求

服务器调用`listen`可以宣告何时可以接受连接请。`count`指该进程所要入队的连接请求数量。
一旦队列满了则系统会拒绝剩下的请求。一旦调用了`listen`，服武器就可以开始接受连接请求了。如果成功会返回0。否则可以打印`errno`获取失败原因。

```c
//sock_fd表示server
//count表示要最大监听的连接，超出之后，客户端的连接会被refused
listen(int sock_fd, int count);
```

#### 接受连接

服务器调用`accept`来获得连接请求并建立连接。该描述符连接到调用`connect`的客户端。如果不关心客户端表示，可以将addr和len设置为`NULL。`对于阻塞式`TCP`如果没有连接到来，`accept`会一直阻塞，只到一个连接事件发生。如果`sock_fd`处于非阻塞模式，`accept`会立刻返回。通常由于三次握手的原因可能会返回-1,并将`errno`设置为`EAGAIN`或`EWOULDBLOCK`。

```c
//接受一个到server_socket代表的socket的一个连接
//如果没有连接请求,就会阻塞等待连接的到来
//accept函数返回一个新的socket,这个socket(new_server_socket)用于同连接到的客户的通信
int new_client_socket = accept(server_socket,(struct sockaddr*)&client_addr,&length);
```

#### 请求连接

`connect`中的地址是想与之通信的服务器地址，以及地址长度。 如果成功会返回0，否则返回-1，此时可以通过`errno`获取失败的原因。`connect`函数通常会阻塞直到返回，这个时间大约为75s到几分钟，因此为了提高性能，通常设置为非阻塞模式如果是非阻塞，会立即返回结果，有可能因为三次握手的原因导致产生`EINPROGRESS`错误。一般此时利用`select`等待之后检测套接字是否可写，如果可写即认为连接成功。

```c
connect(int, const struct sockaddr *, socklen_t)
```

#### 数据发送

`send`用于发送数据到系统进程缓冲区，然后写入链路。当启用了`Nagle`算法，会在缓冲区进行小包的重组。由于缓冲区并不一定大小完全可用，`send`的返回值表示真正写入的字节。

```c
//发送数据，成功返回发送的字节数，否者返回-1.
//如果是报文设限的协议，则单个报文超出最大尺寸时，send失败并且errno置为EMSGSIZE。
//对于字节流协议，send会阻塞直到整个数据被传输
ssize_t send(sock_fd, const void *buffer, size_t length, int flags);

//对于无连接的，需要sendto指定地址
ssize_t sendto(int sock_fd, const void *buffer, size_t length, int flags, const struct sockaddr * dest_addr, socklen_t des_len);

//使用不止一个的选择来通过套接字发送数据。
sendmsg(int sock_fd, const struct msghdr *msg, int flags);
struct msghdr {
	void		*msg_name;	/* [XSI] optional address */
	socklen_t	msg_namelen;	/* [XSI] size of address */
	struct		iovec *msg_iov;	/* [XSI] scatter/gather array */
	int		msg_iovlen;	/* [XSI] # elements in msg_iov */
	void		*msg_control;	/* [XSI] ancillary data, see below */
	socklen_t	msg_controllen;	/* [XSI] ancillary data buffer len */
	int		msg_flags;	/* [XSI] flags on received message */
};
```

#### 数据接收

`recv`是用户进程去系统进程的缓存区去取数据，可以通过`buffer_len`参数指定接收长度，实际收到的长度以系统缓冲区的缓存大小为准，并且不大于指定的接收长度。

```c
//若无可用消息或对方已经按序结束则返回0，若出错返回-1（对非阻塞式当系统进程的没缓存时也会返回-1）
//若发送者已经调用shutDown结束传输，则返回0.
//对于阻塞式socket通常recv没有数据可用时会阻塞等待。
//对于非阻塞式
ssize_t recv(int sock_fd, void *buffer, size_t buffer_len, int flags);
//若要指定发送者,通常用于无连接的套接字
ssize_t recvfrom(int sock_fd, void *buffer, size_t buffer_len, int flags, struct sockaddr *restrict addr, socklen_t *restrict addr_len);
//若要放入多个缓冲区
ssize_t recvmsg(int sock_fd, struct msghdr *msg, int flags);
```

#### 关闭连接 

关闭连接可以通过`close`和`shutdown`：

```c
int shutdown(int socketfd, int how);
close(int socket);
```

`shutdown`函数可以中断套接字套接字通信是双向的，因此调用`shutdown`可以停止输入/输出。how描述关闭哪一端,`SHUT_RD`关闭读取，`SHUT_WR`关闭写入，`SHUT_RDWR`同事关闭读取写入端。可以只关闭一个方向的传输。

`close`关闭对文件或者套接字的访问，并且释放该描述符以便重新使用。<b>`close`会将套接字的引用计数减一，只有所有引用全部被关闭后才会释放Socket。通常在close后socket并不会立刻关闭，此时会处于`TIME_WAIT`阶段，以防接收不到最后的包。</b>

#### select

[GUN官方文档](https://www.gnu.org/software/libc/manual/html_node/Waiting-for-I_002fO.html)如此描述`select`函数:

```
You cannot normally use read for this purpose, because this blocks the program until input is available on one particular file descriptor; input on other channels won’t wake it up. You could set nonblocking mode and poll each file descriptor in turn, but this is very inefficient.

A better solution is to use the select function. This blocks the program until input or output is ready on a specified set of file descriptors, or until a timer expires, whichever comes first. This facility is declared in the header file sys/types.h.
```

前面的一些关键字都是阻塞式的，会阻塞当前的线程，直到事件发生，同一时刻只能给一个客户提供服务。因此要么阻塞当前的线程，要么就为每一个客户端启用一个线程进行数据的传输，可以参考[第一个例子](#demo1)。

另一种方式就是采用非阻塞属性，关于[属性](#property)看一看下一节。

而`select`是允许程序监听多个文件描述符，直到一个或多个文件描述符可以用于I/O操作，则会通知到服务端进程。这样可以使一个服务端进程对应多个客户端进程。`select`会返回发生修改的文件描述符个数。如果为0表示没有文件描述符发生改变，如果小于0则可能是哪里出错了。

```c
//max_fdp，指集合中所有文件描述符的范围，即所有文件描述符最大值加1。
//readfds, 表示监视文件描述符可读状态，如果有文件可读，则select返回大于0.可以设置为NULL表示不关心可写状态
//writefds, 表示监视文件可写状态，如果有文件可写，则select返回大于0.可以设置为NULL表示不关心可读状态
//errorfds, 监视文件错误状态。
//timeout, 超时时间，如果没有可读文件，也没有可写文件，则到达超时直接返回
//如果传入NULL，表示为阻塞模式，直到文件描述符发生变化为止。
//如果设置时间为0，则表示为非阻塞，无论描述符是否有变化，直接返回。
//如果设置时间大于0，则在超时时间内阻塞，如果事件发生则返回，否则超时之后一定返回0。
//当超时之后selct会修改timeout参数值，因此多次使用时每次都要传入新的。
int select(int maxfdp, fd_set* readfds, fd_set* writefds, fd_set* errorfds, struct timeval* timeout);

//pselect的超时时间可以设置到毫微秒级别
//pselect可以设置 sigset_t
//pselect不会修改传入的timeout参数值。
int pselect(int nfds, fd_set *readfds, fd_set *writefds,
            fd_set *exceptfds, const struct timespec *timeout,
            const sigset_t *sigmask);
```

首先需要理解`select`模型，`select`采用位域来表示文件描述符，可以看到实际是一个以32个bit对齐的数组。每一个`fd_bit`对应一个文件描述符。我们知道在Unix系统中，一切都是文件，所以本质上还是对文件的监测。文件描述符个数是固定定义的，并且苹果官方注释不要重定义文件描述符大小。


```c
#ifdef FD_SETSIZE
#define __DARWIN_FD_SETSIZE     FD_SETSIZE
#else /* !FD_SETSIZE */
#define __DARWIN_FD_SETSIZE     1024
#endif /* FD_SETSIZE */
#define __DARWIN_NBBY           8                               /* bits in a byte */
#define __DARWIN_NFDBITS        (sizeof(__int32_t) * __DARWIN_NBBY) /* bits per mask */
#define __DARWIN_howmany(x, y)  ((((x) % (y)) == 0) ? ((x) / (y)) : (((x) / (y)) + 1)) /* # y's == x bits? */

typedef struct fd_set {
	__int32_t       fds_bits[__DARWIN_howmany(__DARWIN_FD_SETSIZE, __DARWIN_NFDBITS)];
} fd_set;
```

系统还定义了一些宏命令用于操作`fd_set`:

```c
void FD_ZERO(fd_set *fdset);            /* 初始化文件描述符*/
void FD_SET(int fd, fd_set *fdset);     /* 把socket中的文件描述符导入fdset */
void FD_CLR(int fd, fd_set *fdset);     /* 把socket中的文件描述符从fdset中清除 */
void FD_ISSET(int fd, fd_set *fdset);   /* 判断某一个文件描述符集是否有变化*/
```
<b>文件描述符最大总~~字节数~~(bit数，因为`__DARWIN_NFDBITS`乘了8)为1024。也就是说最多数组中最多只有32个数，但是并不表示只有32个文件描述符</b>。
可以看到`FD_ISSET`的实现为:

```c
/* This inline avoids argument side-effect issues with FD_ISSET() */
static __inline int
__darwin_fd_isset(int _n, const struct fd_set *_p)
{
	return _p->fds_bits[(unsigned long)_n / __DARWIN_NFDBITS] & ((__int32_t)(((unsigned long)1) << ((unsigned long)_n % __DARWIN_NFDBITS)));
}
```
也就是每个`socket`只占用一个bit来进行它的文件描述符，因为只需要表示可读/不可读。可写/不可写。我们做个测试:

```c
fd_set read_sets;
FD_ZERO(&read_sets);
FD_SET(server_socket, &read_sets);
```
初始化一个socket只会，传入一个fd_set。此时打印fd_set可以看到:

```shell
(lldb) po server_socket
5
(lldb) p read_sets
(fd_set) $1 = {
  fds_bits = {
    [0] = 32
    [1] = 0
    ...
    [31] = 0
    }
  }
```
第五个bit被置为了1。也就是说此时把`socket`（文件描述符）注册到`fd_set`中，这里其实也不难理解为啥要分32个一组，主要也是为了方便查找和管理。
而`FD_ISSET`表示，fdset指向的文件描述符集是否包含了指定的`socket`（文件描述符）。其实就是获取哪一个bit是否为真。因为在调用`select`函数之后，发生变化的文件描述符将会被保留，其余字段都将变为0。<b>由于每次select会重置监控句柄，所以每次select都需要先传入一个新的`fd_sets`</b>。

由于文件描述符上限为1024。也就是说socket（文件描述符）最多只能有1024个存在，即最多只能创建1024个TCP连接。
当然也可以通过一些手段去除这个限制，感兴趣可以参考[这篇文章](https://zhuanlan.zhihu.com/p/65810324)。

<b>这里有一些重要的点是：1，`select`的超时时间并不是从被调用开始的，调用所需的时间也不算在倒计时里，因此如果系统繁忙，则超时会在获得处理机时才开始，非常不准确；2.当`select`执行完成时会修改`timeout`参数，由于传入的是指针，因此如果需要重复使用必须重新设置该参数。</b>
关于`fd_set`更多详细的信息可以参考[Liunx官网](https://linux.die.net/man/3/fd_set)。

### TCP状态转换模型

这一段摘自维基百科对于TCP的[介绍](https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_termination)，偷个懒，直接搬运过来了:

断开连接的4次握手，主要是为了保证两端都能close。需要注意的是被动close的一方进入`CLOSE_WAIT`状态等待`LAST_ACK`，当收到`LAST_ACK`就会完全close掉。而主动close一方进入`TIME_WAIT`状态以确保最后的数据能收发干净，并且不影响新的连接。由于此时被动close方已经完全关闭了，因此主动连接方只能等待一段时间自己关闭，而这个时间一般为2*MSL，[MSL](https://en.wikipedia.org/wiki/Maximum_segment_lifetime)指的是TCP数据段在网络系统中存活最大时间，一般为2分钟。

![image-20191215160911678](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/Socket/image1.png?raw=true)

由于`TIME_WAIT`的存在，导致在`close`之后系统资源并不能立即释放，如果存在大量连接，可能导致系统资源持续的高占用。并且由于timeout的存在导致服务端不能实时重启。 同样的服务端主动close也会导致大量客户端进入`CLOSE_WAIT`状态。

可以看到看到`TIME_WAIT`其实是一种正常的`TCP`的优化手段，并且时间基本是固定的不会再受其它的原因影响，只是大量的`TIME_WAIT`才会导致问题，因此需要时只需要避免`TIME_WAIT`的存在即可。

但是如果长时间处于`CLOSE_WAIT`则说明B端其实是有问题，要么是没有发出`FIN`，要么就是没有收到A端的`ACK`，说明程序有问题或者链路有问题，并且会被一直卡着，而不像`TIME_WAIT`到时间自己关掉。所以更需要重视的`CLOSE_WAIT`问题。

### <span id="property">Sockect属性设置</span>
`setsockopt`和`getsockopt`函数用于设置获取socket的属性，

```c
#include <sys/types.h>
#include <sys/socket.h>
//s, 需要设置的socket
//level， 参数级别，即协议层级，大多数时候为SOL_SOCKET表示一个socket-level的参数
//optname， 参数名
//optval， 参数值
//optlen， 参数长度
//返回值, 0表示成功， -1表示失败，且此时errno会返回错误。
int setsockopt( int s,
                int level,
                int optname, 
                const void * optval,
                socklen_t optlen );
```

是一个系统层的`socket`参数。所有的参数项可以参见[gnu的官网介绍](https://www.gnu.org/software/libc/manual/html_node/Socket_002dLevel-Options.html#Socket_002dLevel-Options).

我们只关注几个常用的参数:

#### `SO_LINGER`

```c
struct linger l;
//等待时间
l.l_linger = 0;
//是否激活linger，一般设置为1才生效
l.l_onoff = 1;
setsockopt(_connectSock, SOL_SOCKET, SO_LINGER, &l, sizeof(struct linger));
```

我们知道`socket`在主动断开连接后会保留将近2分钟的`TIME-WAIT`，所以如果存在大量的短连接，会导致大量的`TIME-WAIT`，导致占用系统资源。通过设置`SO_LINGER`可以限制`close`之后释放资源的时间，会把`sendbuffer`中未发完的数据丢弃，并且发送的是`RST`信息，而不是正常的四次握手断开`TCP`连接，因此不会进入`TIMEWAIT`状态。这样做最大的缺点就是可能导致最后的包没有收到，所以如果需要传输的资源是大量小文件，而且不应该丢失就不要设置这个属性。

#### `SO_REUSEADDR`

在实验中，发现当我重启服务端，重复`bind`一个端口时会报错。

```c
Server Bind port : 2234 Failed!, errno: Address already in use 
```

因为我们使用的`TCP`可靠连接，因此只允许有一个`socket`绑定特定的IP和端口号。在重启服务进行端口`bind`时因为`TIMEWAIT`当前端口仍然被上一个`socket`绑定所以报错。而设置`SO_REUSEADDR`参数既可以重用当前的地址，但是属性必须在调用`bind`函数之前设置：

```c
int value = 1;    
setsockopt(server_socket, SOL_SOCKET, SO_REUSEADDR, &value, sizeof(int));
```

不过`SO_REUSEADDR`也有风险，调用之后，前一个`socket`未接收到数据将会被丢弃，但是如果等待`TIMEWAIT`结束又可能导致新连接不能成功，这段时间内的数据丢失，所以需要根据实际情况来做选择是否使用该属性。

#### `TCP_NODELAY`

在`TCP`中，有一个叫[`Nagle`的算法](https://en.wikipedia.org/wiki/Nagle%27s_algorithm)。大概就是就是一个delay的算法，会把一些小块的包合并以提高传输效率。我们知道每一个包都有包头，所以合并小包其实可以避免这一部分的浪费，

一下两张图都摘自:https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_termination。 图侵删。

![image-20191215171134752](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/Socket/image2.png?raw=true)

![image-20191215171222496](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/Socket/image3.png?raw=true)

在IPV4，每一个包头要站32个字节，IPV6中每一个包头要占将近60个字节。 对于早期互联网网速并不发达时，这个带宽消耗还是不小的，因此诞生了这个算法。

而如今网速早已不再存在这种问题，`Nagle`算法反而导致数据延迟，因此我们需要设置属性避免这个问题:

```c
int value = 1;
setsockopt(server_socket, IPPROTO_TCP, TCP_NODELAY, &value, sizeof(int));
```

#### `SO_NOSIGPIPE`

我们知道在Unix系统中利用信号机制进行进程间的通信，信号触发进程的中断，并且触发信号处理函数，比如如下`SIGPIPE`信号，表示对一个已经关闭的`socket`写入数据，会导致App进程crash, 我们以Demo1为例子，只启动client，然后直接朝fd里写数据，可以看到log如下：

```c
[Client1] Connect failed, errno: Connection refused

Message from debugger: Terminated due to signal 13
```

13正是`SIGPIPE`的实际值。在实际开发中，难保不会发生在写数据时候`socket`发生了`close`而导致程序崩溃一般在App端可以如下处理:

```objective-c
#import <signal.h>
void handleSignal(int signal) {
}
//捕获SIGPIPE并执行handleSignal函数
signal(SIGPIPE, &handleSignal);        
//忽略SIGPIP信号
signal(SIGPIPE, SIG_IGN);        
```

点开signal的声明可以看到函数的原型，这里非常有意思，值得思考一下：

```c
/*
 * For historical reasons; programs expect signal's return value to be
 * defined by <sys/signal.h>.
 */
__BEGIN_DECLS
    void(*signal(int, void (*)(int)))(int);
__END_DECLS
```

首先`__BEGIN_DECLS`，`__END_DECLS`宏命令是gun的buildin宏，为了避免C++对符号进行重组导致`Undefined Symbol`错误。在[GUN官方文档](http://cmd.inp.nsk.su/old/cmd2/manuals/gnudocs/gnudocs/libtool/libtool_36.html)中可以看到详细介绍，我们在查看一些系统级头文件时经常会看到这个宏。

然后`signa`函数的声明非常有意思，我也是看了一会才看明白这里指针的艺术。

这里先分析下：

首先先看最里层有一个函数，它是一个函数的指针，参数为int型，无返回值。假如我们以func称呼它：

```c
typedef void (*func)(int) ;
```

然后像反向剥洋葱一般展开，可以看到再外一层是一个命名为signal的函数指针，它有两个参数，一个为int型，另一个是func1函数指针，由于它只是函数原型中的一个声明，因此并没有命名：

```c
signal(int, func);
```

把signal作为一个指针再向外展开，可以看到其实是一个signal指针函数，它的参数为int型，返回值为空。

```c
void (*signal)(int);
```

因此可以改写成:

```c
func signal(int, func);
```

那么我们可以猜测一下系统内核的实现大概是:

```c
static func handlers[100];
//当出现异常时调用raisesignal函数触发
void raisesignal(int signal) {
    func current_handler = handlers[signal];
    if (current_handler) {
        current_handler(signal);
    } else {
        printf("uncatched signal: %d\n", signal);
    }
}

void(*signal1(int a, void (*func)(int)))(int) {
    handlers[a] = func;
    return 0;
}

```

则在App端我们捕获`SIGPIPE`信号:

```objc
//在.m里我们模拟cacth signal
void signalHandler(int signal) {
    printf("catch signal: %d\n", signal);
}
signal1(SIGPIPE, &signalHandler)
```

则我们模拟一下出现异常信号，先触发添加捕获的`SIGPIPE`，在随意触发一个`SIGKILL`,则在console可以看到log如下：

```c
catch signal: 13
uncatched signal: 9
```

9恰好就是未进行捕获的`SIGKILL`。13则已经被捕获到了。

当然另一种方式，在设置属性避免`socket`发出`SIGPIPE`信号，不过这种属性只会对`send`函数生效。

```c
int value = 1;
setsockopt(server_socket, SOL_SOCKET, SO_NOSIGPIPE, &value, sizeof(int));
```

#### `O_NONBLOCK`

`O_NONBLOCK`表示非阻塞，可以通过设置它实现非阻塞`socket`。`fcntl`函数用于操作`socket`，`F_GETFL`参数表示获取参数，`F_SETFL`参数用户设置参数，在系统源码中为了提高效率，很多都采用了位域运算来表示信息。

```c
   //先利用fcntl函数获取socket参数
   long arg;
   if( (arg = fcntl(server_socket, F_GETFL, NULL)) < 0) {
        SERVER1_LOG("fcntl failed: (%s)\n", strerror(errno));
        close(server_socket);
        return NO;
    }
    //设置为非阻塞式
    arg |= O_NONBLOCK;
    //如果要设置为阻塞式
   //arg &= ~O_NONBLOCK;
    if( fcntl(server_socket, F_SETFL, arg) < 0) {
        SERVER1_LOG("fcntl failed: (%s)\n", strerror(errno));
        close(server_socket);
        return NO;
    }
```

## <span id="demo1">实现简单的服务端与客户端</span>

[Demo地址]()
首先以实现一个简单的阻塞式的服务端与客户端来理解`socket`的工作原理:

## 服务端

对于服务端则工作流程为:

![image-20191215211849500](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/Socket/image4.png?raw=true)

由于我们使用的是`accept`函数会阻塞当前线程，它会阻塞线程直到连接事件发生。因此不能在主线程中处理`accept`,同样为了不阻塞已连接线程的数据传输，应该单独为其启用一个线程进行数据的传输。所以当客户端较多时，多线程带来的性能损耗非常大。

为了和连接的客户端通信，我们需要把连接进来的客户端套接字保存下来。然后在收发数据时根据指定的套接字就可以和不同的客户端通信。当客户端断开连接时，把客户端套接字移除掉,这样一个简易的聊天系统就有了雏形：

#### 初始化

<details>
    <summary>代码</summary>


```objective-c
- (BOOL)initServer {
    //设置一个服务器地址
    struct sockaddr_in server_addr;
    //初始化数据为0
    bzero(&server_addr, sizeof(server_addr));
    //IPV4网络
    server_addr.sin_family = AF_INET;
    //INADDR_ANY表示不需要绑定特定的IP，此时会使用本地主机localhost
    server_addr.sin_addr.s_addr = htons(INADDR_ANY);
    //设置端口号
    server_addr.sin_port = htons(PORT);

    //初始化socket, 传输方式为TCP
    int server_socket = socket(AF_INET, SOCK_STREAM, 0);
    if (server_socket < 0) {
        printf("Create Socket Failed!, errno: %s\n", strerror(errno));
        return false;
    }

    //设置参数
    struct linger l;
    l.l_linger = 0;
    l.l_onoff = 1;
    int intval = 1;
    setsockopt(server_socket, SOL_SOCKET, SO_LINGER, &l, sizeof(struct linger));
    //设置可以重用地址，否则端口不能重复绑定
    setsockopt(server_socket, SOL_SOCKET, SO_REUSEADDR, &intval, sizeof(int));
    //设置取消`Nagle`算法
    setsockopt(server_socket, IPPROTO_TCP, TCP_NODELAY, &intval, sizeof(int));
    //设置取消`SIGPIPE`信号
    setsockopt(server_socket, SOL_SOCKET, SO_NOSIGPIPE, &intval, sizeof(int));
    
    fd_set read_sets;
    FD_ZERO(&read_sets);
    FD_SET(server_socket, &read_sets);
    
    //使socket绑定到对应的地址
    int ret = bind(server_socket, (struct sockaddr *)&server_addr, sizeof(server_addr));
    if (ret != 0) {
        printf("Server Bind port : %d Failed!, errno: %s \n", PORT, strerror(errno));
        return false;
    }

    //设置最大连接数，并开始监听
    ret = listen(server_socket, MAX_LISTEN_COUNT);
    if (ret != 0) {
        printf("Server Listen Failed!, errno: %s\n", strerror(errno));
        return false;
    }
    server_socket_fd = server_socket;
    //为了不阻塞主线程，把socket的监听放在新奇的线程中
    [NSThread detachNewThreadSelector:@selector(run) toTarget:self withObject:nil];

    return true;
}
```
</details>

<b>需要注意由于默认构建的是阻塞式socket，因此会卡在accept函数，直到接收到第一个连接位置，因此应该把accept函数放在另一个线程，避免阻塞主线程。</b>

#### 建立连接

<details>
    <summary>代码</summary>


  ```objective-c
- (void)run {
    isClosed = NO;
    printf("Server start......\n");
    //为了避免线程终止，需要一直run
    while (!isClosed && server_socket_fd != -1)
    {
        //定义客户端的socket地址结构client_addr
        struct sockaddr_in client_addr;
        socklen_t length = sizeof(client_addr);

        //接受一个到server_socket代表的socket的一个连接
        //如果没有连接请求,就会阻塞在这里等待第一个连接到来
        //accept函数返回一个新的socket,这个new_server_socket即连接的客户端socket
        int new_client_socket = accept(server_socket_fd, (struct sockaddr *)&client_addr, &length);
        if (new_client_socket < 0) {
            printf("Server Accept Failed, errno: %s!\n", strerror(errno));
            break;
        }
        [connectedClients addObject:@(new_client_socket)];
        if (self.delegtate && [self.delegtate respondsToSelector:@selector(server:didUpdateConnectedClients:)]) {
            [self.delegtate server:self didUpdateConnectedClients:connectedClients.copy];
        }
        isConnected = true;
        printf(" one client connted: %d\n", new_client_socket);
        //单个socket的通信应该放在自己的线程中
        [NSThread detachNewThreadSelector:@selector(readData:)
                                 toTarget:self
                               withObject:[NSNumber numberWithInt:new_client_socket]];
    }
    //走到这里说明需要关闭监听用的socket
    close(server_socket_fd);
    printf("Close server\n");
}
  ```
</details>

首先梳理一下这里的逻辑，有一个特点的线程监听连接事件。在while循环里，如果没有连接事件发生，就会阻塞在`accept`这里，如果有一个客户端发起连接，则`accpt`返回客户端`socket`。并且继续执行，开启一个新的线程用于该客户端数据的读取。而循环再一次执行，再一次阻塞在`accept`处，等待下一个连接事件发生。
<b>需要注意的是`accept`函数会阻塞，直到一个连接到来并返回客户端套接字，此时我们可以会这个客户端套接字开一个线程进行与之相关的数据传输，注意这个客户端套接字是通信的唯一标识符。为了实现与多个客户端通信，我们应该在客户端连接时保存下客户端套接字。</b>

#### 数据接收

<details>
  <summary>代码</summary>


  ```objective-c
// 读客户端数据
- (void)readData:(NSNumber *)clientSocket {
    char buffer[MAX_BUFFER_SIZE];
    int intSocket = [clientSocket intValue];
    
    //这里我们规定当客户端发来"-"时表示需要终止连接
    while (buffer[0] != '-' && server_socket_fd != -1 && [connectedClients containsObject:clientSocket]) {
        bzero(buffer, MAX_BUFFER_SIZE);
        //接收客户端发送来的信息到buffer中
        size_t recv_length = recv(intSocket, buffer, MAX_BUFFER_SIZE, 0);
        SERVER_LOG("recv length : %ld\n", recv_length);
        if (recv_length > 0) {
            SERVER_LOG("client:%s\n", buffer);
            if (self.delegtate && [self.delegtate respondsToSelector:@selector(server:didReceiveBuffer:length:socket:)]) {
                [self.delegtate server:self didReceiveBuffer:buffer length:(int)recv_length socket:clientSocket.intValue];
            }
        }
        else if (recv_length == 0) {
            SERVER_LOG("Client disconnected\n");
            [self closeClient:clientSocket];
        }
        else {
            SERVER_LOG("receive data error: %s\n", strerror(errno));
        }
        
    }
    //关闭与客户端的连接
    SERVER_LOG("client:close\n");
    close(intSocket);
}
//当客户端断开是停掉客户端，并移除客户端记录
- (void)closeClient:(NSNumber *)client {
    int client_fd = [client intValue];
    close(client_fd);
    [connectedClients removeObject:client];
  	//更新连接状态
    if (self.delegtate && [self.delegtate respondsToSelector:@selector(server:didUpdateConnectedClients:)]) {
            [self.delegtate server:self didUpdateConnectedClients:connectedClients.copy];
        }
}

  ```
</details>

先分析下逻辑，由于recv函数会阻塞直到接收到数据才返回数据的长度，因此需要专门有一个线程处理当前的客户端发送的数据。使用while循环保持线程常驻。 为了减少buffer的开销，我们可以用一个buffer即可。recv函数会`memcpy`数据进buffer, 因此只需要取接收的长度的部分即是客户端发来数据。在通知给delegate时我们带上客户端socket，这样就可以知道到底是哪个客户端发来的数据。

<b>需要注意`recv`函数返回的值是实际接收的字节长度，`send`函数返回的是实际发送的字节长度。如果出错误则返回为-1。如果客户端断开，`recv`返回为0。如果出现错误返回为-1,可以通过`strerror(errno)`来打印错误。`send`和`recv`函数都可以指定接收和发送数据到对应的套接字。</b>

#### 数据发送

<details>
    <summary>代码</summary>

```objc
- (void)sendMsg:(NSString *)msg toSocket:(int)client_fd{
    if (![_connectedClients containsObject:@(client_fd)]) {
        SERVER_LOG("client invalid");
        return;
    }
    size_t length = msg.length;
    const char *buffer = [msg cStringUsingEncoding:NSUTF8StringEncoding];
    send(client_fd, buffer, length, 0);
}
```

</details>

`send`时指定一个客户端`socket`进行发送即可发送到指定的客户端。

### 客户端

客户端与服务端很相似，也是需要需要初始化一个套接字，然后连接的指定的地址，然后进行数据传输。

![image-20191223170020709](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/Socket/image5.png?raw=true)

#### 初始化

<details>
  <summary>代码</summary>

  ```objective-c
- (void)start {
    //IPV4 字节流
    int temp_socket_fd = socket(AF_INET, SOCK_STREAM, 0);
    //创建用于internet的流协议(TCP)socket,
    //用server_socket代表服务器socket
    if (temp_socket_fd < 0) {
        printf("Create Socket Failed!, errno: %s", strerror(errno));
        return;
    }
    if ([self handleAddress] == false) {
        close(temp_socket_fd);
        CLIENT_LOG("handle host failed, errno: %s", strerror(errno));
        return;
    }
    int ret = connect(temp_socket_fd, (const struct sockaddr *)&_serverAddress, sizeof(_serverAddress));
    if (ret < 0) {
        CLIENT_LOG("Connect failed, errno: %s", strerror(errno));
        return;
    }

    CLIENT_LOG("connect success: %d", server_socket_fd);
    server_socket_fd = temp_socket_fd;
    _isConnected = true;
    [NSThread detachNewThreadSelector:@selector(receiveMsg) toTarget:self withObject:nil];

    return;
}
  ```
</details>

<b>对于客户，相对比较简单，只需要向客户端套接字发送指定，需要注意的connect是在TCP连接中需要完成三次握手，时间不定，默认connect具有75秒以上的超时时间，因此可以采用非阻塞的socket来防止。如果serverSocket不可连接则会直接返回-1， 并且`errno`会包含错误原因。</b>

<b>使用非阻塞还有一个好处是，可以在connect后进行短暂的等待，避免线程阻塞后长时间占用系统资源，通常规定一个等待时间后fd_set仍然不可用，就停止连接释放线程资源。</b>

### 使用select提高性能

为了提高效率我们，对Demo1中的client和server进行优化，采用非阻塞的方式实现。

我们利用`select`来检测连接的`socket`是否有错误:

<details>
    <summary>代码</summary>

```objective-c
- (void)run {
    while (_server_socket_fd != INVALID_SOCKET && !_isClosed) {
        //尝试accept，此时我们不关注client的地址，只保存client
        int client_socket_fd = accept(_server_socket_fd, NULL, NULL);
        if (client_socket_fd != INVALID_SOCKET) {
            fd_set error_set;
            FD_ZERO(&error_set);
            FD_SET(_server_socket_fd, &error_set);
            
            struct timeval tm;
            tm.tv_sec = 2;
            tm.tv_usec = 0;
            //由于是非阻塞,使用select来查看文件描述符的变化
            int ret = select(client_socket_fd + 1, NULL, NULL, &error_set, &tm);
            //如果文件描述符有变化
            if (ret > 0) {
                //如果client有errno
                if (FD_ISSET(client_socket_fd, &error_set)) {
                    SERVER1_LOG("client error: %s", strerror(errno));
                    close(client_socket_fd);
                }
            }
            //错误误
            else if (ret < 0) {
                SERVER1_LOG("client error: %s", strerror(errno));
                close(client_socket_fd);
                
            }
            //连接成功
            else {
                pthread_mutex_lock(&_lock);
                [_connectedClients addObject:@(client_socket_fd)];
                NSArray *tempArray = _connectedClients.copy;
                pthread_mutex_unlock(&_lock);
                
                if (self.delegtate && [self.delegtate respondsToSelector:@selector(server:didUpdateConnectedClients:)]) {
                    [self.delegtate server:self didUpdateConnectedClients:tempArray];
                }
            }
        }
    }
}
```

</details>

而在读数据时，利用`select`只需要一个线程即可，依次检测哪个socket可读，则取出缓冲区的数据。

<details>
    <summary>代码</summary>

```objective-c
- (void)startRecv {
    //当有连接的client时，尝试recv
    while (self.isRunning && _server_socket_fd != INVALID_SOCKET) {
        pthread_mutex_lock(&_lock);
        NSArray *tempArray = _connectedClients.copy;
        pthread_mutex_unlock(&_lock);
        for (NSNumber *clientValue in tempArray) {
            int tmp_cilent_socket_fd = clientValue.intValue;
            fd_set read_set;
            FD_ZERO(&read_set);
            FD_SET(tmp_cilent_socket_fd, &read_set);
            
            struct timeval tm;
            tm.tv_sec = 1;
            tm.tv_usec = 0;
            
            int ret = select(tmp_cilent_socket_fd + 1, &read_set, NULL, NULL, &tm);
            //如果文件有可读
            if (ret > 0) {
                //如果发现文件描述符可读
                if (FD_ISSET(tmp_cilent_socket_fd, &read_set)) {
                    char buffer[MAX_BUFFER_SIZE] = {0};
                    int recv_size = (int)recv(tmp_cilent_socket_fd, buffer, MAX_BUFFER_SIZE, 0);
                    SERVER1_LOG("recv data from socket: %d, length: %d, msg: %s", tmp_cilent_socket_fd, recv_size, buffer);
                    //如果正确收到了数据
                    if (recv_size > 0) {
                        if (self.delegtate && [self.delegtate respondsToSelector:@selector(server:didReceiveBuffer:length:socket:)]) {
                            [self.delegtate server:self didReceiveBuffer:buffer length:recv_size socket:tmp_cilent_socket_fd];
                        }
                    }
                    //说明客户端已经关闭
                    else if (recv_size == 0) {
                        SERVER1_LOG("client: %d, has closed", tmp_cilent_socket_fd);
                        close(tmp_cilent_socket_fd);
                        [_connectedClients removeObject:clientValue];
                        if (self.delegtate && [self.delegtate respondsToSelector:@selector(server:didUpdateConnectedClients:)]) {
                            [self.delegtate server:self didUpdateConnectedClients:_connectedClients.copy];
                        }
                    }
                }
            }
            //如果有问题
            else if (ret < 0) {
                close(tmp_cilent_socket_fd);
                pthread_mutex_lock(&_lock);
                [_connectedClients removeObject:clientValue];
                NSArray *tempArray = _connectedClients.copy;
                pthread_mutex_unlock(&_lock);

                if (self.delegtate && [self.delegtate respondsToSelector:@selector(server:didUpdateConnectedClients:)]) {
                    [self.delegtate server:self didUpdateConnectedClients:tempArray];
                }
                SERVER1_LOG("client: %d select errno: %s, close it", tmp_cilent_socket_fd, strerror(errno));
            }
        }
    }
}
```

</details>

而在发送数据时，同样先利用`select`判断`socket`是否可写，并且要判断发送的数据是否完全发送:

<details>
    <summary>代码</summary>

```objective-c
- (void)sendMsg:(NSString *)msg toSocket:(int)client_fd {
    if (!self.isRunning || _server_socket_fd == INVALID_SOCKET) {
        SERVER1_LOG("sendMsg failed! server has been closed");
        return;
    }
    
    if (![_connectedClients containsObject:@(client_fd)]) {
        SERVER1_LOG("sendMsg failed! client:%d has been closed", client_fd);
        return;
    }
    
    const char *buffer = [msg cStringUsingEncoding:NSUTF8StringEncoding];
    int length = (int)msg.length;
    int needSendLength = length;
    //检测一下client是否有可写空间
    while (1) {
        fd_set write_set;
        FD_ZERO(&write_set);
        FD_SET(client_fd, &write_set);
        
        struct timeval tm;
        tm.tv_sec = 2;
        tm.tv_usec = 0;
        
        int ret = select(client_fd + 1, NULL, &write_set, NULL, &tm);
        if (ret > 0) {
            //可写
            if (FD_ISSET(client_fd, &write_set)) {

                int send_size = (int)send(client_fd, buffer, needSendLength, 0);
                //发送成功，且socket能全部写入缓存
                if (send_size == length) {
                    SERVER1_LOG("send msg success! client: %d, length: %d, msg: %s", client_fd, send_size, buffer);
                    break;
                }
                //发送失败
                else if (send_size < 0) {
                    //表明client已经断了
                    if (errno == SIGPIPE) {
                        close(client_fd);
                        [_connectedClients removeObject:@(client_fd)];
                        if (self.delegtate && [self.delegtate respondsToSelector:@selector(server:didUpdateConnectedClients:)]) {
                            [self.delegtate server:self didUpdateConnectedClients:_connectedClients.copy];
                        }
                    }
                    SERVER1_LOG("sendMsg failed! errno: %s", strerror(errno));
                    break;
                }
                //发送长度不够，说明缓冲区不足
                else {
                    buffer += send_size;
                    needSendLength -= send_size;
                    SERVER1_LOG("Only send length: %d! errno: %s", send_size, strerror(errno));
                }
            }
        }
    }
}
```

</details>

所以最终呈现出来的效果如下，一个服务端对接3个客户端:

![image-20191223004712991](https://github.com/joey520/joey520.github.io/blob/hexo/post_images/Socket/image6.png?raw=true)



## 粘包问题

即使我们再前面设置了非阻塞，切取消了`Nagle`算法。粘包问题依然无法解决，因为`TCP`作为面向连接的字节流数据传输，可能因为网络问题在发送端的系统进程缓冲区粘包，也可能因为系统繁忙读取没跟上，在接收端的系统进程的缓冲区出现粘包。 一个最简单的复现方式就是在接收端打一个断点，挂起recv线程，当再次resume时会发现收到的数据已经是多包粘连在一起。因此最好的解决方法还是指定协议是约定好数据包的分隔符，在接收端重新组包。



## 参考资料

1.UNIX网络编程

2.https://linux.die.net/man/3/fd_set

3.https://www.gnu.org/software/libc/manual/html_node/Socket_002dLevel-Options.html#Socket_002dLevel-Options

4.https://stackoverflow.com/questions/3757289/tcp-option-so-linger-zero-when-its-required

5.https://www.gnu.org/software/libc/manual/html_node/Waiting-for-I_002fO.html

6.https://zhuanlan.zhihu.com/p/65810324

7.https://en.wikipedia.org/wiki/Transmission_Control_Protocol#Connection_termination

8.https://stackoverflow.com/questions/3757289/tcp-option-so-linger-zero-when-its-required

9.https://en.wikipedia.org/wiki/Maximum_segment_lifetime

10.http://cmd.inp.nsk.su/old/cmd2/manuals/gnudocs/gnudocs/libtool/libtool_36.html

11.https://superuser.com/questions/267660/can-someone-please-explain-ifconfig-output-in-mac-os-x