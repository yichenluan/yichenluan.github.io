---
layout: post
title: 浅谈网络编程
subtitle: 庖丁解牛
tags: [Linux]
---

# 一、前言

网络编程是一个兼具广度和深度的话题。往广了讲，上层应用千姿百态，往深了讲就涉及到多核并发编程领域了，笔者对此也是望洋兴叹。所以，本文叫做浅谈网络编程，旨在讨论基于 Linux 下的 TCP 协议，如何进行高性能网络编程工作。具体来说，本文至底向上，从 TCP/IP 到 Sockets，再到 IO 模型与并发模型，最后讨论 RPC 场景下的一些要点。

# 二、TCP

本文不是《TCP/IP 详解》阅读笔记，所以不会详细描述 TCP/IP 的具体细节，不过我们必须理解 TCP 提供了什么样的服务，才能明白协议层与接口层的联系与边界。另一方面，如果能理解 TCP 的某些特性，应用层能够很方便的构建出健壮的网络程序。快速通读 TCP/IP 协议细节可以阅读 [TCP 的那些事儿](https://coolshell.cn/articles/11564.html)。

## 2.1 TCP 总览

下图就是著名的 TCP 状态变化图。

![](http://p890o7lc8.bkt.clouddn.com/20180722172859.png)

我们使用 netstat 观察端口上的连接时，显示的就是这 11 种状态。

## 2.2 TCP 连接建立

![](http://p890o7lc8.bkt.clouddn.com/20180722145217.png)

- TCP 建立连接为什么需要 3 次握手呢？

	本质上是为了连接双方交换 ISN（Inital Sequence Number），另外 TCP 是全双工的协议，所以每次 SYN，都需要有对应的 ACK。加上服务端的 ACK 和 SYN 是在同一个包中，也就形成了 3 次握手。
- 在建立过程中，还会交换 WIN 和 MSS。

 WIN 表示发端通告的窗口大小（可通过套接字选项由进程自行设置）；MSS 表示通告发方期望接收的 MSS（最大报文段长度），目的是尽量避免分段，一般设置为外出接口的 MTU 减去 40 个字节的 TCP、IP 首部长度。
 
## 2.3 TCP 连接交互

![](http://p890o7lc8.bkt.clouddn.com/20180822154218.png)

- TCP 重传机制（超时重传、快速重传、RTT 算法）？滑动窗口（可靠传输、有序、Nagle算法）？流量控制（通告窗口）？拥塞处理（慢启动、拥塞避免算法、快速恢复）？


## 2.4 TCP 连接终止

![](http://p890o7lc8.bkt.clouddn.com/20180722145754.png)

TIME_WAIT 的目的：

- 可靠地实现 TCP 全双工连接的终止。具体来说：客户端 ACK 丢失，试图建立新连接，发送 SYN 给还处于半关闭的服务端，将返回 RST，无法建立连接；若客户端已关闭连接资源，服务端重发 FIN，客户端无法处理该连接，返回 RST。（要是 2MSL 后，最后一个 ACK 还是失败了咋办？爱咋办咋办，不管了，协议已经给出了 2MSL 的最大努力了）
- 允许老的重复分节在网络中消逝。具体来说：就是防止老分节错误的发送到了新建立的连接上。
- MSL：Maximum segment lifetime：在 Linux 下，MSL = 30s
- 阅读：[TIME_WAIT and its design implications for protocols and scalable client server systems](http://www.serverframework.com/asynchronousevents/2011/01/time-wait-and-its-design-implications-for-protocols-and-scalable-servers.html)


TIME_WAIT 造成的影响：

- 同一个五元组代表的连接在 TIME_WAIT 期间无法重用。

如何解决：

- net.ipv4.tcp_timestamps 开启（默认即开启）
- net.ipv4.tcp_tw_reuse 开启（允许复用 TIME_WAIT 连接，通过上一步插入的 timestamp 来解决老分节消逝的问题，同时也能保证复用的连接能够正常建立）
- net.ipv4.tcp_tw_recycle（已被 Linux 移除，不做讨论）

如果开启了 tcp_tw_reuse 后，TIME_WAIT 过多有什么问题吗？

- 没什么问题，TIME_WAIT 导致的 CPU 和 内存 消耗都非常低。 
- 阅读：[Coping with the TCP TIME-WAIT state on busy Linux servers](https://vincent.bernat.im/en/blog/2014-tcp-time-wait-state-linux)

## 2.5 TCP 异常
 
无论何时一个报文段发往连接时出现错误，TCP 都会发出一个复位报文段（RST）。

- 当接收端接收到 RST 后，不必发送 ACK 包来确认。
- 当发送端发送 RST 时，直接丢弃缓冲区中的包，发送 RST。

何时发送 RST：

- 到不存在的端口的连接请求
- 异常终止一个连接
- 检测半打开连接

更多可阅读：[TCP中的RST](https://zhangbinalan.gitbooks.io/protocol/content/tcpde_rst.html)

# 三、Sockets

理解套接字函数最好的材料是 [UNP](http://www.unpbook.com/) 和 [Linux man pages](http://man7.org/linux/man-pages/index.html)。本文不会详细描述套接字函数的使用细节，而是更关注其与 TCP 协议栈的联系。

## 3.1 Socket

Man: [socket](http://man7.org/linux/man-pages/man2/socket.2.html)

```c
#include <sys/socket.h>
int socket(int family, int type, int protocol);
```
socket 函数返回值作为 sockfd，称之为套接字描述符。

sockfd 最终是要指向 socket，那么，socket 是什么呢？

这部分的总结篇幅较长，放在 [Linux in Depth - 文件系统及 Socket 源码解析
](http://jinke.me/2018-08-23-socket-and-linux-file-system/)。

## 3.2 Connect
