---
layout: post
title: "virtualenv"
date: 2013-09-03 23:04
comments: true
categories: 
- virtualenv
- python
---

如果一个生产环境中部署了多个项目，每个项目使用不同版本的Python，或者不同项目依赖同一个模块的不同版本；

如果你有洁癖，希望保证服务器默认环境的干净；

如果你想尝试某些新工具，又不希望因为这些工具安装失败而搞乱整个服务器环境

如果你会遇到如上几种情况，那么virtualenv无疑是你最好的选择。

### 安装

	$ sudo easy_install virtualenv

或者

	$ sudo pip install virtualenv

<!--more-->

### 建立虚拟环境

	$ mkdir myproject
	$ cd myproject
	$ virtualenv venv
	New python executable in venv/bin/python
	Installing distribute............done.

### 使用虚拟环境

	$ . venv/bin/activate

如果以uwsgi的方式启动，那么在uwsgi的配置中增加如下配置（ini格式，其他形式请查[官方文档][1]）：

	virtualenv = myproject的全路径/venv

[1]:	http://uwsgi-docs.readthedocs.org/en/latest/Options.html#home-virtualenv-venv-pyhome