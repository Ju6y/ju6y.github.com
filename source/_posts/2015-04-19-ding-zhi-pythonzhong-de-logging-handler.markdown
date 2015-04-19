---
layout: post
title: "定制Python中的logging handler"
date: 2015-04-19 17:04
comments: true
categories: python, logging
---

作为一个开发，写程序的时候应该都很喜欢打日志，一个是方便调试和定位问题，还有一个是可以将一些关键数据打印出来，方便业务上的统计和监控。但是在实际工作中，往往是不能随心所欲的打日志的，因为我们都知道磁盘io是很慢的，而生成环境的服务器一般都是机械硬盘（用SSD的可以忽略，但SSD的一般空间不会太大，毕竟价格昂贵），即使有raid也架不住程序开到DEBUG级别狂刷日志，大量的CPU都用在等待io上了，很影响性能。

<!--more-->

但是呢，日志全关也不行，因为总会有些数据是需要日志来获取的，或者需要通过打印一些关键数据来监控程序的运行情况。当然，有人会说，要运行时的数据不一定要用日志啊，可以搞个监控服务器，定时向监控服务器推数据嘛。采用这种方式当然能解决需求，但是每增加一项统计监控项都是需要带来一些开发工作量的，灵活性上要比打日志的方式差。
那么，是否有更好的方式呢，当然有，那就是将日志打到别的机器上（就好比青春痘长在别人脸上不让你担心。。）既然走网络，那就得挑个协议，最常用的TCP显然不是最优选择，我们知道TCP是个可靠性高的协议，有很多机制来保证数据传输的可靠性，又是面向有连接的，实际用时可能为了性能还需要维持长连接，总之TCP的优点在打日志上显得不是那么重要。既然不打算用TCP，那就用UDP吧，UDP面向无连接，又没有确认机制，对日志来说丢个1\~2条完全是可接受的。（对日志可靠性要求高的可以用TCP，但本文不会涉及用TCP来打远端日志，但可借鉴本文，实现没有太大区别）
好了，既然明确了需要什么样东西，那就先找找有没有现成的。看了下Python [logging][1]库自带的几个handler实现，倒是也有个用UDP的，叫[DatagramHandler][2]，但是这个handler发送的数据内容是个[LogRecord][3]对象，如果你打算在远端要python来接收数据，那就可以直接用这个handler就行了，也就没后面什么事情了。但我是希望能直接发送string形式的数据，这样远端只要任何能够监控对应udp端口的程序都能做为接收端。
先上代码
	class UdpHandler(DatagramHandler):
	    """
	    Send log which is already formatted to the recieve end in string format.
	    """
	    def emit(self, record):
	        try:
	            log_line = "%s\n" % self.format(record)
	            self.send(log_line)
	        except (KeyboardInterrupt, SystemExit):
	            raise
	        except:
	            self.handleError(record)

我在这里定义了一个类UdpHandler，继承DatagramHandler，另外定义了一个emit方法，其他的方法都可以复用DatagramHandler的。
我来解释下这个emit方法，所有handler类都是在emit这个方法中做实际的“写日志”操作，在执行emit这个方法前已经进行了日志级别的过滤，但是参数record中的实际日志内容只有我们应用程序调用时传入的一个字符串，并没有按配置的那样经过格式化，所以我这里会先调用self.format格式话日志内容，然后加个换行符，然后调用send方法发送，这个send方法是DatagramHandler中定义的，内容比较简单就是调用socket的sendto方法，具体可以看DatagramHandler的源代码。

handler定义好以后是没法直接用的，即没法在配置文件中指定。例如，ini形式的配置文件中指定handler是个Udphandler
	class = UdpHandler
这样程序会报错，说找不到UdpHandler。
其实，这个class的解析过程是这样的，class的值必须是在logging的变量字典的key中，并且对应一个handler类。
源代码是这样的
	klass = eval(klass, vars(logging)) 
会对class的值进行一个eval操作，将handler名字符串转成handler对象，而查找范围是logging模块下可见的所有对象。所以这个UdpHandler类必须是在logging下可见，那就好办了，在UdpHandler定义完再加一行
	vars(logging)[UdpHandler.__name__] = UdpHandler

这样就可以正常在配置文件中使用了。当然，别忘了指定ip和端口
	args=(‘127.0.0.1’, 12345)


[1]:	https://docs.python.org/2/library/logging.html
[2]:	https://docs.python.org/2/library/logging.handlers.html?highlight=datagramhandler#logging.handlers.DatagramHandler
[3]:	https://docs.python.org/2/library/logging.html#logging.LogRecord