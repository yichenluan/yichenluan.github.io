---
layout: post
title: 并发网络服务程序设计方案
tags: [Linux, Program]
---

本文总结自《Linux 多线程服务端编程》6.6.2节。这节内容价值极高，打开了通往并发网络服务程序设计的大门。只有深入理解了下面列举的多种方案的优缺点，后续学习才能有的放矢。

# 概览

下表为 Muduo 总结的 12 种常见方案。

![WechatIMG90.jpeg](http://p890o7lc8.bkt.clouddn.com/WechatIMG90.jpeg)

其中方案 5 就是单线程 Reactor 方案。方案 6、7 是过渡品，重点在于方案 8 和方案 9.

# 方案 0

方案均以粗糙的 Python 代码为例。

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

To be continued...