---
layout: post
title: "socket详解"
date: 2014-08-30 20:35
comments: true
categories: 
- linux
- c/c++
- 网络编程
---

### NAME
socket - 创建一个通信端点
### SYNOPSIS
	#include <sys/types.h>
	
	#include <sys/socket.h>
	int socket(int domain, int type, int protocal);   
 
 
### DESCRIPTION
socket函数创建一个通信端点，并返回这个端点的描述符。

<!--more-->

#### *domain*
domain参数指定了通信域，即选择通信所用的协议族。在\<sys/socket.h\>中定义了这些协议族，已知的格式如下：

Name               | Purpose                         | Man page
\------------------ | ------------------------------- | ------------
PF\_UNIX, PF\_LOCAL  | Local communication             | unix(7)
PF\_INET            | IPv4 Internet protocols         | ip(7)
PF\_INET6           | IPv6 Internet protocols         |
PF\_IPX             | IPX - Novell protocols          |
PF\_NETLINK         | Kernel user interface device    | netlink(7)
PF\_X25             | ITU-T X.25 / ISO-8208 protocol  | x25(7)
PF\_AX25            | Amateur radio AX.25 protocol    |
PF\_ATMPVC          | Access to raw ATM PVCs          |
PF\_APPLETALK       | Appletalk                       | ddp(7)
PF\_PACKET          | Low level packet interface      | packet(7)

###### 注：关于AF和PF
	网上的一些教程例子，在创建一个使用ipv4协议的的socket时，有些将domain这个参数设为AF\_INET，有些设为PF\_INET，从实际效果来看貌似2者没有什么区别，都能正常运作，但实际2者还是有一定的区别：
	AF: Address  Family（地址族）
	PF: Protocol Family（协议族）
	目前，AF和PF在宏定义值上来说2者完全相等（linux详见<bits/socket.h>）。由于历史原因，在早期曾设想一个协议族可以有多个地址族，但现在的实际情况是，一种协议族只对应一种地址族，例如ipv4协议支持ipv4地址，而不支持ipv6地址。
	domain这个参数是用来指定socket的通信域，即通信所用的协议，那么传入PF从语义上来说比传入AF更准确。而AF应用于设定socket的地址族，即用于填充socketaddr结构体时。
	通俗点来说，在创建socket时，并没有绑定到某个具体的地址，这个时候还不需要关心地址相关的内容，所以不应该传入AF，等到使用connect或者bind时，使用AF就比较准确了。
#### *type*
socket有指定的类型，标识了通信语义。目前定义的类型有：

###### \* **SOCK\_STREAM**
提供有序的，可靠的，双向的，面向连接的字节流传输。可能支持带外的数据传输。[^1]
###### \* **SOCK\_DGRAM**
支持数据报[^2]形式数据的传输（固定最大长度的不可靠面向无连接的报文）。
###### \* **SOCK\_SEQPACKET**
提供一个支持有序的，可靠的，双向传输的且面向连接的数据传输通道，传输一组固定长度的数据报。客户端需要在每次读系统调用时读完整个包

###### \* **SOCK\_RAW**
提供原始的网络协议支持。[^3]
###### \* **SOCK\_RDM**
提供可靠但无序的数据报传输。
###### \* **SOCK\_PACKET**
已过时，不应该在现在的程序里使用。

*一些socket类型并不是所有协议族都支持的，如SOCK\_SEQPACKET这个在AF\_INET中就没实现*

#### *protocol*
指定socket对应的传输协议。一般情况下domain所指定的协议族[^4]中只有一个协议能支持指定的socket类型，我们可以设置这个参数为0来使用协议族中支持指定socket类型的协议。然而也存在多个协议支持某个socket类型的情况，这个时候就一定要用这个参数来指定使用哪个协议。
### RETURN VALUE
成功则返回socket的标识，失败则返回-1。

### ERRORS
**EACCES** 不允许创建指定类型或协议的socket

**EAFNOSUPPORT** 实现不支持指定的地址族

**EINVAL** 位置协议，或者协议族不可用

**EMFILE** 处理文件表溢出

**ENFILE** 达到系统限制的打开文件数上限

**ENOBUFS** or **ENOMEM** 内存不足，直到能够分配到足够的资源socket才能被创建

**EPROTONOSUPPORT** 通信域不支持该协议类型或者协议







[^1]:	带外数据(out—of—band data)，有时也称为加速数据(expedited data)，是指连接双方中的一方发生重要事情，想要迅速地通知对方。这种通知在已经排队等待发送的任何“普通”(有时称为“带内”)数据之前发送。带外数据设计为比普通数据有更高的优先级，带外数据是映射到现有的连接中的，而不是在客户机和服务器间再用一个连接。

[^2]:	自包含的，带有足够寻址信息的，可独立地从数据源行走到目标计算机，而不需要依赖早期的源和目标计算机之间交换信息以及传输网络的数据包。[RFC 1594][1]

[^3]:	实际上，我们常用的网络编程都是在应用层的报文的收发操作，也就是大多数程序员接触到的流式套接字(SOCK\_STREAM)和数据包式套接字(SOCK\_DGRAM)。而这些数据包都是由系统提供的协议栈实现，用户只需要填充应用层报文即可，由系统完成底层报文头的填充并发送。然而在某些情况下需要执行更底层的操作，比如修改报文头、避开系统协议栈等。这个时候就需要使用SOCK\_RAW的方式来实现。

[^4]:	注意这里的协议与上面的协议族是两个不同的概念，前者是指网络层的协议，由于它对于到传输层会出现许多协议，比如IPv4可以用来实现TCP或UDP等等传输层协议，所以称为协议族。相应的传输层的协议就简单地称为协议。

[1]:	http://tools.ietf.org/html/rfc1594