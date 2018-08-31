---
layout: post
title: OS in Depth
subtitle: 源码面前，了无秘密
tags: [OS]
---

所谓程序员的三大浪漫：操作系统、编译原理、图形学，其出处已不可考。

我对图形学没有任何认知，不过看到 Miloyip 给出的[游戏程序员路线图](https://miloyip.github.io/game-programmer/game-programmer-zh-cn.pdf)中计算机图形学那一节的篇幅，也打消了一窥其究竟的想法。

编译原理的了解也非常粗浅，龙书看了全忘，SICP 读了一半，3 年前用 Lisp 写的八皇后问题（[构造数据抽象](http://jinke.me/2015-10-21-SICP-2/)）现在看已经完全看不懂了。不过一想到，要是写了个 Lisp 的解释器，以后论坛约架就方便了，所以等闲下来再把 SICP 拎出来读。

但操作系统不一样，谁还没脑子发热，想过要自己写一个操作系统的时候呢？和操作系统相关（不包括 APUE 这种）的书我读过：[《编码》](https://book.douban.com/subject/4822685/)、[《CSAPP》](https://book.douban.com/subject/26912767/)、[《现代操作系统》](https://book.douban.com/subject/3852290/)、[《Operating Systems: Three Easy Pieces》](http://pages.cs.wisc.edu/~remzi/OSTEP/)；还上了一门 Coursera 上北大的[《操作系统原理》](https://www.coursera.org/learn/os-pku)。这里面不推荐《现代操作系统》，概念太多，教学组织不合理，重要的不重要的一股脑全塞给你；极其不推荐《操作系统原理》这门课，简直浪费生命。

`OS in Depth` 这个系列我会在计算机科班的基础知识上（我不会再像大学老师一样解释进程、线程的概念），读薄、读厚与操作系统相关的知识。