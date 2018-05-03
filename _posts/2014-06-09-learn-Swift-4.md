---
title:  Swift 学习笔记 「集合类型」
layout: post
tags: [Program]
---

###数组

- 声明

	```
	var shoppingList: String[] = ["Eggs", "Milk"]
	```
	
	但由于 Swift 的类型推导，你这么写也是可以的。
	
	```
	var shoppingList = ["Eggs", "Milk"]
	```
	
- 方法

	确定数组内元素的个数
	
	```
	shoppingList.count
	```
	在数组最后添加一个元素
	
	```
	shoppingList.append("Flour")
	```
	
	也可以
	
	```
	shoppingList += "Baking Powder"
	```
	
	替换元素，Swift 允许替换范围大于替换后的元素个数
	
	```
	shoppingList[4...6] = ["Bananas", "Apples"]
	```
	
	在具体某个位置添加元素
	
	```
	shoppingList.insert("Maple Syrup", atIndex: 0)
	```
	
	移除具体某个位置的元素
	
	```
	let mapleSyrup = shoppingList.removeAtIndex(0)
	```
	
	获取数组下标和对应的值
	
	```
	for (index, value) in enumerate(shoppingList){...}
	```
	
###字典

- 声明

	```
	var airports: Dictionary<string, string> = ["TYO":"Tokyo", "DUB":"Dublin"]
	
	var namesOfIntegers = Dictionary<Int, String>()
	```
	
- 方法

	统计字典元素个数
	
	```
	airports.count
	```
	
	更新字典值，如果不存在该 key ，则会返回 nil
	
	```
	let oldValue = airports.updateValue("Dublin International", forKey: "DUB")
	```
	
	删除字典值
	
	```
	let removedValue = airports.removeValueForKey("DUB")
	```
	
	获取字典 key
	
	```
	let airportCodes = Array(airports.keys)
	for airportCode in airports.keys{...}
	```
	
	获取字典 value
	
	```
	let airportName = Array(airports.values)
	for airportName in airports.values{...}
	```
