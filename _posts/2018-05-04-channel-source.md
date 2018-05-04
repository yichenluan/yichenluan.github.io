---
layout: post
title: Go in Depth - Channel 源码解析
subtitle: 源码面前，了无秘密
tags: [Go]
---

我忘记在哪看到过陈硕说过，对于非内核开发者而言，看 Linux 内核源码有效的方式是只看主干逻辑，忽略错误分支，避免捡了芝麻丢了西瓜。本文及未来的源码解析都是建立在这个思想基础上的。

Channel 在使用 Go 进行并发编程中起到了非常核心的作用。Go 使用 goroutine 和 Channel 实现了 CSP 并发模型，关于 CSP，可以看这篇博客：[并发之痛 Thread，Goroutine，Actor](http://jolestar.com/parallel-programming-model-thread-goroutine-actor/)。本文无意于讨论并发编程，关注点只在 Channel 的具体实现上面，只有知道了其真正的实现方式，才能知道 Channel 能解决什么问题，不能解决什么。

- Go 版本：1.9.2
- 源码文件：`runtime/chan.go`


## Struct

每个 Channel 都是一个 `hcahn` 数据结构：

```go
type hchan struct {
	qcount   uint           // buffer中数据总数
	dataqsiz uint           // buffer的容量
	buf      unsafe.Pointer // 指向 buffer 的指针
	elemsize uint16         // channel中数据类型的大小
	closed   uint32         // channel是否关闭，0 => false，其他都是true
	elemtype *_type         // channel数据类型
	sendx    uint           // 对于 send 行为而言，该次可放入位置的 index
	recvx    uint           // 对于 recv 行为而言，该次可读取位置的 index
	recvq    waitq          // 接收 goroutine 等待队列
	sendq    waitq          // 发送 goroutine 等待队列

	lock     mutex          // 互斥锁
}
```

从这个数据结构，我们可以有个大体的印象，即 Channel 结构是一个头结点，保存一些元信息，其中有个指针指向真正存放数据的 buffer 对象，而这个 buffer 可以理解为一个数组。另外，通过 Mutex 来保护临界区的并发。

## Make

创建一个 Channel 的代码结构如下：

```go
func makechan(t *chantype, size int64) *hchan {
    elem := t.elem
    
    // 忽略了边界检查代码

    var c *hchan
    // 如果是无指针类型 || channel size 为0
    if elem.kind&kindNoPointers != 0 || size == 0 {
    	// 一次性把所需内存分配出来
    	// 注意， 这里的 hchanSize 已经是按 8 字节对齐后的大小了。
        c = (*hchan)(mallocgc(hchanSize+uintptr(size)*elem.size, nil, true))
        // size 大于 0，即分配了有效的 buffer 内存，令 c.buf 指向 buffer 数组
        if size > 0 && elem.size != 0 {
            c.buf = add(unsafe.Pointer(c), hchanSize)
        } else {
            // race detector uses this location for synchronization
            // Also prevents us from pointing beyond the allocation (see issue 9401).
            c.buf = unsafe.Pointer(c)
        }
    } else {
    	// 分两次分配内存
        c = new(hchan)
        c.buf = newarray(elem, int(size))
    }
    c.elemsize = uint16(elem.size)
    c.elemtype = elem
    c.dataqsiz = uint(size)

    return c
}
```
有一个地方还没有想明白：

**Note:** 区分是否一次性分配所需内存的关键在于 `kindNoPointers`，什么是所谓的无指针类型？为什么要分开考虑是否一次性分配内存？
{: .box-note}

skoo 的[博客](http://skoo.me/go/2013/09/20/go-runtime-channel)上画了一张图，较好的总结了整个 Channel 的结构：
![hchan](https://ws3.sinaimg.cn/large/006tKfTcly1fqzd0v4al5j30zk0dit9p.jpg)
当然，要注意， Hchan 这个头不一定和后面的 buffer 在内存分布上是连续的。（话又说回来，即使逻辑上连续，到真实物理内存上，也不一定连续，我说这个干嘛。。。）

## Send

当我们在代码中写 `ch <- x` ，实际上是调用了 `chansend` 函数，该函数代码结构为：

```go
// ep: 数据的指针
// block：是否阻塞，详见下文
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
   // 往 nil 发数据
    if c == nil {
    	// 非阻塞情况下，直接返回 false
        if !block {
            return false
        }
        // gopark 将当前 goroutine 休眠，等待第一个参数唤醒，但这里传入的是nil，所以会一直休眠下去。
        gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
        throw("unreachable")
    }
    // ...
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
        (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
        return false
    }
    // 开始加锁
    lock(&c.lock)

    if c.closed != 0 {
        unlock(&c.lock)
        panic(plainError("send on closed channel"))
    }
    // 情况1：buf可用数据为空，有goroutine阻塞在取数据上
    if sg := c.recvq.dequeue(); sg != nil {
        // Found a waiting receiver. We pass the value we want to send
        // directly to the receiver, bypassing the channel buffer (if any).
        // 直接将数据发给该 goroutine.
        // send 会调用 goready，用以唤醒拿到数据的 goroutine.
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }
    // 情况2：没有阻塞在获取数据的 goroutine，且 buf 内还有空间
    if c.qcount < c.dataqsiz {
        // Space is available in the channel buffer. Enqueue the element to send.
        // 获取当前可插入数据的指针
        qp := chanbuf(c, c.sendx)
        if raceenabled {
            raceacquire(qp)
            racerelease(qp)
        }
        // 复制数据
        typedmemmove(c.elemtype, qp, ep)
        // send index 前移
        c.sendx++
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        // 可用数据数增加
        c.qcount++
        unlock(&c.lock)
        return true
    }
    // 情况3. buf 内空间已满
    // 非阻塞情况下，直接返回 false
    if !block {
        unlock(&c.lock)
        return false
    }
    // 否则，阻塞该 goroutine.
    // Block on the channel. Some receiver will complete our operation for us.
    gp := getg()
    mysg := acquireSudog()
    // ...
    gp.waiting = mysg
    gp.param = nil
    // 将该 goroutine 的结构放入 sendq 队列
    c.sendq.enqueue(mysg)
    // 休眠
    // 等待 goready 唤醒
    goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)
    // ...
    return true
}
```

### 关于阻塞非阻塞
一般情况下，传入的参数都是 `block=true`，即阻塞模式，一个往 channel 中插入数据的 goroutine 会阻塞到插入成功为止。

非阻塞是只这种情况：

```go
select {
case c <- v:
	... foo
default:
	... bar
}
```
这种情况下，编译器会将其改为：

```go
if selectnbsend(c, v) {
	... foo
} else {
	... bar
}
```
其中， selectnbsend 调用 chansend 时，传入的 `block=false`.

### 关于临界区

加锁解锁的代价非常低，当我们讨论锁的代价的时候，我们值得是临界区产生的锁竞争。下面分析下三种情况的临界区。

- 情况 1
	
	goroutine 之间的数据复制

- 情况 2

	数据复制到 buffer 中，更新两次（sendx / qcount）内存
	
- 情况 3

	获取当前 goroutinne，构造 mysg 结构，放入队列
 	


可以看到，情况1的临界区最小，情况3的临界区最大。
 	
## Recv

同样的，当我们代码中出现 `x <- ch`，实际最终调用的是 `chanrecv` 函数，该函数代码结构为：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// 从 nil 收数据，将会一直休眠
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, "chan receive (nil chan)", traceEvGoStop, 2)
		throw("unreachable")
	}
	// 加锁
	lock(&c.lock)
	// ...
	
	// 情况1. sendq 队列不为空
	if sg := c.sendq.dequeue(); sg != nil {
		// 取 recv 队列队首数据给当前 goroutine
		// 将 sendq 队列队首数据放入 buffer 中相同的 slot
		// 通过 goready 唤醒相应 sendq 中的 goroutine
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
	
	// 情况2. 没有阻塞在 sendq 的 goroutine，且buffer中有数据
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		// 复制数据
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		// recv index 前移
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		// 可用数据减少
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
	
	// 情况3. buffer空间为空，阻塞该 goroutine.
	gp := getg()
	mysg := acquireSudog()
	// ...
	// 将该 goroutine 相应的结构放入 recvq
	c.recvq.enqueue(mysg)
	// 休眠，等待相应的 goready 唤醒
	goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

	return true, !closed
}
```
recv 的逻辑和 send 互为表里，相互配合，各自维护一个 index 指针来指向自己的生产 / 消费队列，整体不难理解。

## Close

关闭一个 Channel 只有一些逻辑上的考量，具体逻辑如下：

1. 检查 channel 状态，不能重复关闭 channel
2. 释放 recvq 中所有的 reader，将其加入 gist 队列中
3. 释放 sendq 中所有的 sender，将其加入 gist 队列中（sender会panic）
4. 遍历 gist 队列中的结构，通过 goready 唤醒这个 goroutine.

## Summary

软件工程没有银弹，Channel 也不是万能的。在高并发写多读少的情况下，Channel 内部的那把锁产生的代价可能会非常大，这时可能就需要考虑使用其他的数据结构+共享机制了。

产生这个想法是在分析异步日志库的性能问题是发现的，多goroutine写+单goroutine读，中间只通过一个 Channel 来传递日志信息，性能损耗非常可观，关于异步日志库的问题，后续会有更多博客发出来。

---
#### 参考：
- [channel in Go's runtime](http://skoo.me/go/2013/09/20/go-runtime-channel)
- [Go Channel源码分析](https://github.com/yangyuqian/technical-articles/blob/master/go/channel-implementation-cn.md)
- [Go Channel 源码剖析](http://legendtkl.com/2017/08/06/golang-channel-implement/)


#### More:

- 博客使用 Disqus 作为评论区，需要自备梯子才能看到。