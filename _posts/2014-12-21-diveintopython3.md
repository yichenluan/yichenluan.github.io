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
```

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
	
##Chapter 3.解析

###处理文件和目录

- 当前工作目录

	```
	>>> import os 
	>>> print(os.getcwd()) 			#获取当前工作目录
	C:\Python31
	>>> os.chdir('/Users/pilgrim/diveintopython3/examples') #改变当前工作目录
	>>> print(os.getcwd()) 
	C:\Users\pilgrim\diveintopython3\examples
	```

- 处理文件名和目录名

	```
	>>> import os
	>>> print(os.path.expanduser('~')) 	#用来将包含 ~ 符号的路径扩展为完整的路径
	c:\Users\pilgrim
	#很方便的构造出用户目录下的文件和目录的路径
	>>> print(os.path.join(os.path.expanduser('~'), 'diveintopython3', 'examples', 'humansize.py')) 
	c:\Users\pilgrim\diveintopython3\examples\humansize.py
	```
	
	os.path 也包含用于分割完整路径名，目录名和文件名的函数
	
	```
	>>> pathname = '/Users/pilgrim/diveintopython3/examples/humansize.py'
	>>> os.path.split(pathname) 
	('/Users/pilgrim/diveintopython3/examples', 'humansize.py')
	>>> (dirname, filename) = os.path.split(pathname) 
	>>> dirname 
	'/Users/pilgrim/diveintopython3/examples'
	>>> filename 
	'humansize.py'
	>>> (shortname, extension) = os.path.splitext(filename) 
	>>> shortname
	'humansize'
	>>> extension
	'.py'
	```
	
###列表解析

- 实现通过对列表中每一个元素应用一个函数的方法来将一个列表映射到另一个列表

	```
	>>> a_list = [1, 9, 8, 4]
	>>> [elem * 2 for elem in a_list] 
	[2, 18, 16, 8]
	>>> a_list 
	[1, 9, 8, 4]
	>>> a_list = [elem * 2 for elem in a_list] 
	>>> a_list
	[2, 18, 16, 8]
	```

- 列表解析可以使用任何的 Python 表达式

	```
	>>> import os, glob
	>>> [f for f in glob.glob('*.py') if os.stat(f).st_size > 6000] 
	['pluraltest6.py',
	'romantest10.py',
	'romantest6.py',
	'romantest7.py',
	'romantest8.py',
	'romantest9.py']
	```
	
###字典解析

- 字典解析和列表解析类似，只不过它生成字典而不是列表

	```
	>>> import os, glob
	>>> metadata = [(f, os.stat(f)) for f in glob.glob('*test*.py')] 
	>>> metadata[0] 
	('alphameticstest.py', nt.stat_result(st_mode=33206, st_ino=0, st_dev=0,
	st_nlink=0, st_uid=0, st_gid=0, st_size=2509, st_atime=1247520344,
	st_mtime=1247520344, st_ctime=1247520344))
	>>> metadata_dict = {f:os.stat(f) for f in glob.glob('*test*.py')} 
	>>> type(metadata_dict) 
	<class 'dict'>
	>>> list(metadata_dict.keys()) 
	['romantest8.py', 'pluraltest1.py', 'pluraltest2.py', 'pluraltest5.py',
	'pluraltest6.py', 'romantest7.py', 'romantest10.py', 'romantest4.py',
	'romantest9.py', 'pluraltest3.py', 'romantest1.py', 'romantest2.py',
	'romantest3.py', 'romantest5.py', 'romantest6.py', 'alphameticstest.py',
	'pluraltest4.py']
	>>> metadata_dict['alphameticstest.py'].st_size 
	2509
	```

- 小技巧：交换字典的键和值

	```
	>>> a_dict = {'a': 1, 'b': 2, 'c': 3}
	>>> {value:key for key, value in a_dict.items()}
	{1: 'a', 2: 'b', 3: 'c'}
	```
	
###集合解析

```
>>> a_set = set(range(10))
>>> a_set
{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
>>> {x ** 2 for x in a_set} 
{0, 1, 4, 81, 64, 9, 16, 49, 25, 36}
>>> {x for x in a_set if x % 2 == 0} 
{0, 8, 2, 4, 6}
>>> {2**x for x in range(10)} 
{32, 1, 2, 4, 8, 64, 128, 256, 16, 512}
```


##Chapter 4.字符串

###格式化字符串

-  `'{0:.1f} {1}'.format(size, suffix)` 发生了什么？
-  Python 3 支持把值格式化成字符串，最基本的用法是使用单个占位符将一个值插入字符串

	```
	>>> username = 'mark'
	>>> password = 'PapayaWhip'                             
	>>> "{0}'s password is {1}".format(username, password)  
	"mark's password is PapayaWhip"
	```
	
- 复合字段名

	```
	>>> import humansize
	>>> si_suffixes = humansize.SUFFIXES[1000]      
	>>> si_suffixes
	['KB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB']
	>>> '1000{0[0]} = 1{0[1]}'.format(si_suffixes)  
	'1000KB = 1MB'
	```
	
	可以通过利用 Python 的语法访问到对象的元素或属性，这就叫复合字段名。
	
- 格式说明符

	```
	'{0:.1f} {1}'.format(size, suffix)
	```
	
	在替换域中，冒号(:)标示格式说明符的开始。“.1”的意思是四舍五入到保留一们小数点。“f”的意思是定点数
	
###其他常用字符串方法

```
>>> query = 'user=pilgrim&database=master&password=PapayaWhip'
>>> a_list = query.split('&')                            
>>> a_list
['user=pilgrim', 'database=master', 'password=PapayaWhip']
>>> a_list_of_lists = [v.split('=', 1) for v in a_list]  
>>> a_list_of_lists
[['user', 'pilgrim'], ['database', 'master'], ['password', 'PapayaWhip']]
>>> a_dict = dict(a_list_of_lists)                       
>>> a_dict
{'password': 'PapayaWhip', 'user': 'pilgrim', 'database': 'master'}
```


###Python 源码的编码方式

在 Python 2 里，.py 文件默认的编码方式为 ASCII，Python 3 的源码的默认编码方式为 UTF-8

如果想使用不同的编码方式来保存 Python 代码，我们可以在每个文件的第一行放置编码声明（encoding declaration）：

`# -*- coding: windows-1252 -*-`

##Chapter 5.正则表达式

见 [正则表达式](http://jinke.me/2013/11/29/regex.html)

##Chapter 6.闭合 与 生成器