---
title:  Python中的排序
layout: post
guid: urn:uuid:6a0f8df5-6f92-4a07-996c-4061e23d0026
tags:
  - Python
---


 缓更。。。
 
 
 排序一向是写程序中不可避免的功能，我在写操作系统Lab1时遇到了需要对一个List进行排序，而且list里保存的是一个包含多个值的类，现在要根据不同值进行排序，一开始我是新建一个字典，然后将要排序的值加上ID作为字典的键，然后对字典的键调用sort()方法
 
 如下：
 
 ```
proceedDic = {}
for item in proceedList:
    proceedDic[str(item.timeArrived)+','+str(item.numArrived)] = item
keyList = [[int(item.split(',')[0]), int(item.split(',')[1])] for item in proceedDic.keys()]
keyList.sort()
NowList = [ proceedDic[str(item[0])+','+str(item[1])] for item in keyList]
 ```
 
 
 说实话，很恶心的一段程序，然后我采用了直接保存为list的方法进行排序：
 
 ```
sortList = [[item.timeArrived,item.numArrived,item] for item in proceedList]
sortList.sort()
NowList = [item[2] for item in sortList]
 ```
 
 看起来稍微好点，但也蛮怪的，等这段繁忙得时间断过去，再好好探究下Python的排序方法。
