---
title:  Github 快速配置指南
layout: post
tags: [Github]
---

因为我时不时就重装下系统，无论是 Mac OS X 还是 Ubuntu，而这两个系统又都是我使用 Github 学习的环境。每次重装完后总得去上网 Google Github的配置流程。这不，刚刚才设置好 Github，这次干脆就记录下来。

##SSH 配置

要让本地与 Github 通信，需要先配置好 SSH key.
	 
1. 检查是否已有 SSH key

	```
	cd  ~/.ssh
	```
	如果出现`No such file or directory` ，说明本地缺少 SSH key

2. 创建新的 SSH key

	```
	ssh-keygen -t rsa -C "email@email.con"
	```
	这时会出现保存路径提示和之后的密码提示，全部直接回车默认即可。
	
3. 将 SSH key 添加到你的 Github 上

	首先到上一步的默认保存文件夹内找到 id_rsa.pub 文件，打开后，复制里面的所有内容。
	
	然后进入 Github 中个人设置里面的 `SSH Public Keys` ，添加 Key。

4. 测试是否添加成功

	```
	ssh -T git@github
	```
	如果出现`Hi yichenluan! You've successfully authenticated, but GitHub does not provide shell access.`就说明 OK 了

	
SSH 配置结束

##与远程版本库通信

1. 配置 git 的用户名和 email

	```
	git config --global user.name "yourname"
	git config --global user.email "youremail"
	```
	
2. 克隆一个远程版本库

	```
	git clone git@github.com:yichenluan/yichenluan.github.io.git
	```
	
之后就可以正常对版本库进行操作了。
