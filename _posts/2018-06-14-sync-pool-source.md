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

文档中提到，sync.Pool 是一个存储临时对象的池，如果 Pool 持有某个对象的唯一引用，那么这个对象随时可能被析构掉。下面来看下标准库中 fmt 是如何使用 sync.Pool 来暂存 buffer。

```go
type buffer []byte

type pp struct {
    buf buffer
    //...
}

var ppFree = sync.Pool{
    New: func() interface{} { return new(pp) },
}

func newPrinter() *pp {
    p := ppFree.Get().(*pp)
    // Do more work.
    return p
}

func (p *pp) free() {
    p.buf = p.buf[:0]
    // Do more work.
    ppFree.Put(p)
}
```

初始化 Pool 对象时传入的 New 参数就是新建需要暂存对象的初始化函数。关于 sync.Pool 的使用就不过多赘述了。

# Struct

先把 Pool 涉及到的结构列出来：

```go
type Pool struct {
    noCopy      noCopy
    // 实际指向的是 [P]poolLocal
    local       unsafe.Pointer // 指针任意结构的指针，可以理解为 void*
    localSize   uintptr     // typedef uint64   uintptr;
    New         func() interface{}
}

type poolLocal struct {
    poolLocalInternal

    // 字节对齐，避免 false sharing
    // 关于 false sharing，后续有博客哟
    pad [128 - unsafe.Sizeof(poolLocalInternal{}) % 128]byte
}

type poolLocalInternal struct {
    // 只能由当前 P (P 是 Go 中的概念，理解为附着在线程上的 Goroutine 调度器) 使用
    private interface{}
    // 可以被其他 P 使用
    shared []interface{}
    Mutex
}
```

先大概说下 Pool 的整体结构，当我们新建一个 Pool 对象，实际真正存储数据的是 local 指针指向的 []poolLocal对象，更多的信息请往下阅读。

# Get

源码如下（把 race 相关的去掉了）：

```go
func (p *Pool) Get() interface{} {
    // l : *poolLocal
    l := p.pin()
    // 这里为什么可以不用锁呢？
    // 因为 p.pin() 中将 goroutine 设为了不可抢占。
    x := l.private
    l.private = nil
    // 允许抢占
    runtime_procUnpin()
    // 如果 private 为空
    if x == nil {
        l.Lock()
        // 从 shared 中试着取一个
        last := len(l.shared) - 1
        if last >= 0 {
             x = l.shared[last]
             l.shared = l.shared[:last]
        }
        l.Unlock()
        // 如果 shared 也没有可用对象了
        if x == nil {
            // 调用 getSlow(), 从其他 poolLocal.shared 中试着取一个。
            x = p.getSlow()
        }
    }
    // 分别在自己的 private，自己的 shared，别的 shared 中都没有取到。
    if x == nil && p.New != nil {
        // 只能 new 了
        x = p.New()
    }
    return x
}
```

在 Struct 中可以看到每个 P 对应的 poolLocal 分别有 privated 和 shared，对应的 Get() 过程就是从: self_private -> self.shared -> other_shared 挨个试着获取对象的过程。下面来看下一些细节信息：

```go
// pin 将当前 goroutine 固定到 P 上，禁止抢占，返回该 P 对应的 poolLocal。
func (p *Pool) pin() *poolLocal {
    // pid 就是 P 的唯一 ID
    pid := runtime_procPin()
    // s 代表 []poolLocal 的 size
    s := atomic.LoadUintptr(&p.localSize)
    l := p.local
    if uintptr(pid) < s {
        // 说明空间已分配好，直接返回
        return indexLocal(l, pid)
    }
    // 否则，可能需要make
    return p.pinSlow()
}

func (p *Pool) pinSlow() *poolLocal {
    // Retry
    //...
    // GC 相关，见下
    if p.local == nil {
        allPools = append(allPools, p)
    }

    // []poolLocal 的 size 就是 GOMAXPROCS
    size := runtime.GOMAXPROCS(0)
    local := make([]poolLocal, size)
    atomic.StorePointer(&p.local, unsafe.Pointer(&local[0]))
    atomic.StoreUintptr(&p.localSize, uintptr(size))
    return &local[pid]
}

// func getSlow() 就是从 []poolLocal 中其他 poolLocal.shared 中尝试获取对象的过程。
```

# Put

直接看源码：

```go
func (p *Pool) Put(x interface{}) {
    if x == nil {
        return
    }

    l := p.pin()
    if l.private == nil {
        // 这里不用锁的原因与 Get 一样
        l.private = x
        x = nil
    }
    runtime_procUnpin()
    if x != nil {
        l.Lock()
        l.shared = append(l.shared, x)
        l.Unlock()
    }
}
```

相比于 Get，Put 的代码就简单多了，对于 Put 而言，也是先取出该 goroutine 所在 P 对应的 poolLocal，如果 poolLocal.private == nil，则赋值为 x，否则放到 shared 中。

# GC

上问提到，当 Pool 持有对象的唯一引用时，这个对象随时可能被析构掉，那么具体何时呢？

```go
// 当 stop the world 来临，在 GC 之前会调用该函数
func poolCleanup() {
    for i, p := range allPools {
        // 置 allPools 所有成员为 nil
        // ..
    }
    allPools = []*Pool{}
}

func init() {
    runtime_registerPoolCleanup(poolCleanup)
}
```

可以看到，sync.Pool 实际上是每次 GC 来临都会清空掉所有的 Pool 以及在 Pool 中的对象。
