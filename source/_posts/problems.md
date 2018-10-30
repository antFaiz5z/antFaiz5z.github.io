---
title: (Always-Update) 常见问题
comments: true
toc: false
date: 2017-12-18 16:51:23
categories: Linux
tags: 
  - always-update
  - deepin
  - makefile
---

<!--more-->

## Deepin 下 NTFS 磁盘为只读属性

此种情况是由于 Windows 开启了快速启动, 这时在 Linux 里面使用 ntfs-3g 挂载这个NTFS 分区时就只能以只读方式挂载了, 不然可能会影响 Windows 的快速启动.

不需要重装系统或者在Windows系统中关闭快速启动, 执行以下命令即可

```sh
$ df -h
$ sudo ntfsfix /dev/sdaxxx
```

如果已经手动删除了 Windows 系统的话, 可以手动挂载 NTFS 分区.

ntfs-3g 有 remove_hiberfile 选项, 用来处理这个问题的, 具体可以参考 man ntfs-3g 手册.

## makefile：变量替换

```makefile
SRCS := $(wildcard src/*.c)
OBJS := $(SRCS:%.c=%.o)
```

`$(SRCS:%.c=%.o)`中不可有空格，否则失效

## `No package 'dbus-1' found`

```bash
sudo apt-get install libdbus-1-dev
```

## `No package “gio-2.0” found`

```bash
sudo apt-get install libperl-dev libgtk2.0-dev
```

## [`error while loading shared libraries: xxx.so.x`](http://www.cnblogs.com/lidabo/p/3935250.html)

1、如果共享库文件安装到了/lib或/usr/lib目录下, 那么需执行一下ldconfig命令

ldconfig命令的用途, 主要是在默认搜寻目录(/lib和/usr/lib)以及动态库配置文件	/etc/ld.so.conf内所列的目录下, 搜索出可共享的动态链接库(格式如lib*.so*), 进	而创建出动态装入程序(ld.so)所需的连接和缓存文件. 缓存文件默认为	/etc/ld.so.cache, 此文件保存已排好序的动态链接库名字列表. 

2、如果共享库文件安装到了/usr/local/lib(很多开源的共享库都会安装到该目录下)或其它"非/lib或/usr/lib"目录下, 那么在执行ldconfig命令前, 还要把新共享库目录加入到共享库配置文件/etc/ld.so.conf中, 如下:

```bash
sudo cat /etc/ld.so.conf
  include ld.so.conf.d/*.conf
sudo echo "/usr/local/lib" >> /etc/ld.so.conf
sudo ldconfig
```

3、如果共享库文件安装到了其它"非/lib或/usr/lib" 目录下,  但是又不想在/etc/ld.so.conf中加路径(或者是没有权限加路径). 那可以export一个全局变量LD_LIBRARY_PATH, 然后运行程序的时候就会去这个目录中找共享库. 

LD_LIBRARY_PATH的意思是告诉loader在哪些目录中可以找到共享库. 可以	设置多个搜索目录, 这些目录之间用冒号分隔开. 比如安装了一个mysql到	/usr/local/mysql目录下, 其中有一大堆库文件在/usr/local/mysql/lib下面, 则可以	在.bashrc或.bash_profile或shell里加入以下语句即可:

export LD_LIBRARY_PATH=/usr/local/mysql/lib:$LD_LIBRARY_PATH    

一般来讲这只是一种临时的解决方案, 在没有权限或临时需要的时候使用.

## 动态链接库的创建与使用

```bash
#编译生产libexample.so文件如下：
gcc -shared -fPIC func.c -o libexample.so

#编译生产可执行文件main如下：
gcc main.c -o main -L ./ -l libexample.so
#其中-L指明动态链接库的路径，-l后是链接库的名称
```

## 安装交叉编译环境

```bash
sudo apt-get install gcc-arm-linux-gnueabihf
```

默认安装版本为5.X， 编译内核时会报无gcc5.h的错误，
于是安装4.X：

```bash
sudo apt-get install gcc-4.7-arm-linux-gnueabihf
```

由于指定安装4.7版本不会创建arm-linux-gnueabih-gcc[-**]等命令的软链接，于是强行覆盖指向5.x各命令的软链接：

```bash
ln -sf arm-linux-gnueabih-gcc[-**]-4.7  arm-linux-gnueabih-gcc[-**]
```