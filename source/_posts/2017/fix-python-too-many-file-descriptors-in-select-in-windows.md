---
title: 解决Windows下Python的too many file descriptors in select()报错
date: 2017-07-15 01:17:51
categories:
 - Tech
tags:
- Python
- C++
---

使用 Python 进行 Socket 编程时，如果向轮询列表中注册过多的 file descriptors (fd/文件描述符), 执行 `poll` 操作则会抛出 too many file descriptors in select() 的错误，然而这个问题并不是 Windows 出现了偏差，并且是可以解决的。
<!--more-->
## Windows 中的 Socket 模型 ##
在 Windows 下进行 Socket 编程时，共有六种模型可供选择，分别是

* select 选择
* WSAAsyncSelect 异步选择
* WSAEventSelect 事件选择
* Overlapped I/O 事件通知
* Overlapped I/O 完成例程
* IOCP(I/O Completion Port) 完成端口

我们经常听说到的 Linux 下的 I/O 复用模型 epoll 其性能优秀，同时支持水平触发和边缘触发，只通知就绪的 fd，而 IOCP 同样拥有这些优点，那么为什么还会遇到 too many file descriptors 这种问题？

遗憾的是， Python 官方在 Windows 平台中仅提供了 select 的实现。

相对于通知机制，select 会将部分时间浪费在轮询上，并且在用户态/内核态中多次复制也会造成性能下降。

## 问题定位 ##
### 是巨硬的锅？ ###
有人会说，select 函数是有 fd 限制的，那么去看看 MSDN 上的文档 [select function](https://msdn.microsoft.com/en-us/library/windows/desktop/ms740141)
```cpp
int select(
  _In_    int                  nfds,
  _Inout_ fd_set               *readfds,
  _Inout_ fd_set               *writefds,
  _Inout_ fd_set               *exceptfds,
  _In_    const struct timeval *timeout
);
```
select 函数有5个参数，但是对于第一个参数，MSDN 指出：
>_nfds [in] Ignored. The nfds parameter is included only for compatibility with Berkeley sockets._

在注释中同样写到：
>_The select function returns the number of sockets meeting the conditions. A set of macros is provided for manipulating an fd\_set structure. These macros are compatible with those used in the Berkeley software, but the underlying representation is completely different._

显然第一个参数并不生效，虽然它描述了 fd 的数量，但 Windows 下的实现方式并不与 Berkeley Unix 相同。

在备注的末尾，指出了关键的部分：
>_The variable FD\_SETSIZE determines the maximum number of descriptors in a set. (The default value of FD\_SETSIZE is 64, which can be modified by defining FD\_SETSIZE to another value before including Winsock2.h.)_

在宏中定义的 `FD_SETSIZE` 看起来确实会决定着文件描述符的最大数量限制，虽然其默认值是64，但是我们可以通过预先定义 `FD_SETSIZE` 的方式来改变这个数量限制。

在 MSDN 的另一篇文章中，类似于 Winsock 编程指南一样的手册里，提到了关于这个最大数量限制的事情 [Maximum Number of Sockets Supported](https://msdn.microsoft.com/en-us/library/windows/desktop/ms739169)：
>* _The maximum number of sockets supported by a particular Windows Sockets service provider is implementation specific. The Microsoft Winsock provider limits the maximum number of sockets supported only by available memory on the local computer._
>* _However, third-party Winsock providers may have limitations on the numbers of sockets supported. An application should make no assumptions about the availability of a certain number of sockets._
>* _The maximum number of sockets that a Windows Sockets application can use is not affected by the manifest constant FD\_SETSIZE._

翻译一下：

* 由 Windows Sockets 支持服务提供的 sockets 最大连接数限制是基于特定实现的。Microsoft Winsock provider 仅用可用的最大内存来限制 sockets 最大连接数。
* 第三方 Winsock provider 可能会在提供的 sockets 支持上做出自己的限制。
* Windows Sockets 程序能够使用的最大 sockets 连接数不受 manifest 中的常量 FD\_SETSIZE 限制。

既然没有限制，那么来看看 `WinSock2.h` 中的定义
```cpp
typedef struct fd_set {
        u_int fd_count;               /* how many are SET? */
        SOCKET  fd_array[FD_SETSIZE];   /* an array of SOCKETs */
} fd_set;
```
显而易见， `fd_count` 才是指明了当前已设置的 fd 数量的关键，也就是说，只要我们自己定义 `FD_SETSIZE` 宏即可解除这个限制。
### 是 Python 的锅？ ###
找到 Python 的源码 [cpython](https://github.com/python/cpython)，在 `cpython/Modules/selectmodule.c` 中实现了所用到的 select.pyd 动态库。

在文件一开头的注释中，就能看到：
>_select - Module containing unix select(2) call.
>Under Win32, select only exists for sockets._

也就是说， Python for Windows 确实只有 select 这一个实现。。。。

直接找到报错行，可以看到：
```cpp
#if defined(MS_WINDOWS) && !defined(FD_SETSIZE)
#define FD_SETSIZE 512
#endif

static int
seq2set(PyObject *seq, fd_set *set, pylist fd2obj[FD_SETSIZE + 1])
{
    /* some code here */
    if (index >= (unsigned int)FD_SETSIZE) {
        PyErr_SetString(PyExc_ValueError, "too many file descriptors in select()");
        goto finally;
    }
    /* some code here */
}
```
Python 对于 `fd_count` 并没有过多的处理，仅仅是简单判断后即返回，并且 `FD_SETSIZE` 的默认值是 512。

那么 Python 的二进制分发是不是一样的呢？
反汇编 select.pyd，直接找到关键块
```x86asm
.text:1D1110F0                 cmp     [ebp+var_4], 200h
.text:1D1110F7                 jge     short loc_1D111120
.text:1D1110F9                 inc     [ebp+var_4]
.text:1D1110FC                 add     [ebp+var_8], 4
.text:1D111100                 mov     [esi-8], edi
.text:1D111103                 mov     edi, [ebp+var_4]
.text:1D111106                 mov     [esi-4], eax
.text:1D111109                 mov     eax, [ebp+var_C]
.text:1D11110C                 mov     dword ptr [esi], 0
.text:1D111112                 add     esi, 0Ch
.text:1D111115                 mov     dword ptr [esi], 0FFFFFFFFh
.text:1D11111B                 jmp     loc_1D111080
.text:1D111120
.text:1D111120 loc_1D111120:   ; CODE XREF: sub_1D111040+B7ij
.text:1D111120                 mov     ecx, ds:PyExc_ValueError
.text:1D111126                 mov     edx, [ecx]
.text:1D111128                 push    offset aTooManyFileDes ; "too many file descriptors in select()"
.text:1D11112D                 push    edx
.text:1D11112E                 call    ds:PyErr_SetString
.text:1D111134                 add     esp, 8
```
看到前两行
```x86asm
cmp     [ebp+var_4], 200h
jge     short loc_1D111120
```
200h 即十进制数 512， cmp 后立刻是 jge，也就是说当 ebp+var_4 所指向的变量大于等于 512 时，跳转到 1D111120 处，与 selectmodule.c 中的源码相同。

那么解决方案很显然了，只要修改 selectmodule.c 中的宏定义即可。
## 环境搭建 ##
在 `cpython/PCbuild/readme.txt` 中，给出了编译的步骤，只需要一步步来即可
*<center>以下环境仅适用于 Python2<br>Python3 可直接使用 Visual Studio 2017 编译</center>*

1. 卸载当前已安装的任何 Microsoft Visual C++ 2010 Redistributable Package
2. 安装 Windows SDK 7.1
3. 安装 Microsoft Visual C++ 2008 任意版本
4. 安装 Microsoft Visual C++ 2010 任意版本
5. 安装 TortoiseSVN （注意要勾选 command line client tools）
6. 安装 Git for Windows

一切准备妥当之后，先下载依赖，配置环境
```html
call cpython\PCbuild\get_externals.bat
call cpython\PCbuild\build_env.bat
```

打开 Microsoft Visual C++ 2010，新建一个 Solution，把 pythoncore.vcxproj 和 select.vcxproj 添加进解决方案，把两个项目的 `Platform Toolset` 都改成 Windows7.1SDK，即可编译完成。

<center>========== Build All: 2 succeeded, 0 failed, 0 skipped ==========</center>

Cheers~