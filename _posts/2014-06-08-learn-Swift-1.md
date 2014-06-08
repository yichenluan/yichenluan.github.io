---
title:  Swift 学习笔记 「基础」
layout: post
tags:
  - Program
---

看了个把小时，才把第一章「The Basics」看完，它本身是没有生词的，但阅读速度就是上不去。

###变量与常量

- 声明

	常量：	`let constants = 10`
	
	变量：	`var variables = 0`
	
- 类型标注

	```
	var welcomeMessage: String
	```

- 命名

	1. 你可以使用非常多的字符来作为变量或常量的名字，其中包括 Unicode 字符。
	2. 声明并赋值了一个常量后，就不能再更改它的值了。

###注释

- 注释与 C 语言的注释风格类似，不同的在于，Swift 允许注释嵌套。

###分号

- Swift 并不需要在每个语句后面写上分号。
- 在下面这种情况下，分号是必须的。

	```
	let cat = "cat"; println(cat)
	```
	
###整数

- Swift 提供了 8， 16， 32 和 64 位的无符号和有符号整型。
- 如 8 位无符号整型是 UInt8，32 位有符号整型为 Int32
- 整数范围：你可以通过如下语句获得整数范围

	```
	let minValue = UInt8.min
	let maxValue = UInt8.max
	```
	
###数值类型转换

- Swift 中不存在隐式类型转换，所有类型转换都必须显示声明。
- 例：

	```
	let twoThousand: UInt16 = 2000
	let one: UInt8 = 1
	let twoThousandAndOne = twoThousand + UInt6(one)
	```
	
###类型别名

- 你可以使用 `typealias` 来定义类型别名
- 例：

	```
	typealias AudioSample = UInt16
	var maxAmplitudeFound = AudioSample.min
	```
	
###布尔值

- Bool 只有 true 或者 false 两种，在 Swift 中，`if 1 {...}` 是错误的语句。

###元组

- 元组内的变量可以是任何类型，并且不需要是一样的类型。
- 从元组中获取数据：

	```
	let http404Error = (404, "Not Found")
	let (statusCode, statusMessage) = http404Error
	```
	
	如果你需要忽略元组的一部分，你可以使用`_`
	
	```
	let (justTheStatusCode, _) = http404Error
	```
- 你可以为元组的元素命名

	```
	let http200Status = (statusCode: 200, description = "OK")
	```
	
###Optionals

- Optionals 指的是某个参量可能具有某个值，也可能没有值
- 例如，String 有个 toInt 的方法，如果字符串是"123"，那么能正常转换，但如果是"hello, world"，那就不能进行转换

	```
	let possibleNumber = "123"
	let convertedNumber = possibleNumber.toInt()
	```
	
	这时我们就称 convertedNumber 是个"optional Int"或"Int?"
	
- 你可以使用 if 来判断参变量是否具有值
- 当你确定某个 optional 确实具有某个值，可以使用感叹号来获取该值

	```
	let possibleString: String? = "An optional string."
	println(possibleString!)
	```
	
###断言

- 断言会在运行时判断一个逻辑条件是否为 true，这样，你就可以使用断言来保证在运行其他代码前，某些重要的条件被满足。
- 例:

	```
	let age = -3
	assert(age >= 0, "Error")
	```
	
	上面的断言说明只有在满足 `age >= 0` 的情况下，才会正常继续执行，不然，就会中断并显示后面的信息。
	
- 断言信息可以被省略，如：

	```
	assert(age >= 0)
	```