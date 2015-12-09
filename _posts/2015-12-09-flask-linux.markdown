---
title: Ubuntu 下 WSGI + Nginx + Supervisor 部署 Flask
layout: post
tags:
  - Web
  - Linux
---

学生特惠时买了台阿里云三年的服务器，我正好在学 Flask，所以就需要考虑在 Linux 下部署 Flask 的问题，本篇记录了参考各种文章后从零开始进行部署的所有步骤。

## 一、安装 Python 环境

由于 Ubuntu 默认安装了 Python 2.7，所以第一步先开始安装 Python 的pip 工具。

#### PIP

``` shell
apt-get isntall pip
```

#### Virtualenv

Virtualenv可以创建虚拟环境。虚拟环境是 Python 解释器的一个私有副本，在这个环境中安装的私有包，不会影响系统中安装的全局Python解释器。

```
pip install virtualenv
```
我的项目目录为 `/root/Flask`，那么可以在 `Flask` 文件夹内新建一个虚拟环境，并命名为 `venv`

```
cd /root/Flask
virtualenv venv
```
接下来开启虚拟环境
```
source venv/bin/activate
```
##二、安装 Flask
使用 `pip` 在虚拟环境中安装 Flask
```
pip install flask
```
然后在 `/root/Flask` 文件夹中新建一个标注 Flask 运行文件 `test.py` 作为例子：

```
from flask import Flask

app = Flask(__name__)


@app.route("/")
def hello():
    return "Hello World!"
   
if "__name__" == "__main__":
    app.run()
```

##三、安装 uWSGI
直接通过`pip`安装 uWSGI会出错，正确做法为：
```
apt-get install build-essential python-dev
pip install uwsgi
```


##四、配置 uWSGI

为了配置 uWSGI，直接在项目目录（`/root/Flask`)下新建一个 `config.ini` 文件作为配置文件：

```
[uwsgi]

# uwsgi 启动时所使用的地址与端口
socket = 127.0.0.1:8001 

# 指向网站目录
chdir = /root/Flask

# python 启动程序文件
wsgi-file = test.py 

# python 程序内用以启动的 application 变量名
callable = app 

# 处理器数
processes = 4

# 线程数
threads = 2

#状态检测地址
stats = 127.0.0.1:9191
```

由于我们将使用 supervisor 来管理引导 uWSIGI 的启动，所以这里不需要通过命令(`uwsgi config.ini`）来运行。

##五、安装 Supervisor

安装 Supervisor：

```
apt-get install supervisor
```
下面对其进行配置：

只需要新建一个 `.conf` 文件在 `/etc/supervisor/conf.d` 文件夹下即可，在这个例子中，我们新建一个`test_supervisor.conf`：

```
[program:test]
# 启动命令入口
command=/root/Flask/venv/bin/uwsgi /root/Flask/config.ini

# 命令程序所在目录
directory=/root/Flask
#运行命令的用户名
user=root
		
autostart=true
autorestart=true
#日志地址
stdout_logfile=/root/Flask/uwsgi_supervisor.log	
```

启动服务：`service supervisor start`

终止服务：`service supervisor stop`

##六、安装 Nginx

Nginx 是一个反向代理软件。

```
apt-get install nginx
```

配置 Nginx：

将 `/etc/nginx/sites-available/default` 文件删除，替换成新的 `default` 文件即可。

```
	server {
	  listen  80;
	  server_name XXX.XXX.XXX; #公网地址
	
	  location / {
		include      uwsgi_params;
		uwsgi_pass   127.0.0.1:8001;  # 指向uwsgi 所应用的内部地址,所有请求将转发给uwsgi 处理
		uwsgi_param UWSGI_PYHOME /root/Flask/venv; # 指向虚拟环境目录
		uwsgi_param UWSGI_CHDIR  /root/Flask; # 指向网站根目录
		uwsgi_param UWSGI_SCRIPT test:app; # 指定启动程序
	  }
	}
```

然后重启 Nginx：

```
service nginx restart
```

到这一步，就完成了部署 Flask 的所有步骤。

---

参考文章： [阿里云部署 Flask + WSGI + Nginx 详解](http://www.cnblogs.com/Ray-liang/p/4173923.html)