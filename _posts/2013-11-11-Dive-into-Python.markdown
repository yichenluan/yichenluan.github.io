---
title: 《Dive into Python》笔记
layout: post
guid: urn:uuid:6a0f8df5-6f92-4a07-996c-4061e23d0026
tags:
  - Python
---

我学python正式的入门教材是`Head First Python`，其间掺杂着看了一下老鼠书，但是老鼠书实在太过繁琐，最终还是没坚持看下去。这学期数字图像处理这门课平时作业我选择采用python来做，在强大的PIL库的支持下，做的还是顺风顺水的，不过很多时候，会发现自己连一些基本的语法都记不清楚了，所以，我计划通过每天看一章`Dive into Python`来重新过一遍语法。但哪知道，光第二章就给了我很多惊喜，解决了我的一些疑问，下面是基本笔记：

- ##Chapter 2
	- 文档化函数
		
		- 通过给出一个doc string（文档字符串）来文档化一个Python函数
		- 实例：

		```
		def function(something):
			"""This is a function.
	
			Return something."""
		```
		
		- 格式：由一对三重引号包围。第一行为概述，第二行为空行，第三行时详细描述
		- 事实上在任何时候都应该使用 doc string

	- 万物皆对象
		
		- 对Python而言，万物皆对象，列表是对象，函数是对象，甚至模块也是对象
		- 模块导入搜索路径
			
		```
		import sys
		sys.path
		sys.path.append('/my/new/path')
		```

	- 测试模块
		`if __name__=="__main__":`这是一个测试模块的技巧
		- 模块时对象，且这个对象有一个内置属性__name__
		- 如果import一个模块，则，__name__的值通常时模块的文件名
		- 如果直接运行一个模块，那么，__name__将是__main__
		- 实例代码：

		```
		import odbchelper
		odbchelper.__name__
			'odbchelper'
		```


- ##Chapter 3
	- 一次赋多值

    	```
		>>>v=('a','b','c')
		>>>(x,y,z)=v
		>>>x
		'a'
		>>>y
		'b'
		>>>z
		'c'
		```

		```
		>>>(mon,tues,wedens,thurs,fri,satur,sun)=range(7)
		```
    - 字符串的join方法

		```
		return ";".join(["%s=%s"% (k,v) for k,v in params.items()])
		```
	join方法把list中的元素连接成单个字符床，每个元素用分号隔开
	
- ##Chapter 4
    - 先给出第4章程序：
        
        ```
        def info (object, spacing =10, collapse = 1):
            methodList = [method for medthod in dir(object) if callable ( object, method))]
            processFunc = collapse and (lambda s: " ".join(s.split())) or (lambda s:s)
            print "\n".join["%s %s"%
                (method.ljust(spacing),
                processFunc(str(getattr(object,method).__doc__)))
                 for method in methodList]
        if __name__ == "__main__":
            print info.__doc__
        ```

    - 使用可选参数和命名函数
    
        ```
        def info (object,spacing=10,collapse=1):
        
        info(odbchelper)
        info(odbchelper,12)
        info(odbchelper,collapse=0)
        info(spacing=15,object=odbchelper)
        ```
        要意识到参数不过是个字典
        
    - 通过`getattr`来获取对象引用
        使用`getattr`，可以得到一个直到运行时才知道名称的函数的引用
        
        最常见的例子是通过它来创建分发者
        
        首先假设在`statsout`模块定义了三个输出函数：`output_html`,`output_xml`,`output_text`
        
        那么定义唯一的输出函数如下：
        ```
        import statsout
        
        def output(data,format='text'):
            output_function = getattr(statsout,'output_%s'%format)
            return output_function(data)
        ```
        
    - 列表的过滤
    
        例子：
        ```
        li = ['a','mpilgrim','foo','b','c','d']
        [elem for elem in li if len(elem)>1]
        ```
        
    - and 和 or的特殊用法
        - `1 and a or b`相当于 `1 ? a:b`;`0 and a or b`相当于`0 ? a:b `
        - 但时单`a` 为空时，and-or技巧失效
        - 解决方法：
            ```
            a = ''
            b = 'second'
            (1 and [a] or [b])[0]
            ```
        注意[a]是个非空列表
        
    - 使用lambda函数
        ```
        g = lambda x: x*2
        processFunc = collapse and (lambda s : ' '.join(s.spilt())) or (lambda s:s)
        ```
