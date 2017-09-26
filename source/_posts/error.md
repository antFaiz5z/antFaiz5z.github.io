---
title: 一些问题
comments: true
toc: false
date: 2017-09-12 17:06:20
categories: Linux
tags: Linux
---

- `No package 'dbus-1' found`

```bash
sudo apt-get install libdbus-1-dev
```

- [`error while loading shared libraries: xxx.so.x`](http://www.cnblogs.com/lidabo/p/3935250.html)


```bash
# cat /etc/ld.so.conf
include ld.so.conf.d/*.conf
# echo "/usr/local/lib" >> /etc/ld.so.conf
# ldconfig
```

- 动态链接库的创建与使用

```bash
#编译生产libexample.so文件如下：
gcc -shared -fPIC func.c -o libexample.so

#编译生产可执行文件main如下：
gcc main.c -o main -L ./ -l libexample.so
#其中-L指明动态链接库的路径，-l后是链接库的名称
```
