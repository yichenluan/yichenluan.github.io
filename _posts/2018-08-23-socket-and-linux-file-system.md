---
layout: post
title: Linux in Depth - 文件系统及 Socket 源码解析
subtitle: 源码面前，了无秘密
tags: [Linux]
---

本文目标是深入探讨下 sockfd 指向的 socket 到底是啥。首先，socket 当然是文件（Unix 下又有什么不是文件呢），但是我希望能够将 socket 和普通文件联系起来，有一个明确的概念，也就是：文件在 Linux 下到底是什么？Socket 又与其有何区别？

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181224165832.png)

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

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181224165900.png)

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

![](https://sleepy-1256633542.cos.ap-beijing.myqcloud.com/20181224165918.png)

终于，socket 与 Linux 文件系统的关系，已经了然于胸。
