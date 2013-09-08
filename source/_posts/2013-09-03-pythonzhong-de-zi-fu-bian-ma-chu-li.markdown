---
layout: post
title: "Python中的字符编码处理"
date: 2013-09-03 00:00
comments: true
categories: python,字符
---

相信那些经常使用Python处理中文或者其他非西文字符的人都多多少少被字符的编码转换困扰过。当然，只要是国内的程序员，不管你使用哪种编程语言都会遇到字符编码转换的问题，但这个问题在Python中更为突出。

我们知道在Python内部和许多第三方模块内部，字符串都默认是使用unicode来表示的（Python 2.x中字符串分为str类型和unicode类型，本文提到的带编码的字符串都是指str类型，而提到的unicode编码的字符串都是指unicode类型），也就是说，你传入一个字符串给一个内置函数或者第三方模块的接口，它们内部会事先将字符串的编码转换为unicode，而unicode最常见的实现方式是utf8编码（关于更多的字符编码信息请自行谷歌），所以这些函数通常会将获得的字符串参数默认为utf8编码，并尝试解码为unicode，这就导致了如果你传入的是非utf8编码的字符串，调用这些接口就会抛出异常，终止你的代码。

所以为了能正常调度这些接口，需要每次传入正确编码的字符串。众所周知，有个比较强大的Python第三方模块chardet，使用这个模块就能很方便的检测出大部分的字符串的字符编码。但这个模块存在2个问题，至少是我工作中遇到的比较显著的2个问题：第一，检测较长的字符串时（例如一个文本文件的编码）性能不太好，比较慢；第二，只要一个字符串中混入了一个字节的错误编码就会导致整个字符串的字符编码检测不出来。

下面是我针对这2个问题的优化方案（所有测试都是在Python 2.7.2中进行，系统OS X 10.8.4）__*(2013-09-03更新)*__：

```
def to_unicode(string):
    #1
    if isinstance(string, unicode):
        return string
    try:
        string_uni = string.decode("UTF-8")
        return string_uni
    except UnicodeDecodeError:
        pass
    
    #2
    INIT = 1000
    MAX = 10
    charset = None
    if len(string) <= INIT: 
        encoding_info = chardet.detect(string)
        charset = encoding_info['encoding']
    else:
        detector = UniversalDetector()
        for i in range(MAX):
            detector.feed(string[INIT*i:INIT*(i+1)])
            if detector.done:
                break
        detector.close()
        charset = detector.result.get('encoding')
    #3
    if not charset:
        INIT = 1
        MAX = 100
        detector = UniversalDetector()
        for i in range(MAX):
            detector.feed(string[INIT*i:INIT*(i+1)])
            if detector.done:
                mylog.LOG.ERROR("decode char one by one worked!")
                break
        detector.close()
        charset = detector.result.get('encoding')
    #4
    if not charset:
        return string.decode('UTF-8', 'ignore')
    elif charset[:2].upper() == 'GB':
        charset = 'GB18030'
    #5
    try:
        return string.decode(charset, 'replace')
    except UnicodeDecodeError:
        mylog.LOG.ERROR(traceback.format_exc())
        return string
```       
 
这个函数的功能是将一个字符串转为unicode类型，下面来解释下这段代码： ~~第一部分先用最常见的几个中文字符编码尝试去解码字符串~~，*第一部分先判断是否是unicode，来作兼容，因为如果传入的字符串本身就已经是unicode了，下面的代码都执行不了，然后先强制作utf8解码，(2013-09-03新加)*这是考虑到使用chardet的性能问题，因为直接调用decode函数的成本会比使用chardet.detect成本低很多。~~如果解码成功就返回，这几行代码基本能处理大部分的字符串。这里需要注意下几个编码的尝试顺序，必须将gb18030放在最后，因为偶数个字节的utf8编码字符串或者big5编码字符串都能被gb18030解码而不抛出异常，当然解码的结果是错误的。~~
*这里去掉了big5和gb18030编码的强行解码，因为发现gb18030编码的字符串可以被big5解码而不报错，反之亦然。至于原因目前还没搞懂，只知道gb2312是gbk的子集，gbk是gb18030的子集，各自都向下兼容。(2013-09-03更新)*

```
>>> a = '中文'
>>> a
'\xe4\xb8\xad\xe6\x96\x87'
>>> print a.decode('gb18030')
涓枃
>>> b = a.decode('utf8').encode('big5')
>>> b
'\xa4\xa4\xa4\xe5'
>>> b.decode('gb18030')
u'\u3044\u3085'
>>> print b.decode('gb18030')
いゅ
>>> c = a.decode('utf8').encode('gb18030')
>>> c
'\xd6\xd0\xce\xc4'
>>> print c.decode('big5')
笢恅
```

如果是奇数个字节的utf8编码字符串就不能被gb18030解码：

```
>>> c = '中文字'
>>> c
'\xe4\xb8\xad\xe6\x96\x87\xe5\xad\x97'
>>> c.decode('gb18030')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
UnicodeDecodeError: 'gb18030' codec can't decode byte 0x97 in position 8: incomplete multibyte sequence
```

这里使用gb18030，而没有使用gbk或者gb2312，是因为gb18030是后两者的超集。

第二部分中，对长度小于1000的字符串直接使用detect函数进行编码检测，而超过100的同样是考虑性能问题，采用了每次增量检测长度100的字符串，最多增加9次，即限制检测的最大的长度为1000，如果能提前确定了编码，就会跳出循环，避免了对大字符串的整体检测，因为一个字符串往往都是同一种编码。

第三部分中，如果第二部检测不到编码结果，往往是遇到了字符串中混有错误编码的情况，那么对字符串的前100个字符逐个检测，直到检测出编码或者检测完100个字符为止。检测完毕如果还是未检测出编码，就使用utf8解码，并传入‘ignore’，跳过不能解码的字节，减小错误编码造成的影响。

这里补充下decode函数的说明：

```
decode(...)
    S.decode([encoding[,errors]]) -> object
      
    Decodes S using the codec registered for encoding. encoding defaults
    to the default encoding. errors may be given to set a different error
    handling scheme. Default is 'strict' meaning that encoding errors raise
    a UnicodeDecodeError. Other possible values are 'ignore' and 'replace'
    as well as any other name registered with codecs.register_error that is
    able to handle UnicodeDecodeErrors.
```
    
上面是使用help查看decode函数的内置说明，可以看到调用decode，如果只传入编码信息，默认的异常处理模式为strict，只要遇到不能解码的就会抛出UnicodeDecodeError异常，另外你可以传入ignore来跳过不能解码的字节，或者replace来强制使用指定的编码解码。这个异常处理模式的指定，在encode函数中也同样适用。

例：

```
>>> '中文'
'\xe4\xb8\xad\xe6\x96\x87'
>>> '中文'.decode('utf8')
u'\u4e2d\u6587'
```

手动在上面utf8编码的字符串中混入一个错误编码，并且已经无法正常使用utf8解码

```
>>> d = '\xe4\xb8\xad\xdd\xe6\x96\x87'
>>> d
'\xe4\xb8\xad\xdd\xe6\x96\x87'
>>> print d
中?文
>>> d.decode('utf8')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/System/Library/Frameworks/Python.framework/Versions/2.7/lib/python2.7/encodings/utf_8.py", line 16, in decode
    return codecs.utf_8_decode(input, errors, True)
UnicodeDecodeError: 'utf8' codec can't decode byte 0xdd in position 3: invalid continuation byte
```

下面测试下使用ingore和replace后的不同

```
>>> d.decode('utf8', 'ignore')
u'\u4e2d\u6587'
>>> print d.decode('utf8', 'ignore')
中文
>>> d.decode('utf8', 'replace')
u'\u4e2d\ufffd\u6587'
>>> print d.decode('utf8', 'replace')
中�文
>>> '中文'.decode('utf8')
u'\u4e2d\u6587'
```

发现，使用ignore正确地跳过了错误编码，使用replace则会将那个错误编码强制转换，造成乱码。

*第4部分中，不知道是不是chardet的bug，如果是gb18030编码的字符串会被检测成为gb2312，而gb2312的字符集要比gb18030小，导致某些生僻字解码出错，这个函数里如果遇到是gb字符集都统一用gb18030去解码，保证解码成功。(2013-09-03更新)*

```
>>> a = '中文阿舒服的沙发法撒爹翻爾薾'.decode('utf8').encode('gb18030')
>>> chardet.detect(a)
{'confidence': 0.99, 'encoding': 'GB2312'}
>>> print a.decode('gb2312')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
UnicodeDecodeError: 'gb2312' codec can't decode bytes in position 24-25: illegal multibyte sequence
```

再回到上面的函数代码，第五部分中，不管之前有没有检测出编码，都会对字符串进行强制解码，当然保险起见，这里还加了try，来捕获有可能抛出的异常。

最后，虽然字符编码问题很令人头疼，但只要耐心搞清楚背后的原理，处理起来也没有想象中那么困难。
