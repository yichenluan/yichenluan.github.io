---
title:  Swift 学习笔记 「函数」
layout: post
tags:
  - Program
---

###定义函数

```
func sayHello(personName: String) -> String {
	let greeting = "Hello, " + presonName + "!"
	return greeting
}
```

###函数参数和返回值

- 返回多个值的函数（用元组实现）

	```
	func count(string: String) -> (vowels: Int, consonants: Int) {
		...
	}
	```
	
- 外部参数名

	```
	func join(string s1: String, toString s2: String, withJoiner joiner: String)
		-> String{
		...
	}
	
	join(string: "hello", toString: "world", withJoiner: ",")
	```
	
- 外部参数名的缩写

	如果你的外部参数名就和形参名一样的话，在形参前加上 # 后即可。
	
	```
	func containsCharacter(#string: String, #characterToFind: Character) -> Bool{
		...
	}
	```
	
- 默认参数值

	```
	func join(string s1: String, toString s2: String, withJoiner joiner: String = " ") -> String{
		...
	}
	```
	
- 带默认参数值的外部参数名

	即使你没给带默认参数值的参数一个外部参数名， Swift 会自动给它一个相同名字的外部参数名。
	
- 可变参数

	```
	func arithmeticMean(numbers: Double...) -> Double {
		...
	}
	
	arithmeticMean(1, 2, 3, 4, 5)
	arithmeticMean(3, 8, 19)
	```
	
	可变参数是以一个数组的形式传递过去。
	需要注意，一个函数至多只能有一个可变参数，并且一定是最后一个参数。
	
- 变量参数

	默认传递给函数的参数都是常量参数，如果你想改变参数的值，你需要使用变量参数，即在参数前加上 var
	
	```
	func alignRight(var string: String, count: Int, pad: Character) -> String {
		...
	}
	```
	
- 输入输出参数

	即使是变量参数，也只能在函数体内被改变，如果你想让函数修改参数的值，你需要在参数前加上 inout，并且在调用函数时，在实参前加上 &
	
	```
	func swapTwoInts(inout a: Int, inout b: Int) {
		...
	}
	
	swapTwoInts(&someInt, &anotherInt)
	```
	
###函数类型

```
func addTwoInts(a: Int, b: Int) -> Int {
	...
}
```


```
var mathFunction:(Int, Int) -> Int = addTwoInts
```

- 函数类型作为参数类型

	```
	func printMathResult(mathFunction: (Int, Int) -> Int, a: Int) {
		...
	}
	```
	
- 函数类型作为返回类型

	```
	func chooseStepFunction(backwards: Bool) -> (Int) -> Int {
		...
	}
	```
	
---
END