---
layout: post
title: "python正则匹配中match与search的区别"
date: 2013-09-26 16:28
comments: true
categories: 
- python
- 正则表达式
---

这2个函数的区别简单来说就是match是从目标字符串开头开始匹配起，而search是会匹配目标字符串的任意位置。

	>>> re.match("c", "abcdef")  # No match
	>>> re.search("c", "abcdef") # Match
	<_sre.SRE_Match object at 0x2b1f072946b0>

也可以理解为，match是比search在匹配时多加了个"^"，也就是说使用search的时候在匹配字符串前加个"^"，那结果就跟match一致了。但是大部分情况下如此，但不完全相同，后文详解。

<!--more-->

	>>> re.match("c", "abcdef")    # No match
	>>> re.search("^c", "abcdef")  # No match
	>>> re.search("^a", "abcdef")  # Match
	<_sre.SRE_Match object at 0x2b1f07294718>

如果使用match是在匹配字符串前增加".*"，那结果也跟search一样了。

	>>> re.match("c", "abcdef")    # No match
	>>> re.match(".*c", "abcdef")  # Match
	<_sre.SRE_Match object at 0x2b1f07294718>
	>>> re.search("c", "abcdef")   # Match
	<_sre.SRE_Match object at 0x2b1f072946b0>

前文提到的不完全相同主要是存在与匹配多行字符串的情况下，match只会搜索一行，而search会逐行搜索。

	>>> re.match('X', 'A\nB\nX', re.MULTILINE)    # No match
	>>> re.search('^X', 'A\nB\nX', re.MULTILINE)  # Match
	<_sre.SRE_Match object at 0x2b1f07294718>
	>>> re.match('.\*X', 'A\nB\nX', re.MULTILINE)  # No match


