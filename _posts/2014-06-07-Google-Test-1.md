---
title: Google Test 学习笔记 「编译安装及使用」
layout: post
tags: [Program]
---

Google Test 是 Google 开发使用的 C++ 单元测试框架，使用这个框架的著名项目包括「Chromium projects」和「LLVM 编译器」，下面的安装及使用教程针对于命令行操作，不考虑 VS 或 Xcode 项目。

1. ###获取 gtest

	目前最新的 gtest 版本是 1.7.0 。你可以在[这里](https://code.google.com/p/googletest/downloads/list)下载到最新的 gtest 源码。

2. ###编译安装

	- 首先进入 gtest 目录
	
	- 然后在终端内执行如下两个命令
	
		```
		g++ -I./include -I./ -c ./src/gtest-all.cc  
		ar -rv libgtest.a gtest-all.o  
		```
		
		之后，就在 gtest 目录下生成了 libgtest.a 文件。而这个文件加上 include 文件夹就构成了我们之后写自己的单元测试所需要的一切。
		
	- 要验证编译成功，执行如下命令
	
		```
		cd make
		make
		./sample1_unittest  
		```
		
		如果看到如下输出结果，就说明编译成功了。
		
		```
		Running main() from gtest_main.cc
		[==========] Running 6 tests from 2 test cases.
		[----------] Global test environment set-up.
		[----------] 3 tests from FactorialTest
		[ RUN      ] FactorialTest.Negative
		[       OK ] FactorialTest.Negative (0 ms)
		[ RUN      ] FactorialTest.Zero
		[       OK ] FactorialTest.Zero (0 ms)
		[ RUN      ] FactorialTest.Positive
		[       OK ] FactorialTest.Positive (0 ms)
		[----------] 3 tests from FactorialTest (0 ms total)

		[----------] 3 tests from IsPrimeTest
		[ RUN      ] IsPrimeTest.Negative
		[       OK ] IsPrimeTest.Negative (0 ms)
		[ RUN      ] IsPrimeTest.Trivial
		[       OK ] IsPrimeTest.Trivial (0 ms)
		[ RUN      ] IsPrimeTest.Positive
		[       OK ] IsPrimeTest.Positive (0 ms)
		[----------] 3 tests from IsPrimeTest (0 ms total)

		[----------] Global test environment tear-down
		[==========] 6 tests from 2 test cases ran. (0 ms total)
		[  PASSED  ] 6 tests.
		```
		
3. ###使用示例

	- 我们首先写一个待测函数
	
		```
		int Foo(int a, int b) {
			if(a == 0 || b == 0) {
				throw "Don't do that.";
			}
			int c = a % b;
			if(c == 0)
				return b;
			return Foo(b, c);
		}
		```
		这个函数的作用是求两个数的最大公约数。
		
	- 下面我们来编写一个简单的测试案例
	
		```
		#include "gtest/gtest.h"
		
		TEST(FooTest, HandleNoneZeroInput){
   			EXPECT_EQ(2, Foo(4, 10));
    		EXPECT_EQ(6, Foo(30, 18));
		}
		```
		可以看到，对于检查点的检查，我们用到了 EXPECT_EQ 宏，这个宏用来比较两个数字是否相等。
		
	- 最后，我们加上 main 函数
	
		```
		int main(int argc, char** argv){
    		testing::InitGoogleTest(&argc, argv);
    		return RUN_ALL_TESTS();
		}
		```
		
	- 使用 g++ 编译运行示例文件
	
		我们把上述代码文件命名为 testOfGtest.cpp 。然后我们把之前提到的 libgtest.a 和 include 文件夹复制到 cpp 文件的根目录下。
		
		在命令行里执行如下命令：
		
		```
		g++ -I./include testOfGtest.cpp libgtest.a -o mytest
		./mytest
		```
		
		就可以看到单元测试结果了：
		
		```
		[==========] Running 1 test from 1 test case.
		[----------] Global test environment set-up.
		[----------] 1 test from FooTest
		[ RUN      ] FooTest.HandleNoneZeroInput
		[       OK ] FooTest.HandleNoneZeroInput (0 ms)
		[----------] 1 test from FooTest (0 ms total)

		[----------] Global test environment tear-down
		[==========] 1 test from 1 test case ran. (0 ms total)
		[  PASSED  ] 1 test.
		```
		
		
---
参考：

[初识gtest](http://www.cnblogs.com/coderzh/archive/2009/03/31/1426758.html)

[如何用googletest写单元测试](http://blog.csdn.net/russell_tao/article/details/7333226)

[gtest 安装](http://www.cnblogs.com/coser/p/3212614.html)		



	
