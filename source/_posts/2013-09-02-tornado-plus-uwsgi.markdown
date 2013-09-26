---
layout: post
title: "Tornado+uWSGI"
date: 2013-09-03 00:05
comments: true
categories: tornado,python,uwsgi
---

＃使用Tornado＋uWSGI来处理HTTP请求

一个简单的响应“hello world”的例子如下：

```
#!/usr/bin/env python
#-*-coding:utf-8-*-

import tornado.web
import tornado.wsgi

class handle(tornado.web.RequestHandler):
    def initialize(self):
        self.name = ''

    def get(self):
        self.name = self.get_argument("name", "nobody")
        uid = self.get_cookie("uid", 0) + 1
        self.set_cookie("uid", str(uid))
        self.set_header("New-Header", "Yes")
        self.write("hello %s" % self.name)

    def post(self):
        self.name = self.get_argument("name", "nobody")
        self.set_status(405, "")
        self.write("sorry, %s" % self.name)


application = tornado.wsgi.WSGIApplication([
    (r'/hello', handle),
])
```
<!--more-->
 
例子虽然比较简单，但一些常用的功能都覆盖到了，请求参数的获取、cookie的获取和设置、url map的定义等。获取请求参数的方式跟webpy类似，GET方式和POST方式都是通过一个API来获取，跟flask中的有点区别。

另外需要说明的是，像上面例子中那样定义一个handle类来处理请求的话，必须要继承tornado.web.RequestHandler，并且不能自己在定义\_\_init\_\_函数，如果有初始化工作要做，就定义一个initialize函数，把初始化工作都放到这个函数里，这个函数会在tornado.web.RequestHandler的\_\_init\_\_函数的最后被调用。实际使用起来跟\_\_init\_\_函数没有太大区别。

还有一点跟webpy不一样的是，例子中的handle不能是函数，必须是个继承tornado.web.RequestHandler的类。
