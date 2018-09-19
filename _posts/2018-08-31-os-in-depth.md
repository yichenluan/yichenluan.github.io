---
layout: post
title: OS in Depth - Introduction
subtitle: Just for Fun
tags: [OS]
---

所谓程序员的三大浪漫：操作系统、编译原理、图形学，其出处已不可考。

我对图形学没有任何认知，不过看到 Miloyip 给出的[游戏程序员路线图](https://miloyip.github.io/game-programmer/game-programmer-zh-cn.pdf)中计算机图形学那一节的篇幅，也打消了一窥其究竟的想法。

编译原理的了解也非常粗浅，龙书看了全忘，SICP 读了一半，3 年前用 Lisp 写的八皇后问题（[构造数据抽象](http://jinke.me/2015-10-21-SICP-2/)）现在已经完全看不懂了。不过一想到，要是写了个 Lisp 的解释器，以后论坛约架就方便了，所以等闲下来再把 SICP 拎出来读。

但操作系统不一样，谁还没脑子发热，想过要自己写一个操作系统的时候呢？和操作系统相关（不包括 APUE 这种）的书我读过：[《编码》](https://book.douban.com/subject/4822685/)、[《CSAPP》](https://book.douban.com/subject/26912767/)、[《现代操作系统》](https://book.douban.com/subject/3852290/)、[《Operating Systems: Three Easy Pieces》](http://pages.cs.wisc.edu/~remzi/OSTEP/)；还上了一门 Coursera 上北大的[《操作系统原理》](https://www.coursera.org/learn/os-pku)。这里面不推荐《现代操作系统》，概念太多，教学组织不合理，重要的不重要的一股脑全塞给你；极其不推荐《操作系统原理》这门课，简直浪费生命。

`OS in Depth` 这个系列我会在计算机科班的基础知识上（我不会再像大学老师一样解释进程、线程的概念），以 `OS:TEP` 为脉络读薄、读厚与操作系统相关的知识。

---

操作系统的最核心问题就是虚拟化（virtualization）资源。具体来说，OS 将物理资源（CPU、内存、磁盘）虚拟化为一个更抽象的系统。另一方面，为了让用户能够控制系统，OS 还会提供一系列系统调用（system calls）作为接口（API）。虚拟化的目的就是为了多个程序能够共享资源，同时运行，所以 OS 还要充当资源管理者（resource manager）的角色。

总而言之，操作系统的核心目的就是虚拟化物理资源（CPU、内存、磁盘等）以便用户程序的运行，并在这个过程中，管理这些资源。

### Virtualizing the CPU

所谓虚拟化 CPU 就是给进程一个独占 CPU 运行的假象。为了做到这件事，需要解决很多问题，其中有些是最根本的思想及实现，比如操作系统到底是如何实现这种假象的，我们称之为机制（mechanisms），还有些在机制之上，更细节的问题，比如说多个进程都试图运行，操作系统需要决定谁先谁后，这种称为方案（policy），我们后续的学习就是抓住最根本的 mechanisms，看不同 policy 的思考与选择。

### Virtualizing Memory

类似的，虚拟化内存就是给进程一个独占所有内存的假象。操作系统给每个进程一个虚拟地址空间，进程在虚拟地址空间中操作内存，由操作系统完成虚拟地址和物理地址的转换，当然，除此之外，另一个关键问题就是操作系统如何管理物理内存这个有限的资源。

### Concurrency

操作系统本身就是一个并发系统，所以如何处理并发是一个最基础的问题。后续我们将看到操作系统为此提供了什么抽象，以及利用这些抽象我们在用户层面对并发数据结构的思考。


### Persistence

操作系统将硬盘这种持久存储抽象为文件系统。这是另一个很大的话题。操作系统为了更有效率及更优雅的使用磁盘作出了很多努力。


### Conclusion

操作系统虚拟化（Virtualize）物理资源（CPU、内存、磁盘），同时处理由此带来的并发（Concurrency）问题，同时将文件持久化（Persistence）存储。

接下来就需要挨个去分析了，我希望做到的程度是：

1. 从概念上理解操作系统的 mechanisms；
2. 大致分析 Linux 或其他内核在这些机制上的典型实现；
3. 从概念上理解操作系统的 policy，并分析不同 policy 之间的优劣；
4. 阅读现代操作系统选择的 policy，并从源码层面从中分析思考。