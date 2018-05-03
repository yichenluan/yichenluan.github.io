---
title: VPS + Shadowsocks 同时监听 IPv4 和 IPv6
layout: post
tags: [Web, Linux]
---

前些日子逛 V2EX 看到有人在讨论 Github Student Developer Pack，其中赠送了 DigitalOcean 的 50 刀劵。想起自己在一年多钱就成功申请了 Github 的学生开发包，而且自己目前在用的多态代理在 OS X 下时灵时不灵的，于是就买了个 DO 最便宜，一个月 5 刀的 VPS。

北理工的校园网 IPv4 按流量收费，10G 十元，IPv6 不限流量，所以，就要在 VPS 上搭建同时监听 IPv4 和 IPv6 的 Shadowsocks。这样就能在有 IPv6 的情况下，免流量免登陆校园网络实现翻墙上网。具体步骤如下：

###准备 VPS

我选的是 DigitalOcean 最便宜的那一款 VPS，每个月 5 刀，支持 IPv6，每月 1TB 流量。经过测速后，选择了 San Francisco Region，系统选择了 Ubuntu 14.04 x64。

创建 Droplet 后，系统会发一封包含 root 密码的邮件到邮箱中，第一次 ssh 到 VPS 上会要求修改 root 的密码。

###安装 Shadowsocks

首先更新软件源：

```
apt-get update 
```

然后安装 Shadowsocks：

```
apt-get install python-pip

pip install shadowsocks
```
Shadowsocks 目前已经停止更新，Github页面也停留在 `Removed according to regulations`。Shadowsocks不能说改变了世界，但至少让我们觉得这个世界变得更美好了些。有理想有情怀的软件工程师不外如是。

###配置 Shadowsocks

配置 Shadowsocks 非常简洁，新建一个 `config.json` 文件，在文件中写入：

```
# 注释版配置
{
    "server":"servier_ip",   # 服务器IP
    "server_port":65432,     # ss服务器所使用的端口号，建议改到30000-60000
    "password":"password",   # ss服务器密码，轻易不要分享
    "timeout":60,            # 超时时间，建议设置为60
    "method":"rc4-md5"         # 加密方式，需要和客户端配合设置
}
```

写好配置文件后就可以启动了：

```
ssserver -c your-path-to-config.json  -d start
```

停止命令：

```
ssserver -d stop
```

###同时监听 IPv4 和 IPv6

到目前为止只要在客户端上下载 Shadowsocks 客户端，按服务器上的配置填入相关内容，就完成了整个的配置环节。但是，我需要的是同时监听 IPv4 和 IPv6。

网上说的最多的几种方法在我这都行不通，第一种是把 config 中的 server 改成如下：

```
"server" : "::",
```
第二种这么改：

```
"server":["[::0]", "0.0.0.0"],
```

第三种这么改：

```
"server":["your-IPv4-address","your-IPv6-address"],
```

这三种方式要么报错，要么只能监听 IPv6 地址，总之不能正常工作，最后正确方法是，建立两个配置文件，并分别启动 Shadowsocks 分配给不同进程，实现同时监听。具体来说：

- 新建 config4.json 和 config6.json，内容为：

	```
	config4.json

	{
   		"server":"your-IPv4-address"
    	"server_port":11111,
    	"password":"ipv4-pwd",
    	"timeout":60,
    	"method":"rc4-md5"
	}
	
	config6.json

	{
   		"server":"your-IPv6-address"
    	"server_port":22222,
    	"password":"ipv6-pwd",
    	"timeout":60,
    	"method":"rc4-md5"
	}
	
	```
这里要注意的是，两个配置文件的端口不能一样。

- 分别启动

	```
	ssserver -c your-path-to-config4.json -d start --pid-file ss1.pid
	ssserver -c your-path-to-config6.json -d start --pid-file ss2.pid
	```
这样就完成了 Shadowsocks 的配置，并使其同时监听 IPv4 和 IPv6。

###配置客户端

配置客户端很简单，去[官网](https://shadowsocks.org)上下载好客户端，安装在服务端填写的内容输入到客户端上，点击连接，即可跨越 GFW

###使用 Proxifier 实现全局代理

参考[教程](https://aiguge.xyz/proxifier/)，经过简单的设置，就可以让诸如 Steam 之类的软件走代理。

注意，Steam 需要在快捷方式的属性中，加入 `-tcp`，才可以正常访问。

经实测，Steam 选 SF 附近的下载节点，使用 IPv6 免学校计费流量登陆，速度可以稳定在 9Mb/s 以上，DO 是 1G 带宽，所以这个速度应该是宿舍方面的限制。


---

参考文章：

[用Shadowsocks配置Proxifier代理玩Steam游戏图文教程](https://aiguge.xyz/proxifier/)

[Digital Ocean + Shadowsocks + Ipv6，免流量科学上网](http://blog.kyangcis.me/2015/01/01/my-first-vps/)

[使用shadowsocks轻松搭建翻墙环境教程](https://blog.phpgao.com/shadowsocks_on_linux.html)


















