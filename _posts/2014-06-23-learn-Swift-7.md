---
title:  Swift 学习笔记 「闭包」
layout: post
tags:
  - Program
---

###Sort 函数

sort 函数需要传入两个参数：

- 已知类型的数组
- 闭包函数，该闭包函数需要传入与数组类型相同的两个值，并返回一个布尔类型值来告诉sort函数当排序结束后传入的第一个参数排在第二个参数前面还是后面。如果第一个参数值出现在第二个参数值前面，排序闭包函数需要返回true，反之返回false。

一种方式是直接写一个普通函数

```
let names = ["Chris", "Alex", "Barry"]

func backwards(s1: String, s2: String) -> Bool {
    return s1 > s2
}
var reversed = sort(names, backwards)
```

###闭包表达式语法

```
{ (parameters) -> returnType in
    statements
}
```

所以改写如下：

```
reversed = sort(names, { (s1: String, s2: String) -> Bool in return s1 > s2 } )
```
###根据上下文推断类型

```
reversed = sort(names, { s1, s2 in return s1 > s2 } )
```

###单表达式闭包隐式返回

```
reversed = sort(names, { s1, s2 in s1 > s2 } )
```

###尾随闭包

可以增强可读性

```
func someFunctionThatTakesAClosure(closure: () -> ()) {
    // 函数体部分
}

// 以下是不使用尾随闭包进行函数调用
someFunctionThatTakesAClosure({
    // 闭包主体部分
})

// 以下是使用尾随闭包进行函数调用
someFunctionThatTakesAClosure() {
  // 闭包主体部分
}
```

---
END



