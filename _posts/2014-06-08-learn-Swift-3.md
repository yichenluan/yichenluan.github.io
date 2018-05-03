---
title:  Swift 学习笔记 「字符串和字符」
layout: post
tags: [Program]
---

###字符串

- 使用 `countElements` 函数来统计字符串中字符的个数

	```
	let message = "hello, world!"
	println("message has \(countElements(message)) characters"		
	```
- 前缀和后缀：`hasPrefix` `hasSuffix`

	```
	var act1SceneCount = 0
	for scene in romeoAndJuliet {
		if scene.hasPrefix("Act 1") {
			++act1SceneCount
		}
	}
	```



###字符

- 声明字符

	```
	let yenSign: Character = "$"
	```

- 你可以通过 `+` 直接合并字符串和字符
