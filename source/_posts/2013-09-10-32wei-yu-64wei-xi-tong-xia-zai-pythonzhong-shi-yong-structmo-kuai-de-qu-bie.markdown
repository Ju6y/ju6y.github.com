---
layout: post
title: "32位与64位系统下在python中使用struct模块的区别"
date: 2013-09-10 13:31
comments: true
categories: python
---

今天有人问我，在32位系统下使用struct pack了一个数据在64位系统下unpack时报错，应该怎么解决。并且该32位系统部署了其他服务没法重装系统，同时运行在64位系统下的unpack代码也访问不到，唯一能做就是修改pack部分代码。代码如下：

```
strData = struct.pack("36sQQ", strSession, llChips, 0)
```

第一眼看了代码，觉得是pack "Q"有问题，也就是打包**unsigned long long**类型的数据，即8个字节的无符号整型，映像中好像是32位系统不支持这么大的整型，于是决定下个python源码看看struct是怎么pack数据的。

<!--more-->

看了会儿源码找到了打包**unsigned long long**类型的函数，发现了一个**ifdef**，这个**ifdef**来判断系统是否支持**unsigned long long**类型数据的pack，并且还附有一段注释：

```
/* We can't support q and Q in native mode unless the compiler does;
   in std mode, they're 8 bytes on all platforms. */
ifdef HAVE_LONG_LONG
typedef struct { char c; PY_LONG_LONG x; } s_long_long;
#define LONG_LONG_ALIGN (sizeof(s_long_long) - sizeof(PY_LONG_LONG))
#endif 
```

注释中说，标准模式下，所有平台的**unsigned long long**类型都是8字节，完全没提到32位或者64位，我心里一想不会是记错了吧，赶紧谷歌了一下，发现struct的官方文档对"Q"这个类型的平台支持进行了说明，如下：

> The 'q' and 'Q' conversion codes are available in native mode only if the platform C compiler supports C long long, or, on Windows, __int64. They are always available in standard modes.


同样没提及64位和32位，我一想既然这样索性自己试一下就知道是不是支持了。于是找来了32位机器，在python命令行下试验了一下pack "Q"类型数据，并对比了64位下的结果，完全没有区别，事实证明我之前对**unsigned long long**类型的理解是错的。既然打包“Q”类型没有问题，那这个unpack不了的问题又出在哪里呢，我猜想是不是字节对齐的问题，因为pack的时候会对数据进行字节补齐，于是继续看struct文档，下面是pack时，可以指定的字节序和对齐方式的标识：

Character | Byte order             | Size     | Alignment 
--------- | ---------------------- | -------- | ----------
@         | native                 | native   | native    
=         | native                 | standard | none      
<         | little-endian          | standard | none      
\>        | big-endian             | standard | none      
!         | network (= big-endian) | standard | none      

我们可以看到跟对齐关系有关的也就是2种，对齐和不对齐（这不废话么）。如果在pack函数的第一个参数fmt字符串最前面加了@就会使用字节对齐，另外需要注意的是，如果没有上面的任何一个符号，默认就是使用@，原文如下:

> If the first character is not one of these, '@' is assumed.

看到这里可能有人会怀疑是字节序的问题，其实目前大部分平台都是使用小端序，即little-endian，只有少数平台使用big-endian，文档中也有说明：

> Native byte order is big-endian or little-endian, depending on the host system. For example, Intel x86 and AMD64 (x86-64) are little-endian; Motorola 68000 and PowerPC G5 are big-endian; ARM and Intel Itanium feature switchable endianness (bi-endian)

可以看到如果使用的是Intel和AMD的CPU，那就是使用的小端序。

我们继续看字节对齐的问题，文档提到：

> Native size and alignment are determined using the C compiler’s `sizeof` expression.

可以看到如何对齐取决于c编译器，一般也就是平台有关。经过一番查找，发现32位下是4字节对齐，64位下是8字节对齐，然后看到文章开头的pack代码里有个“36s”，36明显不能被8除尽，于是我对比了32位与64位下的pack结果：

32位:

```
>>> struct.pack('36sQQ', '1'*36, 1, 0)
'111111111111111111111111111111111111\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```
64位：

```
>>> struct.pack('36sQQ', '1'*36, 1, 0)
'111111111111111111111111111111111111\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```

可以发现，64位的'\x01'前多了4个空字节，这其实就是字节补齐的结果，到这里其实就知道怎么解决这个问题了，就是给32位的pack结果增加4个空白字节就行，可以用字符串拼接，如下：

```
>>> struct.pack('36s', '1'*36) + '\x00\x00\x00\x00' + struct.pack('QQ', 1, 0)
'111111111111111111111111111111111111\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```

也可以直接指定为40个字符的字符串，如下:

```
>>> struct.pack('40sQQ', '1'*36, 1, 0)
'111111111111111111111111111111111111\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```








