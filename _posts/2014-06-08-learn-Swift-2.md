---
title:  Swift 学习笔记 「基本操作符」
layout: post
tags: [Program]
---

这部分的内容大多和已经学过的语言如 C 、Python 类似，就不一一列出了。

###赋值运算符
	
需要注意的是 `a = b` 是不返回任何值的。所以 `if x = y {...}` 是错误的。
	
###取余运算符

```
var remainder = a % b
```
基本逻辑是公式如下:

```
a = (b * some multiplier) + remainder
```
	
例：
	
```
9 = (4 * 2) + 1
-9 = （4 * -2）+（-1）
```
	
###区间运算符

- a...b 是个闭区间
- a..b 是个左闭右开区间

---
END
