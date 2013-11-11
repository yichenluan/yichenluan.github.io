---
title: 托管在Github上的个人主页
layout: post
tags:
  - Github
---

折腾了快有2个小时，总算是把这个网站搭建成功了。期间参考了很多博客的内容，这里就不一一列出了，下面记录下搭建步骤。

- 在Github上新建repo，并命名为`yichenluan.github.io`
- 安装jekyll：
	```
	gem install jekyll
	```

	但是在第一次安装jekyll的时候遇到了一些问题，装完之后，执行`jekyll -v`命令，会报错。

	捣鼓了半天，期间安装了 `Ruby1.9.1`和`Ryby1.9.1-dev`，再重新执行`gem install jekyll`就成功了。

- 新建博客文件夹
	
	```
	jekyll new myblog
	```

	这样子，jekyll就自动把一个最基础的博客所需要的文件全部自动生成到这个文件夹内了

- 预览网站
	
	```
	cd myblog
	jekyll serve
	```

	在浏览器里输入`http://localhost:4000`即可看到刚才新生成的网站

- 托管到Github

	直接把myblog文件夹内的所有文件复制到本地的仓库内，然后执行：

	```
	git add .
	git commit -m ""
	git push origin master
	```
	到这一步，就可以通过`yichenluan.github.io`访问个人主页了

- 更新日志
	
	如需更新日志，直接在本地仓库内的_post文件夹内新建类似于`2013-11-07-name.markdown`文件并编辑，再push到Github上即可。


---
#END

