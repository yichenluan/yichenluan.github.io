---
title:  Swift 学习笔记 「枚举」
layout: post
tags:
  - Program
---

###定义

```
class SomeClass {
    // class definition goes here
}
struct SomeStructure {
    // structure definition goes here
}
```

```
struct Resolution {
    var width = 0
    var heigth = 0
}
class VideoMode {
    var resolution = Resolution()
    var interlaced = false
    var frameRate = 0.0
    var name: String?
}
```

###类和结构体

结构体和类都使用构造器语法来生成新的实例：

```
let someResolution = Resolution()
let someVideoMode = VideoMode()
```


###属性访问

```
someVideoMode.resolution.width = 12880
println("The width of someVideoMode is now (someVideoMode.resolution.width)")
```

###结构体类型的成员逐一构造器

```
let vga = resolution(width:640, heigth: 480)
```

###结构体和枚举是值类型

值类型被赋予给一个变量，或本省被传递给一个函数的时候，实际上操作的是它的拷贝，在 Swift 中，所有的基本类（整数、浮点数。。。）都是值类型。

###类是引用类型

引用类型被赋予给别的变量后，操作的是它的引用而不是拷贝。

```
let tenEighty = VideoMode()
tenEighty.resolution = hd
tenEighty.interlaced = true
tenEighty.name = "1080i"
tenEighty.frameRate = 25.0

let alsoTenEighty = tenEighty
alsoTenEighty.frameRate = 30.0

println("The frameRate property of tenEighty is now \(tenEighty.frameRate)")
// 输出 "The frameRate property of theEighty is now 30.0"

```

###恒等运算符

通过恒等运算符来判定两个常量或者变量是否引用同一个类实例：

- 等价于（===）
- 不等价于（!===）

```
if tenEighty === alsoTenTighty {
    println("tenTighty and alsoTenEighty refer to the same Resolution instance.")
}
//输出 "tenEighty and alsoTenEighty refer to the same Resolution instance."
```

###集合类型的赋值和拷贝行为

这里的集合指的是数组和字典，在 Swift 中，数组和字典都是通过结构体的形式来实现。然而字典是值类型，而数组却比较复杂

- 数组的赋值和拷贝行为

	默认情况下，数组进行的不会进行值拷贝的，拷贝行为仅仅当操作有可能修改数组长度时才会发生。
	
	```
	var a = [1, 2, 3]
	var b = a
	var c = a
	
	println(a[0])
	// 1
	println(b[0])
	// 1
	println(c[0])
	// 1
	
	a[0] = 42
	println(a[0])
	// 42
	println(b[0])
	// 42
	println(c[0])
	// 42
	
	a.append(4)
	a[0] = 777
	println(a[0])
	// 777
	println(b[0])
	// 42
	println(c[0])
	// 42
	```
	
- 确保数组的唯一性

	通过 unshare 方法来保证数组的唯一性
	
	```
	b.unshare()
	
	b[0] = -105
	println(a[0])
	// 77
	println(b[0])
	// -105
	println(c[0])
	// 42
	``` 
	
	
- 强制复制数组

	我们可以通过调用数组的 copy 方法来进行强制显式复制。
	
	```
	var names = ["Mohsen", "Hilary", "Justyn", "Amy", "Rich", "Graham", "Vic"]
	var copiedNames = names.copy()
	```


---
END 


















