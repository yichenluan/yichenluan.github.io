---
title: Python 单元测试
layout: post
tags:
  - Python
---

Python 自带一个单元测试框架，被恰当地命名为 unittest 模块。

在这里，假设一个方法名为 to_roman() ，这个方法的作用是将阿拉伯数字转换为罗马数字。

```
import roman1
import unittest

class KnownValues(unittest.TestCase):               
    known_values = ( (1, 'I'),
                     (2, 'II'),
						  ...
                     (3999, 'MMMCMXCIX'))           

    def test_to_roman_known_values(self):           
        '''to_roman should give known result with known input'''
        for integer, numeral in self.known_values:
            result = roman1.to_roman(integer)       
            self.assertEqual(numeral, result)       

if __name__ == '__main__':
    unittest.main()
```

1. 为编写测试用例，首先使测试用例类成为 unittest 模块 TestCase 类的子类。
2. TestCase 类提供了 assertEqual 来检查两个值是否相等。
3. 测试方法必须以 test 开头。

下一步需要测试输入值是否在允许的范围内。

```
class ToRomanBadInput(unittest.TestCase):
    def test_too_large(self):
        '''to_roman should fail with large input'''
        self.assertRaises(roman3.OutOfRangeError, roman3.to_roman, 4000)  

    def test_zero(self):
        '''to_roman should fail with 0 input'''
        self.assertRaises(roman3.OutOfRangeError, roman3.to_roman, 0)     

    def test_negative(self):
        '''to_roman should fail with negative input'''
        self.assertRaises(roman3.OutOfRangeError, roman3.to_roman, -1)    
```

1. unittest.TestCaes 类提供 assertRaises 方法，该方法需要以下参数：你期望的异常、你要测试的方法及传入给方法的参数
2. 这里需要注意，为了解决问题，你应该在 roman.py 中定义提到的异常

```
class OutOfRangeError(ValueError):  
    pass                            
```






















