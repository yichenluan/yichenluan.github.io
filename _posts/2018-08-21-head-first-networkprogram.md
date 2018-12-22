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

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102345.png)

我们使用 netstat 观察端口上的连接时，显示的就是这 11 种状态。

## 2.2 TCP 连接建立

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102418.png)

- TCP 建立连接为什么需要 3 次握手呢？

	本质上是为了连接双方交换 ISN（Inital Sequence Number），另外 TCP 是全双工的协议，所以每次 SYN，都需要有对应的 ACK。加上服务端的 ACK 和 SYN 是在同一个包中，也就形成了 3 次握手。
- 在建立过程中，还会交换 WIN 和 MSS。

 WIN 表示发端通告的窗口大小（可通过套接字选项由进程自行设置）；MSS 表示通告发方期望接收的 MSS（最大报文段长度），目的是尽量避免分段，一般设置为外出接口的 MTU 减去 40 个字节的 TCP、IP 首部长度。
 
## 2.3 TCP 连接交互

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102448.png)

- TCP 重传机制（超时重传、快速重传、RTT 算法）？滑动窗口（可靠传输、有序、Nagle算法）？流量控制（通告窗口）？拥塞处理（慢启动、拥塞避免算法、快速恢复）？


## 2.4 TCP 连接终止

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102511.png)

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

man: [socket](http://man7.org/linux/man-pages/man2/socket.2.html)

```c
#include <sys/socket.h>
int socket(int family, int type, int protocol);
```
socket 函数返回值作为 sockfd，称之为套接字描述符。

sockfd 最终是要指向 socket，那么，socket 是什么呢？

这部分的总结篇幅较长，放在 [Linux in Depth - 文件系统及 Socket 源码解析
](http://jinke.me/2018-08-23-socket-and-linux-file-system/)。

## 3.2 Bind

man: [bind](http://man7.org/linux/man-pages/man2/bind.2.html)

```c
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr* myaddr, socklen_t addrlen);
```

调用 Bind，可以指定 IP 地址或者端口，可以都指定，也可以都不指定。

## 3.3 Connect

man: [connect](http://man7.org/linux/man-pages/man2/connect.2.html)

```c
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr* servaddr, socklen_t addrlen);
```

Connect 函数开启 TCP 连接的三次握手过程，对应于 TCP 状态图中的：`CLOSED -> SYN_SEND -> ESTABLISHED`

- 发出 SYN 后，没有收到对应的 ACK，会进行重试；
- 对于阻塞模式的 Connect 而言，当三次握手完毕，状态变为 `ESTABLISHED`，connect 返回。


## 3.4 Listen

man: [listen](http://man7.org/linux/man-pages/man2/listen.2.html)

```c
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```

Listen 将对应的 socket 指定为监听 socket。对应于 TCP 状态图中的：`CLOSED -> SYN_RECEIVED`

参数中除了指定的 `sockfd` 外，还有参数 `backlog`。`backlog` 是与监听后的连接建立有关的。

当服务端调用 listen 进行端口监听后，客户端调用 connect 试图建立 TCP 连接，服务端的过程对应于 TCP 状态图中的 `LISTEN -> SYN_RECEIVED -> ESTABLISHED`。

其中 `SYN_RECEIVED` 状态对应于服务端接收到客户端发来的 `SYN`，但还没有接收到返回的 `ACK`，`ESTABLISHED` 代表服务端接收到 `ACK`，已经完成了三次握手过程。

那么，对应的，协议栈有两个队列暂存处于这两种状态的连接，分别是：

 - SYN 队列（收到 `SYN`，但还未收到客户端返回的 `ACK`）
 - ACCEPT 队列（完成了三次握手的连接，但还未被用户取出）

在 Linux 下，backlog 用来指定 ACCEPT 队列的大小，具体如下：

- Size(Accept_Queue) = min(backlog, `/proc/sys/net/core/somaxconn`)
- Size(SYN_Queue) = `/proc/sys/net/ipv4/tcp_max_syn_backlog`

这时，出现的问题是：如果客户端发送 `SYN` 后就掉线，无法对服务端发来的 `SYN-ACK` 做出对应的 `ACK` 回应，会发生什么事？

可想而知，服务端无法判断 `SYN-ACK` 是丢失在网络中还是客户端无法回应，所以会进行重试。在 Linux 下，服务端使用指数退避进行超时重试（1s, 2s, 4s, 8s, 16s, 32s)，也就是说，服务端总计需要 63s 才会放弃该连接。

于是，问题出现了，黑客们故意发送大量的 `SYN` 给服务端，同时不返回三次握手最后的 `ACK`，这样服务端每个这样的连接都需要在 SYN 队列中保留 63s，于是当 SYN 队列被耗尽后，正常的连接就无法被处理。这就是所谓的 `SYN Flood` 攻击。

Linux 采用了 `/proc/sys/net/ipv4/tcp_syncookies` 来应对 SYN 泛洪，当 `tcp_syncookies` 开启后，SYN 队列没有逻辑上的长度限制了，之前的 `tcp_max_syn_backlog` 参数也会被忽略，具体来说，服务端接收到建立连接的 `SYN` 后，会将连接信息编码为 cookie 附带在 `SYN+ACK` 中，同时，客户端返回的 `ACK` 将会携带 cookie，由此，服务端可以通过 cookie 来恢复连接，这样服务端就无需在队列中保留 SYN 连接信息。

下面是一些补充细节：

- 如果 `tcp_syncookies` 关闭，当 SYN 队列满后，服务端会丢弃新来的 `SYN`，客户端多次重试后，产生 Timeout 错误。
- 如果 ACCEPT 队列满，由 `/proc/sys/net/ipv4/tcp_abort_on_overflow` 来决定，为 `0` 将忽略 `ACK`；为 `1` 将返回 `RST`，迫使客户端断开连接。
- 注意，如果 `tcp_abort_on_overflow` 为 `0`，忽略 `ACK` 后，服务端会认为没收到 `ACK`，于是会重发 `SYN+ACK`，这样客户端又会重发 `ACK`，如果 ACCEPT 队列一直无法腾出空间，最终会访问 `RST`。
- 另外，当 ACCEPT 队列满后，协议栈还有额外处理，内核将会强制限制 `SYN` 包的接收速率。

关于 TCP 建立连接与 `listen` 的理论已经说的差不多啦，来看看 Real World。

```
// 以下参数皆从一台线下机器获取
// ACCEPT 队列最大值为 4096
$ cat /proc/sys/net/core/somaxconn
4096

// SYN 队列最大值为 8192
$ cat /proc/sys/net/ipv4/tcp_max_syn_backlog
8192

// 开启了 syncookies
$ cat /proc/sys/net/ipv4/tcp_syncookies
1

// overflow 设为了 0
$ cat /proc/sys/net/ipv4/tcp_abort_on_overflow
0
```

```c
// UB
sev->backlog = 2048;

// Muduo
int ret = ::listen(sockfd, SOMAXCONN);

// brpc
if (listen(sockfd, INT_MAX) != 0) {
    //             ^^^ kernel would silently truncate backlog to the value
    //             defined in /proc/sys/net/core/somaxconn
    return -1;
}

// Golang
func maxListenerBacklog() int {
	fd, err := open("/proc/sys/net/core/somaxconn")
	// ...
}
// ...
var listenerBacklog = maxListenerBacklog()
// ...
fd.listenStream(laddr, listenerBacklog);
```
光从这几段代码来看，`UB` 还是差了点火候。

## 3.5 Accept

man: [accept](http://man7.org/linux/man-pages/man2/accept.2.html)

```c
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr* cliaddr, socklen_t* addrlen);
```

`accept` 就是从 ACCEPT 队列中取出一个已完成三次握手的连接，对于阻塞 `accept` 而言，当 ACCEPT 队列为空时，就会阻塞住。

`accept` 的出参和入参的 `sockfd` 是两个完全不相干的套接字。

## 3.6 Close

man: [close](http://man7.org/linux/man-pages/man2/close.2.html)

默认行为是将套接字标记为已关闭，然后立即返回。TCP 将尝试发送已排队数据，然后再进行 TCP 断开连接的过程。

如果想直接发送 FIN，可以调用 `shutdown`。

# 四、IO 模型

## 4.1 阻塞 IO

### 阻塞 Accept

上文已经提到了，如果调用 `accept` 时，ACCEPT 队列中无可用连接返回，`accept` 将会阻塞直到新连接到达。

### 阻塞 Connect

阻塞 `connect` 意味着要等到 TCP 建立连接三次握手中的第二步（客户端发出的 `SYN` 对应的 `ACK`）到达后才返回。

### 阻塞 Read / Write

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102553.png)

一图胜千言。

- 当 socket 的接收缓冲区低于低水位时，read 阻塞。
- 当 socket 的发送缓冲区没有足够空间时，write 阻塞。

## 4.2 非阻塞 IO

### 非阻塞 Accept

若 ACCEPT 队列中没有可用连接，立即返回 `EAGAIN` or `EWOULDBLOCK`。

- 使用 IO Multiplexing 时，ACCEPT 队列可用，返回可读事件。

### 非阻塞 Connect

若连接能立即建立（本机连本机），正常返回。

否则，返回 `EINPROGRESS`。

- 使用 IO Multiplexing 时，连接建立完毕，返回可写事件。

### 非阻塞 Read / Write

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102614.png)

## 4.3 异步 IO

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102633.png)

# 五、并发模型

到现在为止，我们算是把 `协议栈` 和 `操作系统` 提供的抽象分析完毕了，接下来，就是看我们如何利用这些能力构建网络并发模型。（其实这部分内容之前已经总结过了，[在这](http://jinke.me/2018-05-28-client-server-design/)，不过这次新加了一些深入思考和理解。）

有些同学看到这可能会疑惑，我想看的 `IO Multiplexing` 哪去了？别急，先往下看。

## 5.1 方案 0（Accept + Read/Write）

方案均以粗糙的 Python 玩具代码为例。

```python
import socket

def handle(client_socket, client_address):
    // L6
    while True:
        data = client_socket.recv(4096)
        if data:
            sent = client_socket.send(data)    # sendall?
        else:
            print "disconnect", client_address
            client_socket.close()
            // L13
            break

if __name__ == "__main__":
    listen_address = ("0.0.0.0", 2007)
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.bind(listen_address)
    server_socket.listen(5)

    // L21
    while True:
        (client_socket, client_address) = server_socket.accept()
        print "got connection from", client_address
        // L24
        handle(client_socket, client_address)
```

该方案对应于 `UNP P13. Figure 1-9`。

这个方案实际上不是一个并发服务器，它是迭代的。

`L6 ~ L13` 是这个服务的业务逻辑循环，`L21 ~ L24` 决定了它每次只能处理一个请求，当前请求未处理完前，无法接受下一个请求。

## 5.2 方案 1（Accept + Fork）

方案 1 是 fork()-per-client，也就是 process-pre-connection。

```python
from SocketServer import BaseRequestHandler, TCPServer
from SocketServer import ForkingTCPServer, ThreadingTCPServer

class EchoHandler(BaseRequestHandler):
    def handle(self):
        print "got connection from", self.client_address
        // L9
        while True:
            data = self.request.recv(4096)
            if data:
                sent = self.request.send(data)    # sendall?
            else:
                print "disconnect", self.client_address
                self.request.close()
                // L16
                break

if __name__ == "__main__":
    listen_address = ("0.0.0.0", 2007)
    server = ForkingTCPServer(listen_address, EchoHandler)
    server.serve_forever()
```

该方案对应于 `UNP P651. Figure 30-4`。

`L9 ~ L16` 依旧是业务逻辑代码，但是此时每个客户连接，都会 `fork` 一个子进程去处理。

在这个方案中，业务逻辑已经初步从网络框架中分离出来，但是仍然和 IO 紧密结合。

## 5.3 方案 2（Accept + Thread）
方案 2 是 thread-per-connection。伸缩性收到线程数的限制，线程太多，对操作系统的 scheduler 负担会很大。

```python
// diff echo-fork.py echo-thread.py

if __name__ == "__main__":
    listen_address = ("0.0.0.0", 2007)
    - server = ForkingTCPServer(listen_address, EchoHandler)
    + server = ThreadingTCPServer(listen_address, EchoHandler)
    server.serve_forever()
```

该方案对应于 `UNP P668. Figure 30-26`

相比于方案 1，就是把进程改为了线程。进程和线程有什么区别呢？对于 Linux Kernel 而言，没有区别，所以 Too Many Process 和 Too Many Thread 是一样的，线程创建、销毁和调度带来的代价。除此之外，线程之间共享数据，额外产生的 cache locality 的问题。

## 5.4 方案 3/4（Pre Process/Thread）

这是对方案1和2的优化，为了避免过多进程/线程带来的负担，提前创建好进程/线程池。详细分析描述见 UNP，再次不过多赘述。

额外说下`惊群问题`：传统意义上的惊群问题是指多进程/线程同时阻塞在 Accept 上，这时一个连接到来，内核将所有阻塞在 Accept 上的进程/线程全部唤醒，但是只有一个进程/线程可以拿到新连接，其他的进程/线程等于是白唤醒了，白产生了线程切换调度的代价。

- 现在的 Linux 内核，已经不会发生传统的 Accept 惊群。内核只会唤醒一个阻塞住的进程/线程。
- 但是使用 Epoll 处理非阻塞 Accept，依旧有可能发生惊群。原因是 Epoll 通知某个进程/线程处理 Accept 事件后，会继续通知其他进程/线程，直到全部唤醒或某个进程已将该事件处理完毕。
- Nginx 如何处理惊群：`accept_mutex`，抢到锁的进程才会把 Accept 事件添加到监听事件中，同时加入其他算法减少同时竞争锁的情况。


## 5.5 中场讨论

上面所述的几种方案并发性能如何呢？

不然分析，性能是比较差的。网络编程需要思考两个问题：

- 如何处理 I/O：同步？异步？阻塞？非阻塞？
- 如何处理连接：一个进程一个连接？一个线程一个连接？一个线程多个连接？

上面的方案都可以视为一个线程处理一个连接，且以同步阻塞的方式来处理网络 IO。

那么在并发情况下，需要有大量的线程来处理单个连接，同时其中绝大部分的线程都阻塞在网络 IO（read / write) 上。

怎么办？

读 [The C10K problem](http://www.kegel.com/c10k.html).

简单来总结：IO Multiplexing && Non-Blocking IO.

为什么不聊异步 IO？

 - Linux 异步 IO 并不完善。POSIX AIO 是使用线程池调用阻塞 IO 模拟的；Kernel AIO 并不是为 socket 设计的。
 - 异步 IO 不见得性能要好（总归要做这些工作，将这些事分配给其他线程或者内核去做并没有影响工作的总量）。

通过 Non-Blocking IO 避免线程大量时间阻塞在等待数据到达或发送上，浪费 CPU；通过 IO Multiplexing 来复用线程，使得一个线程能同时处理多个连接。

Non-Blocking IO 上文已经讨论过了，什么是 IO Multiplexing ？

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102706.png)

就是所谓的 IO 多路复用，复用的是什么呢？复用的线程。通过诸如 `select`、`poll`、`epoll` 这种技术使得一个线程可以监听多个连接上的读、写事件。

IO Multiplexing 本文只讨论 Linux 下的 Epoll，原因见 epoll 这个 Linux Kernel Patch：[Improving (network) I/O performance ...](http://www.xmailserver.org/linux-patches/nio-improve.html)。

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102754.png)

### Epoll Usage

直接阅读 man: [epoll](http://man7.org/linux/man-pages/man7/epoll.7.html)

关于 LT（Level-triggered）和 ET（Edge-triggered）：

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102812.png)

### Why Must Non-Blocking IO

问题是这样的，上面我们一直在说 IO Multiplexing 和 Non-Blocking IO，但是，既然我们都用 `epoll` 了，当 `epoll` 告知我们 `EPOLLIN` 事件，也就是 socket 可读，我们直接读就好了，为什么还要用 `Non-Blocking IO` 呢？

当然，这里提到的 `Must Non-Blocking IO` 是指工程角度考虑，`epoll` 不关心文件属性。

我们分情况来讨论：

#### ET

ET 模式下无需多做考虑，必须使用 `Non-Blocking IO`。

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102832.png)

举个例子：client 和 server 通信，server 从连接中 read 到部分数据后，不再继续 read（没有读完缓冲区内的所有数据），这时应用层包不完整，server 无法回应 client，而 epoll 不会再提示 `EPOLLIN` 事件，这条连接直接 hang 住。

所以对于 ET 而言，应用层需要使用 `Non-Blocking IO` 进行读，直到返回 `EAGAIN` 为止。

#### LT

LT 模式看起来没这个问题，反正没读完会一直提示。但是因为某些原因，LT 研究需要使用 `Non-Blocking IO`。

关于原因，网上很多人举例子会说：假设 socket 缓冲区只有 100 byte，但是 read 试图读 200 byte，这时会阻塞，其实这时错误的。

经过实际实验，阻塞 `read` 和 `write` 的语义如下：

**read** :

- 如果缓冲区有数据，read 将会将缓冲区中的所有数据（或者等于指定读的数目）返回，返回值可以小于 read 中的大小参数。
- 如果缓冲区没有数据，read 将会阻塞，直到缓冲区来了数据，此时全部读出，立即返回。

**write** :

- write 会一直阻塞到指定大小的数据全部写入缓冲区为止。也就是说，如果缓冲区可写 100 byte，试图 write 200byte，则 write 调用会阻塞到 200 byte 全部写入缓冲区为止。

所以，LT 模式依旧要用 `Non-Blocking IO` 的原因为：

1. 对于 `write` 而言，如果指定 `write` 的数据大小大于写缓冲区的大小，将阻塞至写完为止。
2. 对于 `read` 而言，`epoll` 返回 `EPOLLIN`，不代表一定可写，见 man [select](http://man7.org/linux/man-pages/man2/select.2.html).

	![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102907.png)
	
### Epoll Implementation

Epoll 的实现原理非常简洁：

- `epoll_create` 会在内核空间初始化一个红黑树和一个链表；
- `epoll_ctl` 会将对应的 `socket` 插入到红黑树中，同时设置目标文件的事件回调函数；
- `epoll_wait` 会观察链表是否非空，如果非空则返回给用户；如果空，则阻塞等待，当某个 `fd` 上的事件来了，会调用事件回调函数，将对应的 `fd` 结构加到链表中，并唤醒等待在 `epoll_wait` 的线程。

额外的一些知识：

- `epoll` 的 `LT` 和 `ET` 的实现非常清晰，每当 `epoll_wait` 将链表中的 `fdlist` 返回给用户态后，将会清空这个链表，不过对于设置为 `LT` 的 `fd`，将会重新加到这个链表中。
- `epoll_wait` 返回链表上的 `fd` 时，会再次判断该 `fd` 的事件状态，如果无效，则不会返回，也不会再次加入到链表中。

由此，最后一个核心问题：`epoll_wait` 会陷入睡眠，等待唤醒，毫无疑问这个操作是事件回调函数来做的，那么，回调函数又是什么机制呢？

这就是协议栈和硬件协助完成的：

1. 当网卡数据到来后，会通过总线向CPU发送硬件中断。
2. 内核获取操作权，执行中断处理逻辑。
3. 中断处理逻辑会确定 `socket` 的相关信息，并调用 `epoll` 注册的回调函数。

总算讨论完了 `IO Multiplexing` 和 `Non-Blocking IO`，下面我们继续我们的网络并发模型。

## 5.6 方案 5（Reactor）


```python
import socket
import select

server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_socket.bind(('', 2007))
server_socket.listen(5)
# serversocket.setblocking(0)

poll = select.poll() # epoll() should work the same
connections = {}
handlers = {}

def handle_input(socket, data):
    socket.send(data) # sendall() partial?

def handle_request(fileno, event):
    if event & select.POLLIN:
        client_socket = connections[fileno]
        data = client_socket.recv(4096)
        if data:
            handle_input(client_socket, data)
        else:
            poll.unregister(fileno)
            client_socket.close()
            del connections[fileno]
            del handlers[fileno]

def handle_accept(fileno, event):
    (client_socket, client_address) = server_socket.accept()
    print "got connection from", client_address
    # client_socket.setblocking(0)
    poll.register(client_socket.fileno(), select.POLLIN)
    connections[client_socket.fileno()] = client_socket
    handlers[client_socket.fileno()] = handle_request

poll.register(server_socket.fileno(), select.POLLIN)
handlers[server_socket.fileno()] = handle_accept

// L42
while True:
    events = poll.poll(10000)  # 10 seconds
    for fileno, event in events:
        handler = handlers[fileno]
        // L46
        handler(fileno, event)
```

这是最基本的单线程 Reactor 方案，所谓 Reactor，是 Douglas C. Schmidt （ACE作者）提出的概念。

Reactor 模式就是事件驱动模型。一般由一个 `event dispatcher`（上文提到的 IO Multiplexing）等待各类事件，事件发生后原地调用对应的 `event handler`，然后继续等待，这个循环过程也称为 `event loop`。

上面例子中的 `L42 ~ L46` 就是 `event loop`，事件发生后，交由 `handle_xxx` 处理，这样子，网络框架就实现了业务逻辑和事件的分离。对于单线程 Reactor，戈君评价如下：

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222103157.png)

图中指出，一个 event loop 对应于一个线程，但是 handle callback 往往是交由业务实现，如果业务回调运行时间很长，则性能非常不可控，因为阻塞在运行 callback 时会导致其他就绪事件无法分发。所以使用这种模型的服务往往是 IO-Bound，或者确定 callback 运行时间很短的专有服务。

Redis 就是这种模型。标准的 IO-Bound 服务，瓶颈几乎一定在网络 IO 上，单线程的 Reactor 就会有很好的性能。

## 5.7 方案 6/7（Reactor + thread per task/ work thread)

方案 6、7 都是试图解决单线程 Reactor 问题的过渡方案。

方案 6 是指不在 Reactor 线程中进行计算任务，而是每个请求到来，创建一个新的线程去计算。

方案 7 是指每个连接到来创建一个计算线程，该连接上的所有请求都由这个线程计算。

想都不用想，这两种方案又回到了一个连接一个线程，性能还不如方案 2。

## 5.8 方案 8（Reactor + thread pool）

方案 8 是解决方案 6、7 为每个请求/连接创建线程的缺陷，可以使用固定大小的线程池。网络 IO 交由 Reactor 线程处理，计算任务则交给 thread pool 处理。

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102937.png)

看上去挺美好，实际上也确实不错，在 IO 压力不大，计算任务存在阻塞（数据库请求、写文件等）非常适合这种方案。

不过来看下戈君的评价：

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222102957.png)

嗯。。。一针见血：

- 分发请求给线程池的过程是 `Highly contended`，高并发的场景下，会严重影响性能。
- IO 由 Reactor 线程管理，计算却交由其他线程，导致读写数据和处理数据不在同一个线程内，这种多线程情况下产生的 `cache bouncing` 代价不可忽视。
- 所有的 `epoll_ctl` 操作都作用在同一个红黑树上，对于短连接而言，会带来大量的 O(logn) 的添加、删除操作，并不是那么的快。

由此引发了方案 9 和方案 10.

## 5.9 方案 9/10（Reactors in threads/processes）

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222103019.png)

非常不错的多线程/进程方案。 one loop per thread + 等同于 CPU 核数的 IO Thread。

其中，一个 Main Thread 负责 Accept，剩下的 event loop 负责接收 Accept 到的连接，且原地处理接下来的 IO 及计算任务。

方案 10 （多进程 + 每个进程一个 Reactor）是 Nginx 的内置方案。连接之间无交互，工作进程之间相互独立。

## 5.10 方案 11 (Reactors + thread pool）

方案 11 是方案 8 和 方案 9 的混合，使用多个 Reactor 来处理 IO 的同时，可以使用线程池来处理计算，能够适用于大部分的场景，因为 thread pool 的数目应当是可调控的，当设为 0 时，就退化为方案 9.

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222103040.png)

## 5.11 Examples

### Muduo

上面的方案就是从中总结的，Muduo 实现了方案 9 和方案 11.

### UB - XPOOL

以线程代替进程的方案 10

### UB - EPPOOL

方案 8

### UB - SXPOOL

方案 11

### Golang && brpc

golang 的 `goroutine` 和 brpc 的 `bthread` 都是 M:N 的用户态线程库。brpc 暂且不提（只看了 bthread 的部分，还没看网络框架的部分），golang 是将 `read` 和 `write` 进行了 hook，给应用层以同步阻塞调用的假象。实际上底层依旧是由 `epoll` 来完成 `IO Multiplexing` 的工作。

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181222103102.png)

# 六、More About RPC

到现在为止，Linux 下的 TCP 网络编程范式已经一个大体的了解了，剩下的则是应用层需要考虑的额外问题。

在 RPC 场景下，有一些问题值得额外关注，如：Logging、Buffer、Timer 等，这里就不做过多讨论了。