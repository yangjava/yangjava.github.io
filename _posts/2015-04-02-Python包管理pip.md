---
layout: post
categories: [Python]
description: none
keywords: Python
---
# Python包管理pip

## pip实战
pip是一个用来安装和管理Python包的工具，是easy_install的替代品，如果读者使用的是Python2.7.9+或Python3.4+版本的Python，则已经内置了pip，无须安装直接使用即可。

pip之所以能够成为最流行的包管理工具，并不是因为它被Python官方作为默认的包管理器，而是因为它自身的诸多优点。pip的优点有：
- pip提供了丰富的功能，其竞争对手easy_install则只支持安装，没有提供卸载和显示已安装列表的功能；
- pip能够很好地支持虚拟环境；
- pip可以通过requirements.txt集中管理依赖；
- pip能够处理二进制格式（.whl）；
- pip是先下载后安装，如果安装失败，也会清理干净，不会留下一个中间状态。

下面以Flask为例，来看一下pip几个常用的子命令。
- 查找安装包：
  `pip search flask`

- 安装特定的安装包版本：
  `pip install flask==0.8`

- 删除安装包：
  `pip uninstall Werkzeug`

- 查看安装包的信息：
```shell
$ pip show flask
Name: Flask
Version: 0.12
Summary: A microframework based on Werkzeug, Jinja2 and good intentions
Home-page: http://github.com/pallets/flask/
Author: Armin Ronacher
Author-email: armin.ronacher@active-4.com
License: BSD
Location: /home/lmx/.pyenv/versions/2.7.13/lib/python2.7/site-packages
Requires: click, Werkzeug, Jinja2, itsdangerous
```

-检查安装包的依赖是否完整：
```shell
$ pip check flask
Flask 0.12 requires Werkzeug, which is not installed.
```

- 查看已安装的安装包列表：
  `pip list`
- 导出系统已安装的安装包列表到requirements文件：
  `pip freeze > requirements.txt`

- 从requirements文件安装：
  `pip install -r requirements.txt`

- 使用pip命令补全：
```shell
pip completion --bash >> ～/.profile
$ source ～/.profile
```

### 加速pip安装的技巧
如果大家使用Python的时间比较长的话，会发现Python安装的一个问题，即pypi.python.org不是特别稳定，有时候会很慢，甚至处于完全不可用的状态。这个问题有什么好办法可以解决呢？
- 使用豆瓣或阿里云的源加速软件安装
  访问pypi.python.org不稳定的主要原因是因为网络不稳定，如果我们从网络稳定的服务器下载安装包，问题就迎刃而解了。我们国内目前有多个pypi镜像，推荐使用豆瓣的镜像源或阿里的镜像源。如果要使用第三方的源，只需要在安装时，通过pip命令的-i选项指定镜像源即可。如下所示：
```shell
pip install -i https://pypi.douban.com/simple/ flask
```