---
layout: post
title: Go in Depth - sync.Pool 源码解析
subtitle: 源码面前，了无秘密
tags: [Go]
---

如果读者看过 Go 标准库中 RPC 的源码，就知道 RPC 中将 Request 和 Response 结构通过一个带锁链表进行对象复用，构造一个对象池，以避免 GC 造成的性能代价。下面是概括性的源码：

```go
type Request struct {
	ServiceMethod 	string
	Seq 				uint64
	next 				*Request
}

type Server struct {
	reqLock 		sync.Mutex
	freeReq 		*Request
}

func (server *Server) getRequest() *Request {
	server.reqLock.Lock()
	req := server.freeReq
	if req == nil {
		req = new(Request)
	} else {
		server.freeReq = req.next
		*req = Request{}
	}
	server.reqLock.Unlock()
	return req
}

func (server *Server) freeRequest(req *Request) {
	server.reqLock.Lock()
	req.next = server.freeReq
	server.freeReq = req
	server.reqLock.Unlock()
}
```
可以看到，这个对象池就是一个用链表实现的对象栈，每次取对象就从中取头结点，如果头结点为空，则 new 一个，释放对象也就是把对象插入到链表的头部。

那么，这个超简易版的对象池性能如何呢，其实不难分析，性能是不错的，因为在压力平稳后，几乎不需要 new 对象，这样锁的临界区都是对链表节点指针的操作。

但是实际上，我在异步日志库项目中，使用这种对象池方法和使用 sync.Pool 来构建对象池，性能差距非常大，Benchmark 显示，sync.Pool 性能能优于带锁链表 10 倍以上。

太阳底下没有新鲜事，想要优化锁的代价，要么减小临界区，要么分散锁竞争，下面看下标准库中 sync.Pool 是如何做的。

- Go 版本：1.9.2
- 源码文件：`sync/pool.go`

# Usage

TODO

