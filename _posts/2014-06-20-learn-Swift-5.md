---
title:  Swift 学习笔记 「控制流」
layout: post
tags: [Program]
---

###For 循环

- for-in 循环

	```
	for index in 1...5 {...}
	```
	
	如果你不需要循环中产生的值，你可以用下划线来忽视它
	
	```
	for _ in 1...power{...}
	```
	
- for-condition-increment 循环

###Switch

- 每个 case 下都必须至少有一个执行语句。
- 范围匹配

	```
	let count = 3000
	switch count{
	case 0:
		...
	case 1...9:
		...
	case 10...99:
		...
	case 100...999:
		...
	default:
		...
	}	
	```
	
- 元组匹配，你可以用下划线来匹配任何可能的值

	```
	let somePoint = (1, 1)
	switch somePoint {
	case(0, 0):
		...
	case(_, 0):
		...
	case(0, _):
		...
	case( -2...2, -2...2):
		...
	default:
		...
	}
	```
	
- 绑定值

	```
	let anotherPoint = (2, 0)
	switch anotherPoint {
	case(let x, 0):
		...
	case(0, let y):
		...
	case let(x, y):
		...
	}
	```
	
- where：switch 中的 case 可以通过 where 来进行更多的判断条件

	```
	let yetAnotherPoint = (1, -1)
	switch yetAnotherPoint {
	case let(x, y) where x == y:
		...
	case let(x, y) where x == -y:
		...
	case let(x, y):
		...
	}
	```
	
###Fallthrough

使用 fallthrough 来获得和 C 中 switch 一样的体验。

```
let interger = 5
switch interger{
case 5:
	...
	fallthrough
default:
	...
}
```

###带标签的语句

给语句带上标签，可以是 continue 和 break 直接影响标签循环

```
gameLoop: while square != finalSquare {
	switch square + diceRoll {
	case finalSquare:
		...
		break gameLoop
	case let newSquare where newSquare > finalSquare:
		...
		continue gameLoop
	default:
		...
	}
}
```

---
END

