---
title: Python 2与 Python 3的一些坑
comments: true
toc: true
date: 2017-08-09 11:44:24
categories: Python
tags: 
    - python
    - pycharm
---

Python的编码真的是好烦啊。

<!--more-->

## ascii 与 unicode

对于Python代码中避免遇到编码问题，有一些小建议：

- 字符编码声明：在代码开头声明编码格式
- 使用codecs的open函数处理文本文件
- 尽可能使用unicode而不是str：在所有字符串的引号前加u

以下方法不总是有用：

```python
import sys  
reload(sys)  
sys.setdefaultencoding('utf8')
```
## 请求URL

值得注意的是，Python2中可以使用`urllib`，但推荐使用更简单的`urllib2`，而在Python3中无法使用`urllib2`。
```python
#Python2：

import urllib2

url ="http://xxxx"  
data = urllib2.urlopen(url).read().decode('utf-8')

#Python3：

 from urllib import request

 url ="http://xxxx" 
 response = request.urlopen(url).read().decode('utf-8')
```

## URL中存在中文

>Python2 有ASCII str()类型，unicode()是单独的，不是 byte 类型。
Python3中，有了 unicode (utf-8) 字符串，以及一个字节类：byte 和 bytearrays。 

>并且Python3.X 源码文件默认使用utf-8编码。

尽管如此，Python3的unicode编码也无法应对中文URL的问题，可以使用`quote()`函数处理中文部分再拼接成完整URL或使用safe参数过滤不需要处理的部分来直接quote整个URL。

```python
#Python2：

import urllib2
import  string

url = urllib2.quote(url, safe=string.printable)

#Python3：

from urllib.parse import quote
import  string

url = quote(url, safe = string.printable)  # safe表示可以忽略的字符
```

## 以二进制方式读写文件

以如`'wb'`、`'rb'`等二进制方式读写文件时，如 `open(filename,'wb')`，在Python3中使用`str()`函数如写入str时可能会出现如下错误，而该错误不在Python2中出现。

```bash
python 3.5: TypeError: a bytes-like object is required, not 'str'
```
具体解决方法有以下两种：

第一种，在open()函数中使用如`'w'`、`'w+'`、`'r+'`属性，即文本方式写入，可以直接解决问题。

第二种，依然在`open()`函数中使用`'wb'`、`'rb'`,可以在使用之前进行转换，如[实例](http://stackoverflow.com/questions/33054527/python-3-5-typeerror-a-bytes-like-object-is-required-not-str)


## import module urllib.xxx

使用Python3时，使用PyCharm点击run正常运行，而在shell中运行 `python xxx.py` 会出现如下问题：

```bash
from urllib import request
ImportError: No module named request

from urllib.parse import quote
ImportError: No module named parse
```

**此问题尚未解决！**

于是改用Python2，使用urllib2，无报错。
