---
title:  为什么Python 中没有switch？
layout: post
tags: [Python]
---

今天在用做操作系统Lab时，碰到需要使用类似C中switch语句功能的情况，但是Python 是没有 Switch 或类似语句的，这是为什么呢？

因为Python 中有比 Switch 更优美的处理方式，那就是字典。

举例而言：

```
#    1 : FCFS,   First Come First Service
#    2 : SJF,     Short Job First
#    3 : SRTF,   Short Remain Time First
#    4 : RR,      Round Robin
#    5 : DPS,    Dynamic Priority
dispatchAction = {
		1 : FCFS,   
    	2 : SJF,      
    	3 : SRTF,   
    	4 : RR,     
    	5 : DPS,   
    	}

dispatch = int( raw_input() )
	
dispatchAction.get(dispatch)(proceedList)
```

---
参考：[link](http://blog.chinaunix.net/uid-7599126-id-331544.html)
