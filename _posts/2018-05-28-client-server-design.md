---
layout: post
title: 并发网络服务程序设计
subtitle: 总结 C++ 多线程服务端编程模型
tags: [Linux, Program]
---

本文总结自《Linux 多线程服务端编程》6.6.2节。这节内容价值极高，打开了通往并发网络服务程序设计的大门。只有深入理解了下面列举的多种方案的优缺点，后续学习才能有的放矢。

# 概览

下表为 Muduo 总结的 12 种常见方案。

![WechatIMG90.jpeg](http://p890o7lc8.bkt.clouddn.com/WechatIMG90.jpeg)

其中方案 5 就是单线程 Reactor 方案。方案 6、7 是过渡品，重点在于方案 8 和方案 9.

# 方案 0

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

方案 0 不是一个并发服务器，是一个 iterative 服务器，它一次只能服务一个客户。

其中，L6 ~ L13 是 Echo 服务的业务逻辑循环，L21 ~ L24 决定了它一次只能服务一个客户连接。

方案 0 是一个基准方案，后续的方案都是在保证这个循环功能不变的情况下，设法高效地同时服务多个客户端。

# 方案 1

方案 1 是 fork()-per-client，也就是 process-pre-connection。这种方案适合“计算相应的工作量远大于fork()的开销”，比如数据库服务器。

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

其中 L9 ~ L16 依旧是方案 0 中的业务逻辑代码，但是此时对于每个客户连接，都会新建一个子进程，在子进程中调用 EchoHandler.handle()，相比于方案 0，可以服务多个客户端。

在方案 1 中，业务逻辑已经初步从网络框架中分离出来，但是仍然和 IO 紧密结合。

# 方案 2

方案 2 是 thread-per-connection。伸缩性收到线程数的限制，线程太多，对操作系统的 scheduler 负担会很大。

```python
// diff echo-fork.py echo-thread.py

if __name__ == "__main__":
    listen_address = ("0.0.0.0", 2007)
    - server = ForkingTCPServer(listen_address, EchoHandler)
    + server = ThreadingTCPServer(listen_address, EchoHandler)
    server.serve_forever()
```

相比于方案 1，就是想 fork 进程改为新建线程。

注意，这里体现了将`并发策略`和`业务逻辑`相分离的思想。

# 方案 3 / 4

方案 3 (Prefork) 是对方案 1 的优化，方案 4 (Pre threaded)是对方案 3 的优化。

更多的分析（包括多进程同时 Accept 导致的惊群）见 UNP。

# 中场总结

以上几种方案均是阻塞式网络编程，程序通常阻塞在 read() 上。

解决这个问题的一个好方法是 IO multiplexing，让一个 thread of control 能处理多个连接。使用 IO 复用几乎肯定要配合 non-blocking IO，原因嘛也很简单，比如说，select 通知 socket 可写，只是说明发送缓冲区的可用字节数高于低水位，如果用户一次发送过多数据，只有一部分能够立即 copy 到缓冲区去，而剩下的如果是阻塞写，就把线程给 Block 了。另外一方面，如果 select 返回 socket 可读，但是如果 TCP 检测到这个数据分节有错误，选择了丢弃该分节，这时阻塞 read 将会 Block 线程。

另一方面，既然选择了 non-blocking IO，肯定就要使用应用层 buffer。对于可读事件来说（水平触发），必须一次性把数据全读出来，不然就反复触发 POLLIN，导致 busy-loop。对于可写事件来说，如果一次性写不完所有数据，希望由网络库来接管这个事件，这就需要 buffer 暂存数据。

回忆下 IO multiplexing 的[基本做法](https://github.com/chenshuo/recipes/blob/master/python/echo-poll.py)，业务逻辑隐藏在了一个事件的大循环中。

Doug Schmidt 指出，网络编程中有很多事务性工作，可以提取为公用的框架，用户只需填上关键的业务逻辑代码，并将回调注册到框架中，就可以实现完整的网络服务，这就是 Reactor 模式的主要思想。

# 方案 5

基本的单线程 Reactor 方案。Reactor 模式就是事件驱动模型。一般是由一个 event dispatcher（IO multiplexing）等待各类事件，待事件发生后原定调用 event handler，然后等待更多时间，所以称为 “event loop”。

![image2015-7-616_3_42.png](http://p890o7lc8.bkt.clouddn.com/image2015-7-616_3_42.png)

下面给出 Reactor 模式的雏形（玩具代码）。

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

可以看到，程序的核心就是 event loop（L42~L46），然后各个事件的处理交由注册好的 handlers。真正业务的处理函数在 handle_input 中，实现了业务和事件的分离。

# 方案 6

本方案是一个过渡方案。收到请求后，不在 Reactor 线程中计算，而是创建一个新线程去计算，如方案 5 的 handle_input 交由另外一个线程去处理，除性能问题外，这样导致的额外问题是 out-of-order，即同一个连接上结果出现的顺序是不确定的，不能保证先到达的请求先返回。

# 方案 7

为了让返回结果顺序确定，可以为每个连接创建一个计算线程，每个连接上的请求发给同一个计算线程去算。但是并发连接数受限于线程数目。

# 方案 8

为了弥补方案 6 为每个请求创建线程的缺陷，可以使用固定大小的线程池。全部的 IO 工作均在一个 Reactor 线程中完成，计算任务则交给 thread pool。模型可见下图：

![WechatIMG91.jpeg](http://p890o7lc8.bkt.clouddn.com/WechatIMG91.jpeg)

线程池的另一个作用是执行阻塞操作，如数据库查询操作。如果 IO 压力较大，一个 Reactor 处理不过来，就引入了方案 9.

# 方案 9

方案 9 是 muduo 内置的多线程方案。方案特点是 one loop per thread，由一个 main Reactor 来负责 accept 事件，然后把连接转交给某个 sub Reactor 中，这样该连接的所有操作都在 sub Reactor 中处理。

![WechatIMG92.jpeg](http://p890o7lc8.bkt.clouddn.com/WechatIMG92.jpeg)

Reactor pool 的大小通常根据 CPU 数目来决定。与方案 8 相比，方案 9 减少了进出 thread pool 的两次上下文切换，同时拥有了较好的 locality。

# 方案 10

这是 Nginx 的内置方案，如果连接之间无交互，这是非常好的选择。

# 方案 11

把方案 8 和方案 9 混合，既利用多个 Reactor 来处理 IO，又使用线程池来处理计算。模型如下：

![WechatIMG93.jpeg](http://p890o7lc8.bkt.clouddn.com/WechatIMG93.jpeg)

# 总结

以上就是 《Linux 多线程服务端编程》总结的 11 种服务端设计模型。作者的推荐是：one loop per thread + thread pool.

- event loop 用作 non-blocking IO 和定时器。
- thread pool 用来做计算。