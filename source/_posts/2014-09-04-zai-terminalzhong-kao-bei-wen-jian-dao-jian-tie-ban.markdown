---
layout: post
title: "在Terminal中拷贝文件到剪贴板"
date: 2014-09-04 12:36
comments: true
categories: 
- Mac
- Terminal
- Workflow
---

工作中经常会在终端中用shell算一些数据，将结果保存到文件中，然后需要把文件发给同事（用email或者RTX）。这时会发现需要你打开Finder，然后进到刚才计算的目录下（往往这个目录在Finder下访问会比较繁琐）然后拷贝文件，或者先将结果文件用cp命令拷贝到某个常用目录比如Downloads，再打开Finder到Downloads下拷贝文件。
你会发现，要完成上面的一整套操作就必须在Terminal和Finder之间进行一次切换（这还不包括复制完文件后再到Mail或者RTX的一次切换），这个切换很影响操作流畅度，即会打断工作流（workflow），毕竟一个是纯命令行的交互，一个是GUI的交互。
其实为了去掉那一次切换也是很简单的，就是在终端中完成将文件拷贝到剪贴板（clipboard）就行了，于是google了一下，得出了下面的shell命令：

	ftc() {
		dir=$(pwd)
		osascript -e 'tell app "Finder" to set the clipboard to ( POSIX file "'$dir/$1'" )'
	}

只要把上面的这个函数复制到.bash\_profile里，然后就能使用ftc这个命令了，用法比较简单就是ftc后面跟要拷贝的文件名（必须在文件所在的目录下执行），另外这个命令还比较简陋，不支持带全路径的文件名和多个文件拷贝。