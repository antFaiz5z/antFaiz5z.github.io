---
title: Scrapy安装及使用
comments: true
date: 2017-08-03 10:02:27
categories: Scrapy
tags: 
    - Scrapy
    - curl
    - pip
    - Anacoda
    - virtualenv
    - PyCharm
---

## 安装

~~[Curl的安装和使用](http://blog.csdn.net/lifan5/article/details/7350154)~~

~~[curl不能支持https的问题](http://blog.csdn.net/wvtear/article/details/9817033)~~

~~[如何在Linux中安装pip](http://www.linuxidc.com/Linux/2017-07/145560.htm)~~

[Anaconda使用总结](http://python.jobbole.com/86236/)
>conda is a tool for managing and deploying applications, environments and packages.

所以，直接按照Scrapy的官方文档安装Anacoda和Scrapy就好了，并且官方建议使用virtualenv

[virtualenv简明教程](http://www.jianshu.com/p/08c657bd34f1)

[Scrapy 1.4 documentation](https://doc.scrapy.org/en/latest/index.html)

[XPath 语法](http://www.w3school.com.cn/xpath/xpath_syntax.asp)
## 使用命令行

在项目目录下运行:

```bash
scrapy crawl xxxxx
```

## 使用PyCharm

如果要用Pycharm作为开发调试工具的话可以在运行配置里进行如下配置：

1. Run->Configuration页面：

    Script填你的scrapy的cmdline.py路径

    若是使用Anacoda安装，一般为：

        /home/faiz/anaconda2/lib/python2.7/site-packages/scrapy/cmdline.py

    若是使用pip安装，一般为：

        /usr/local/lib/python2.7/dist-packages/scrapy/cmdline.py

2. 然后在Scrpit parameters中填爬虫的名字，
            
        crawl xxxxx

3. 最后是Working diretory，找到你的settings.py文件，填这个文件所在的目录。

4. 按小绿箭头就可以愉快地调试了。