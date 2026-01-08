<div align="center">

#  Webcoding

> 这篇文章从简单复杂展现了基于c的网络编程思想，代码作为理解网络的必要，更重要的是体现网络中的设计哲学,我更建议的是尝试手敲代码，根据这份readme所提供的信息来尝试，为了更清晰的展现网络结构，所有的代码就只是对应功能的简单实现，极为粗糙。相信你在这段过程中，可以去体悟到网络设计的底层实现。现在我主要进行的是c的tcp和udp的简单实现，后续会加入对数据报文，ip协议栈，和对go的底层设计的接口分析。尽管并不是自顶向下的构成分析，但是从对一个网络初学者而言，这是不错的起点

![Language](https://img.shields.io/badge/language-C-blue.svg)
![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20WSL-green.svg)
![Editor](https://img.shields.io/badge/Editor-VS%20Code-orange.svg)

</div>

---

## 目录结构 (Contents)

本项目按照学习路线分为以下几个模块：

- [01 Basic (基础概念)](#01-basic-基础概念)
- [02 UDP Socket (UDP 通信)](#02-udp-socket-udp-通信)
- [03 TFTP Implementation (TFTP 协议实现)](#03-tftp-implementation-tftp-协议实现)
- [04 Broadcast & Multicast (广播与多播)](#04-broadcast--multicast-广播与多播)
- [05 TCP Socket (TCP 通信)](#05-tcp-socket-tcp-通信)

---

## 详细介绍 (Introduction)

### 01 Basic (基础概念)
网络编程的基石，主要解决不同机器间的数据表示差异。

* **01_endian (字节序)**
    * 展示了计算机 **小端存储 (Little-Endian)** 与 **大端存储 (Big-Endian)** 的区别。
    * **为什么会有这种存储的差异性**：这纯粹是CPU架构的历史遗留问题（比如 Intel x86 选了小端，而早期的 Motorola 选了大端）。但为了防止乱套，网络协议强行规定了必须用**大端**作为网络字节序。所以我们在发包前必须老老实实把主机的小端序转过去。
  
* **02_htol_htons (字节序转换)**
    * 基于 `<arpa/inet.h>` 头文件。
    ```c
    extern uint16_t htons (uint16_t __hostshort)
     __THROW __attribute__ ((__const__));
    ```
    * 实现 **主机字节序 (Host)** 向 **网络字节序 (Network)** 的转换 (如 `htonl`, `htons`)。
    * **解释一下 `__THROW` 和 `__attribute__`**：这其实是写给编译器看的“小抄”。`__THROW` 告诉编译器这函数绝不抛出异常，`__const__` 告诉编译器这函数是“纯函数”（只依赖输入，没副作用）。这样编译器就能大胆地做优化，把多余的调用给省掉。

* **03_inet_pton (IP地址转换)**
    * 全称 *Presentation to Numeric*。
    * 将点分十进制字符串 (如 "192.168.1.1") 转换为网络传输用的 32位无符号整数。
    ```c
    int inet_pton (int __af, const char *__restrict __cp,
		      void *__restrict __buf) __THROW;
    ```
    * **为什么需要一个 void 类型**：这里设计得很贼，因为 IPv4 用 `struct in_addr` (4字节)，IPv6 用 `struct in6_addr` (16字节)。用 `void*` 就能像万能插头一样，不管你是哪种协议，都能把转换后的二进制数据填进去。

* **04_inet_ntop (IP地址还原)**
    * 全称 *Numeric to Presentation*。
    * 将 32位网络字节序整数还原为人类可读的 IP 字符串。
    ```c
    extern const char *inet_ntop (int __af, const void *__restrict __cp,
			      char *__restrict __buf, socklen_t __len)
     __THROW;
    ```
    * `extern` 意味着这是个外部引用，`__len` 则是为了防止缓冲区溢出（C语言老生常谈的内存安全问题），这一部分被认为是add部分单独开一行。

### 02 UDP Socket (UDP 通信)
无连接的、不可靠的数据传输协议。
* **01_socket(套接字)**
    * 展示了socket套接字创建的函数
    ```c
    int socket (int __domain, int __type, int __protocol)
    ```
    * domain决定IP类型，type则是决定tcp还是udp的传输类型。protocol是具体的协议格式
    * socket是一个int类型，靠文件描述符的抽象在系统层面上实现调用，展现了linux中一切皆文件的设计思想
    * 作为文件描述符，一定要在程序最后对其进行关闭，close在通信过程中就意味着断开连接，在tcp中，这一点变得更为复杂
* **02_sendto**
   * 展示了udp传输类型的数据发送
   ```c
   ssize_t sendto (int __fd, const void *__buf, size_t __n,int __flags, __CONST_SOCKADDR_ARG __addr,socklen_t __addr_len);
   ```
* **ssize_t 是个啥**：其实就是 `signed int`。因为这函数成功时返回发送字节数（正数），失败要返回 -1。如果用普通的 `size_t`（无符号），就没法表示 -1 这个错误状态了。
* **adding**
* 在这里进行数据传输的时候通过addrsocket_in记录和写入ip和port


```c
struct sockaddr_in
{
 __SOCKADDR_COMMON (sin_);
 in_port_t sin_port;			/* Port number.  */
 struct in_addr sin_addr;		/* Internet address.  */

 /* Pad to size of `struct sockaddr'.  */
 // ... padding ...
};

```
* 需要注意的是，尽管我们写入的是sockaddr_in但是封装函数写入的确实sockaddr结构体


```c
struct sockaddr
{
 __SOCKADDR_COMMON (sa_);	/* Common data: address family and length.  */
 char sa_data[14];		/* Address data.  */
};

```


* 可以看到，这实际上是进行了一个数据压缩的过程，这样的设计展现了编程最核心的问题，自然语言和二进制之间的矛盾指出


* **03_bind**
* bind这个函数主要是为了固定ip和端口号，根据这个需求我们可以很容易理解信息的接收方更加需要这个需求。



```c
int bind (int __fd, __CONST_SOCKADDR_ARG __addr, socklen_t __len)
     __THROW;

```

* 在tcp/udp编程时我们一般简单的就把server称为需要绑定的一方，但这实际上是由于信息接受和发送的相对关系所决定的，在多播和组播中我们能够看到这一点的进一步体现
* **04_recvfrom**
* recvform是接受函数
```c
recvfrom (int __fd, void *__restrict __buf, size_t __n, int __flags,
    __SOCKADDR_ARG __addr, socklen_t *__restrict __addr_len)
```


* 这里需要注意的是recvfrom是接受别的主机的数据，所以我们需要预先创建空的结构体供其填入，并且其还更高addrlen来作为接受到的信息长度输出，这种设计使得udp可以很简单的实现多线程工作，代价就是每一个client都需要相应的结构体来对应，而我们将会在后续看到，因为tcp要求的三次握手四次挥手，导致tcp的设计走向了截然不同的道路


* **05_server&&06_clieng** * 这一段是具体的udp的客户端和服务端的运行代码。本质上就是调用上述函数具体实现通信过程
```c
  if(argc<3){
fprintf(stderr,"Usage : %s<IP> <PORT>\n",argv[0]);
exit(1);
} 
```


* 这一段保证了运行程序时输入了正确的ip和port，这里需要注意的是，client里面输入的也是server的ip和port，因为client根本不需要在乎自己，他只需要保证数据交互


```c
 if(recvfrom(sockfd,buf,sizeof(buf),0,(struct sockaddr*)&clientaddr,&addrlen)==-1){
    perror("fail to recvfrom");
    //即时接受失败也可以继续进行
    continue;
}
```


* 这里体现了recvfrom的核心用处，是udp传输中的核心所在，通过接受数据，将数据发送主机的ip记录下来，用来进行sendto操作，这种设计使得udp的多client能极为容易理解设计，尽管在实际执行写起来的时候稍微有点冗余。
* **udp通信交互流程简图**：


```text
[Client]                          [Server]
   |                                 |
   |--- sendto(Data, ServerIP) ----->|
   |                                 | recvfrom (获取 ClientIP)
   |                                 |
   |<-- sendto(Echo, ClientIP) ------|
   |                                 |
recvfrom(Echo)

```


* **adding**
* 我们这里的输入输出主要使用fgets和printf方法


```c
extern char *fgets (char *__restrict __s, int __n, FILE *__restrict __stream)
   __wur __fortified_attr_access (__write_only__, 1, 2) __nonnull ((3));

```


* **缓冲区的大坑**：在c中string以'/0'作为在内存中的数据界限，而fgets则会将换行符计入，如果为了数据的纯洁性应该将换行符去掉，但是对于printf中来说只有遇到换行符才会将数据从缓冲区中打印出。所以这两个函数搭配时不需要做什么处理。



### 03 TFTP Implementation (TFTP 协议实现)

*Trivial File Transfer Protocol* (简单文件传输协议)。

* **begin**
* 在开始之前我想简单的介绍一下理解tftp的核心所在，作为基于udp协议的小文件传输协议，在c中实现最让人恼火的就是**手动构建和分析二进制报文**，你需要像做手术一样去拼凑每一个字节。


* **01_tftp_client**
* **Part 1: 报文构造区**
* 明确下载文件名 `scanf("%s",filename);`
* 第一个难点是去构造数据报文，这玩意不是字符串，是紧凑的二进制。
* **TFTP 二进制报文结构图**：
```text
 2 bytes     string    1 byte     string   1 byte
------------------------------------------------
| Opcode |  Filename  |   0  |    Mode    |   0  |
------------------------------------------------

```


* 代码里用了一个很骚的操作 `sprintf` 来拼接：
```c
packet_buf_len = sprintf((char*)packet_buf,"%c%c%s%c%s%c",0,1,filename,0,"octet",0);
```


* **解释一下字节序**：这里为什么不需要在意大端存储和小端存储的转换？因为 `sprintf` 是按顺序写入单字节的。写入 0 再写入 1，内存里就是 `00 01`，这恰好符合网络字节序的大端要求。


* **Part 2: 接收与解析循环 (State Machine)**
```c
//这里是数据报文的传输层级
unsigned char packet_buf[1024]= "";
```


* packet_buf 用来接受server发送来的数据，这里发送数据全部使用unsigned char 类型来进行发送，而通过数据来储存这个数据，意味者可以简单直接的接受数据包头的二进制数据来进行分析
```c
//错误信息
if(packet_buf[1]== 5){...exit(1)}
//收到server正确的反馈请求
if(packet_buf[1]==3){//进行下一步处理}
```


* 需要注意的是首先要去判断是否存在相应文件，可以用bool或int类型来标识，如果没有则需要先创建相应文件


* **Part 3: 验证与ACK (手动可靠性)**
* 另一个难点就是接受核对数据保重的区块编号，因为udp是不可靠的连接，所以说需要手动去验证是否存在数据丢失。我们需要从数据报头中读取并和本地记录的进行比对。
* **数据验证流程**：


```c
if((num +1) == ntohs(*(packet_buf+2)))
//success -> 发送ack报文
//fail    -> 数据丢失，推出
```


* 如果当前数据包没问题，需要构建ack报文发送给server，来让他来发送下一块数据。这里必须用 `ntohs`，因为包头里的序号是网络序，得转成本地序才能对比。


```c
packet_buf[1]= 4;
```


* 这里尽管发送了文件块数据，但是server只需要对数据包头进行验证
* 如果数据块小于516，即文件数据小于512，那么说明写入结束，但在这里没有去考虑文件大小刚好是512倍数的情况


* **总结**：这样一个客户端主要的难题就是二进制报文的处理，因为udp本身是不可靠的，所以我们就需要手工做数据包做数据包头进行验证，我们可以看到的是，这样一个验证思路实际上和tcp的三次握手非常相像，所以一般由tcp承担文件传输的工作


* **02_tftp_server**
* 服务端逻辑相对被动，主要是解析和反馈：
* 1. 验证数据报文和是否由相关文件


* 2. 自定义区块num并写入包头，用缓冲区作为文件数据中转


* 3. 等待并解析ack数据包


* **构造错误包 (辅助函数)**
这里需要注意的是一个构建错误数据包的函数，为了代码整洁抽出来的：
```c
void senderr(int sockfd,struct sockaddr* clientaddr,char* err,int errcode,socklen_t addrlen){
   unsigned char buf[516] = "";
   // 构造错误包: [00] [05] [00] [ErrCode] [ErrMsg] [00]
   int buf_len = sprintf((char*)buf, "%c%c%c%c%s%c", 0, 5, 0, errcode, err, 0);
   sendto(sockfd,buf,buf_len,0,clientaddr,addrlen);
}
```


* 本身这样的函数使得我们可以迅速的在逻辑末节点（比如文件打不开）发送相关信息。
* **总结**：这里服务端展现的设计思想基本与客户端一致，这里需要注意的是两边都在本地储存了区块号,双方都对数据报文进行了解析



### 04 Broadcast & Multicast (广播与多播)

一对多的通信模式。

* **Broadcast (广播)**: 实现局域网内的全员消息通知。
* **Multicast (多播/组播)**: 实现向特定组内的成员发送数据，比广播更节省带宽。

### 05 TCP Socket (TCP 通信)

面向连接的、可靠的流式传输协议。

* 实现标准的三次握手建立连接及数据传输。
* 涵盖 `listen`, `accept`, `connect` 等核心 API。

---

## 🚀 如何运行 (How to Run)

在 WSL 或 Linux 终端中，使用 gcc 编译对应文件。例如：

```bash
# 编译
gcc 01_basic/01_endian.c -o endian

# 运行
./endian
```
