---
title:  Google Python Style Guide
layout: post
tags: [Python]
---


翻译自[pyguide](http://google-styleguide.googlecode.com/svn/trunk/pyguide.html)


##背景

Python 是 Google 使用的最主要的脚本语言。这个编码风格指南是在 Python 代码里的一系列`可以`和`不可以`。
为了帮助你写出格式正确的代码，我们制定了一个 Vim 的[设置文件](http://google-styleguide.googlecode.com/svn/trunk/google_python_style.vim)。对于 Emacs 而言，默认设置就不错。

---

##Python 语言规则

###Lint

运行`pylint`来检查你的代码。

- *定义*

  pylint 是一个用来发现 Python 源代码里的 bug 和编码风格问题的工具。它会发现那些在 C 、C++ 那样的非动态语言中由编译器捕捉的问题。由于 Python 的动态特性，有些警告可能是错误的，但这种情况非常罕见。
  
- *优点*

	捕捉那些常见的错误，如拼写错误、使用未被赋值的变量等。
	
- *缺点*

	`pylint` 并非完美。为了更好的使用它，我们可能需要：a)针对它写代码 b)忽视警告 c)改进它
	
- *决策*

	确保在你的代码上运行 `pylint` 。为了避免主要的问题被隐藏，你可以忽略那些不合适的警告。
	
	你可以写下一行注释来忽略特定的警告：
	
		```
		dict = 'something awful' # Bad Idea... pylint: disable = redefined-biltin
		```
	
	每个 pylint 的警告都由一个由字母数字组成的代码（C0112）和一个象征性的名字（empty-docstring)标识。最好在新代码里或者更新旧代码时使用象征名。
	
	如果单从象征名看不出忽略该警告的愿意的话，增加一条解释。
	
	通过这种方式来忽略警告的优点在于我们能轻松的搜索和重新考量它们。
	
	你可以通过命令 `pylint --list-msgs` 来获得 pylint 的警告列表。使用命令 `pylint --help-msg=C6409` 来得到对于一个具体警告的更多信息。
	
	使用 `pylint: disable` 而不是旧格式 `pylint: disable-msg`。
	
	`变量未使用`警告可以通过将该变量命名为 `_` 或者在变量名称前加上 `unused_` 前缀来忽略。对于那些不能更改变量名的情况，你可以在函数开头调用一下他们。例：
	
		```
		def foo(a, unused_b, unused_c, d=None, e=None):
		    _ = d, e
		    return a
		```
		

	
	
###Imports

只使用 `import` 来导入模块和包

- *定义*

	在模块之间共享代码的重用机制
	
- *优点*

	简单的命名空间管理。以一种固定的方式来表明每个标识符的来源。`x.obj`说明对象`obj`定义在模块`x`中
	
- *缺点*

	模块名可能会存在冲突。一些模块的名字长的不可思议。
	
- *决策*

	使用 `import x` 来导入模块和包。
	
	使用 `from x import y` ，此时 `x` 是作为前缀的包，而 `y` 是不需要前缀的模块名。
	
	使用 `from x import y as z` ，如果有两个叫做 `y` 的模块被同时导入了，或者 `y` 实在长的令人无法忍受。
	
	例如：模块 `sound.effects.echo` 可能想下面一样被导入：
	
		```
		from sound.effects import echo
		...
		echo.EchoFilter(input, output, delay = 0.7, atten = 4)
		```
		
###Packages

每个模块都通过它的完整路径来导入

- *优点*

	避免模块名之间的冲突。更容易的找到模块。
	
- *缺点*

	部署代码变得困难了，因为你不得不重现包的分层结构。
	
- *决策*

	所有新代码在导入模块时都应该使用它的完整包名。
	
	想下面的例子一样导入：
	
		```
		# Reference in code with complete name.
		import sound.effects.echo
		
		# Reference in code with just module name (preferred).
		from sound.effects import echo
		```
		
###Exceptions

允许异常处理但必须小心使用。

- *定义*

	异常处理是为了处理错误或者别的异常状况而打破代码块的正常控制流的方法。
	
- *优点*

	正常操作代码的控制流不会被错误处理代码弄乱。在特定情况发生时可以让控制流跳过多个步骤，比如说：从 n 层循环中直接跳出而不是继续执行错误代码。
	
- *缺点*

	可能使控制留变得难以理解。在进行函数库调用时容易忽略一些错误情况。
	
- *决策*

	异常处理必须遵循特定的条件：
	
	- 像这样子抛出异常： `raise MyException('Error message')  或者 `raise MyException`。不要使用双参数的形式 `(raise MyException, "Error message")` 或者已废弃的基于字符串的异常处理 `raise "Error message"`




	

























	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	


	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	
	


