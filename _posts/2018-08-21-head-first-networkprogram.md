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

接下来我们需要仔细探讨下 sockfd 指向的 socket 到底是啥。首先，socket 当然是文件（Unix 下又有什么不是文件呢），但是我希望能够将 socket 和普通文件联系起来，有一个明确的概念，也就是：文件在 Linux 下到底是什么？Socket 又与其有何区别？

![](http://p890o7lc8.bkt.clouddn.com/20180823145034.png)

接下来的讨论假定读者已经阅读过 APUE 第三章，对 Unix 文件系统有基本了解。然后我们再来看 Linux 下的文件系统结构。

首先，在 Linux 下，每个进程都有对应的一个 [task_struct](https://github.com/torvalds/linux/blob/master/include/linux/sched.h) （include/linux/sched.h）结构体来管理进程。这个结构体的代码长达 617 行，几乎是内核中最复杂的一个结构体了，鉴于其究极复杂程度，这里只摘要出我们所关心的和文件系统相关的代码：

```c
struct task_struct {
	// ...
	/* Filesystem information: */
	struct fs_struct		*fs;

	/* Open file information: */
	struct files_struct		*files;
	// ...
}
```

我们先看 [fs_struct](https://github.com/torvalds/linux/blob/ead751507de86d90fa250431e9990a8b881f713c/include/linux/fs_struct.h)（include/linux/fs_struct.h)：

```c
struct fs_struct {
	int users;
	spinlock_t lock;
	seqcount_t seq;
	int umask;
	int in_exec;
	struct path root, pwd;
}
// inlcude/linux/path.h
struct path {
	// ...
	struct vfsmount *mnt;
	struct dentry *dentry;
}
```
这个结构比较简单，里面保存的是当前文件系统的信息，从变量名可以看出，还保存了 root 和 pwd 信息。其中的 dentry 我们暂且不管。

然后再看 [files_struct](https://github.com/torvalds/linux/blob/35277995e17919ab838beae765f440674e8576eb/include/linux/fdtable.h)（include/linux/fdtable.h），同样的，我们将细节代码省去：

```c
struct files_struct {
	// ...
	struct fdtable __rcu *fdt;
	// ...
}

struct fdtable {
	unsigned int max_fds;
	struct file __rcu **fd;      /* current fd array */
	unsigned long *close_on_exec;
	unsigned long *open_fds;
	unsigned long *full_fds_bits;
	struct rcu_head rcu;
};
```

可以看到，files\_struct 最主要的内容就是一个指针指向一个 file 结构的数组，数组的大小由 max_fds 标记。

**Note:** 注意，我们通过 open 或者 socket 拿到的 fd 其实就是这个数组的 index，到现在为止，我们的结构及信息都是进程内部私有的，`struct file __rcu **fd;` 指向的 file 结构则是所谓的全局打开文件表了。

[file](https://github.com/torvalds/linux/blob/master/include/linux/fs.h)（include/linux/fs.h）结构的关键代码如下：

```c
struct file {
	// ...
	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;

	unsigned int 		f_flags;
	fmode_t			f_mode;
	struct mutex		f_pos_lock;
	loff_t				f_pos;
}
```
其中，打开文件的状态标志及偏移量就记录在 file 结构内。我们暂时忽略 file_operations，那么，其中最关键的结构就是 f\_path 中包含的 dentry 及大名鼎鼎的 inode 了。

我们先来看 [dentry](https://github.com/torvalds/linux/blob/ad1d69735878a6bf797705b5d2a20316d35e1113/include/linux/dcache.h)（include/linux/dcache.h）：

```c
struct dentry {
	struct dentry *d_parent;	/* parent directory */
	struct qstr d_name;
	struct inode *d_inode;		/* Where the name belongs to - NULL is
					 				* negative */

	const struct dentry_operations *d_op;
	
	struct super_block *d_sb;	/* The root of the dentry tree */
	struct list_head d_child;	/* child of parent list */
	struct list_head d_subdirs;	/* our children */
}
```

dentry 代表的就是操作系统描述文件系统时，给出的目录项的概念。我们可以想象一个文件拥有很多信息，包括文件长度、磁盘位置、创建时间、更新时间等，这些信息 Linux 由 inode 结构来管理，同时，由于符号链接的存在，每个物理文件可以由多个逻辑文件名，所以和文件名相关信息交由 dentry 结构管理。同样的，我们暂时忽略 dentry_operations 这个结构。

其中 d_inode 指针指向对应的 inode 结构。[inode](https://github.com/torvalds/linux/blob/master/include/linux/fs.h)（include/linux/fs.h）结构的关键代码如下：

```c
struct inode {
	// ...
	const struct inode_operations	*i_op;
	
	struct super_block	*i_sb;
	loff_t				i_size;	// 文件大小，字节数
	blkcnt_t            i_blocks; // 文件大小，块数
	struct timespec64	i_atime;
	struct timespec64	i_mtime;
	struct timespec64	i_ctime;
}
```
如上文所述，关于文件的大部分信息都保存在 inode 结构中。到目前为止，我们忽略了 `file_operations`、`dentry_operations`、`inode_operations`，我们现在来看下这些结构是干嘛的：

```c
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	// ...
};

struct dentry_operations {
	int (*d_delete)(const struct dentry *);
	int (*d_init)(struct dentry *);
	char *(*d_dname)(struct dentry *, char *, int);
	// ...
};

struct inode_operations {
	int (*create) (struct inode *,struct dentry *, umode_t, bool);
	int (*link) (struct dentry *,struct inode *,struct dentry *);
	int (*unlink) (struct inode *,struct dentry *);
	int (*symlink) (struct inode *,struct dentry *,const char *);
	int (*mkdir) (struct inode *,struct dentry *,umode_t);
	int (*rmdir) (struct inode *,struct dentry *);
	// ...
};
```

这样子一下子就清晰了，operations 这种结构就是保存这各类适用于该结构的函数指针，具体函数实现再交由如 ext2 这样的具体文件系统来实现。

到现在为止，我们终于对 Linux 文件系统有一个较为清晰的认识了：

![](http://p890o7lc8.bkt.clouddn.com/20180823174916.png)

----

讲了这么多，我们总算可以继续探讨最开始的问题：socket 到底是啥？

当然，socket 是一个文件，不同于一开始，我们现在可以从进程到 inode 都有一个非常正确的认知，不过，我们还需要继续思考，socket 又是如何与 inode 联系起来的？

答案要回到我们最初的调用 `int socket(...)` 中，最终这个函数会调用 `sock_alloc_inode(...)` 来创建 inode，[sock\_alloc\_inode](https://github.com/torvalds/linux/blob/9a76aba02a37718242d7cdc294f0a3901928aa57/net/socket.c)（net/socket.c）中我们关心的代码如下：

```c
static struct inode *sock_alloc_inode(struct super_block *sb)
{ // Linux 是大括号换行党 ==!
	struct socket_alloc *ei;
	// ...
	ei = kmem_cache_alloc(sock_inode_cachep, GFP_KERNEL);
	// ...
	return &ei->vfs_inode;
};
```

看代码发现，虽然这个函数是要创建 inode，实际上函数里却创建了 `socket_alloc` 结构，[socket_alloc](https://github.com/torvalds/linux/blob/master/include/net/sock.h)（include/net/sock.h）：

```c
struct socket_alloc {
	struct socket socket;
	struct inode vfs_inode;
};
```

继续看看 [socket](https://github.com/torvalds/linux/blob/master/include/linux/net.h)（include/linux/net.h）结构：

```c
struct socket {
	socket_state		state;

	short			type;

	unsigned long		flags;

	struct socket_wq	*wq;

	struct file		*file;
	struct sock		*sk;
	const struct proto_ops	*ops;
};
```

现在看到 `proto_ops` 想必不会疑惑了，来看看在 `socket` 这个结构上，定义了哪些函数指针：

```c
struct proto_ops {
	int		family;

	int		(*bind)	     (struct socket *sock,
				      struct sockaddr *myaddr,
				      int sockaddr_len);
	int		(*connect)   (struct socket *sock,
				      struct sockaddr *vaddr,
				      int sockaddr_len, int flags);
	int		(*accept)    (struct socket *sock,
				      struct socket *newsock, int flags, bool kern);
	__poll_t	(*poll)	     (struct file *file, struct socket *sock,
				      struct poll_table_struct *wait);
	int		(*listen)    (struct socket *sock, int len);
	int		(*shutdown)  (struct socket *sock, int flags);
	// ....
};
```

嗯。。。至少这些函数看起来就非常熟悉了。

另外，`socket` 结构中，另一个重要结构是 `sock`，下面把 [sock](https://github.com/torvalds/linux/blob/master/include/net/sock.h)（include/net/sock.h)感兴趣的字段列出来：

```c
// sock 结构非常复杂，贯穿各层
struct sock {
	// 待读包队列
	struct sk_buff_head	sk_receive_queue;
	// receive buffer size
	int			sk_rcvbuf;
	// send buffer size
	int			sk_sndbuf;
	// 待写包队列
	struct sk_buff_head	sk_write_queue;
	// ...
};
```
其中，sk_buff 是网络数据报在内核中的表现形式。不过我们不再往下探究了。

剩下的最后一个问题就是，内核又是如何通过 inode 获得对应的 socket 的呢?

答案在 [SOCKET_I](https://github.com/torvalds/linux/blob/master/include/net/sock.h)（include/net/sock.h）这个函数里：

```c
static inline struct socket *SOCKET_I(struct inode *inode)
{
	return &container_of(inode, struct socket_alloc, vfs_inode)->socket;
}
```

![](http://p890o7lc8.bkt.clouddn.com/20180725194555.png)

终于，socket 与 Linux 文件系统的关系，已经了然于胸。

## 3.2 Connect
