---
title: 《Dive into Python 3》笔记
layout: post
tags:
  - Python
---




##Chapter 1.你的第一个 Python 程序

```
SUFFIXES = {1000: ['KB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB'],
            1024: ['KiB', 'MiB', 'GiB', 'TiB', 'PiB', 'EiB', 'ZiB', 'YiB']}

def approximate_size(size, a_kilobyte_is_1024_bytes=True):
    '''Convert a file size to human-readable form.

    Keyword arguments:
    size -- file size in bytes
    a_kilobyte_is_1024_bytes -- if True (default), use multiples of 1024
                                if False, use multiples of 1000

    Returns: string

    '''
    if size < 0:
        raise ValueError('number must be non-negative')

    multiple = 1024 if a_kilobyte_is_1024_bytes else 1000
    for suffix in SUFFIXES[multiple]:
        size /= multiple
        if size < multiple:
            return '{0:.1f} {1}'.format(size, suffix)

    raise ValueError('number too large')

if __name__ == '__main__':
    print(approximate_size(1000000000000, False))
    print(approximate_size(1000000000000))
````

##Chapter 2.内置数据类型

在 Python 中一切均为对象。

###布尔类型

- True
- False 

###数值类型

- Python 通过是否有 小数点 来分辨 Integer 和 Floating Point
- 使用 type() 函数来检测任何值和变量的类型
- 使用 isinstance() 函数来判断某个值或变量是否为给定某个类型
- 整数可以任意大。（在 Python 2 中，整形分为 int 和 long）
- 浮点数精确到小数点后 15 位
- 分数
		
		>>> import fractions              
		>>> x = fractions.Fraction(1, 3)  
		>>> x
		Fraction(1, 3)
		>>> x * 2                         
		Fraction(2, 3)
		>>> fractions.Fraction(6, 4)      
		Fraction(3, 2)
		>>> fractions.Fraction(0, 0)      
		Traceback (most recent call last):
  		File "<stdin>", line 1, in <module>
  		File "fractions.py", line 96, in __new__
   		 raise ZeroDivisionError('Fraction(%s, 0)' % numerator)
		ZeroDivisionError: Fraction(0, 0)
	
	
###列表

- 创建列表：列表内的元素可以不同
- 列表切片： `a_list[1:-1]  `
- 向列表中新增项

	```
	>>> a_list = ['a']
	>>> a_list = a_list + [2.0, 3]    
	>>> a_list                        
	['a', 2.0, 3]
	>>> a_list.append(True)           
	>>> a_list
	['a', 2.0, 3, True]
	>>> a_list.extend(['four', 'Ω'])  
	>>> a_list
	['a', 2.0, 3, True, 'four', 'Ω']
	>>> a_list.insert(0, 'Ω')         
	>>> a_list
	['Ω', 'a', 2.0, 3, True, 'four', 'Ω']
	```

- 在列表中检索值

	```
	>>> a_list = ['a', 'b', 'new', 'mpilgrim', 'new']
	>>> a_list.count('new')       
	2
	>>> 'new' in a_list           
	True
	>>> 'c' in a_list
	False
	>>> a_list.index('mpilgrim')  
	3
	>>> a_list.index('new')       
	2
	>>> a_list.index('c')         
	Traceback (innermost last):
	  File "<interactive input>", line 1, in ?ValueError: list.index(x): x not in list
	```
	
- 从列表中删除元素

	```
	a_list = ['a', 'b', 'new', 'mpilgrim', 'new']
	del a_list[1]
	a_list.remove('new') 	#删除该值第一次出现
	a_list.pop()				#删除列表中最后的元素，并返回删除的值
	```
	
###元组

- 元组 是不可变的列表
- 使用元祖的好处：
	1. 元组的数独比列表更快
	2. 对不需要改变的数据进行“写保护”将使得代码更加安全
	3. 元组可用作字典键，而列表永远不能当做字典键使用

- 同时赋多值

	```
	v = ('a', 2, True)
	(x, y, z = v
	
	(MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY) = range(7)
	```
	
###集合

- 创建集合 `a_set = { 1, 'a'}` 和 `a_set = set(a_list)`
- 修改集合

	```
	a_set = {1, 2}
	a_set.add(4)
	a_set.update({2, 4, 6}, { 1, 3 5})
	```

- 从集合中删除元素

	```
	a_set.discart(10) 		#删除不存在的值是个空操作
	a_set.remove(10)			#删除不存在的值产生错误
	a_set.pop()				#随机删除某个值，并返回该值
	a_set.clear()				#清空集合，等价于 a_set = set()
	```
	
- 常见集合操作

	```
	>>> a_set = {2, 4, 5, 9, 12, 21, 30, 51, 76, 127, 195}
	>>> 30 in a_set                                                     
	True
	>>> 31 in a_set
	False
	>>> b_set = {1, 2, 3, 5, 6, 8, 9, 12, 15, 17, 18, 21}
	>>> a_set.union(b_set)               #并                               
	{1, 2, 195, 4, 5, 6, 8, 12, 76, 15, 17, 18, 3, 21, 30, 51, 9, 127}
	>>> a_set.intersection(b_set)           #交                            
	{9, 2, 12, 5, 21}
	>>> a_set.difference(b_set)             #补                            
	{195, 4, 76, 51, 30, 127}
	>>> a_set.symmetric_difference(b_set)          #只在其中一个集合出现的元素                     
	{1, 3, 4, 6, 8, 76, 15, 17, 18, 195, 127, 30, 51}
	>>> a_set = {1, 2, 3}
	>>> b_set = {1, 2, 3, 4}
	>>> a_set.issubset(b_set)    #a_set 是 b_set 的子集
	True
	>>> b_set.issuperset(a_set)  
	True
	```
	
