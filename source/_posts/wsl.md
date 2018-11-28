---
title: Back to Windows with WSL or CygWin ?
comments: true
toc: true
date: 2018-11-27 11:07:09
categories: C/C++
tags: WSL
---

## 心路历程

初学 C/C++, 在windows上使用 codeblocks 写写算法甚是不错,  visual studio 宇宙最强 IDE 自然也是不虚(大概是太强了, 至今没有玩明白...).

后接触 linux  开发, 便使用上了 vmware , 快照回滚、双机调试、镜像备份等特别适合于折腾 ubuntu 时环境复现、编译内核啥的.

之后看到了 deepin , 自带 wine 与软件商店尤适合于日常使用, 于是一直执着于纯 deepin 或是 deepin + windows 双系统.

现觉, linux 总是会出现各种兼容性问题比如显卡驱动、办公软件等, 诚然, linux 的斜杠与 LF 和 windows 的反斜杠与 CRLF 相比, "人类"与"反人类"必定是前者令人心驰神往, 但时常由于工作场景需要在双系统或双机器间来回切换总归令人不快, 便寻摸着如何回归 windows...

其实, cmake、git、docker、node等一直都有提供 windows 平台对应的十分成熟的软件，甚至提供 GUI 工具, 于是问题便在于可编写跨平台程序的 C/C++ 编译器.

## 几种方案

### CygWin

MinGW 与 CygWin 都能让你在 windows 下编译 unix 风格的 C/C++ 代码. MinGW 与 CygWin 的区别:

1. 修改编译器, 让 windows 下的编译器把诸如 fork 的调用翻译成等价的形式, 这就是 MinGW 的做法.
2. 修改库, 让 windows 提供一个类似 unix 提供的库,他们对程序的接口如同 unix 一样, 而这些库当然是由 win32 的 API 实现的, 这就是 CygWin 的做法.

CygWin 与 MinGW 的最大区别在于：使用 CygWin 可以在 windows 下调用 unix-like 的 API （如fork、spawn、signals、select、sockets 等）. 但是如果调用了 unix 特有的 API 函数, 在 windows 环境下不能正常运行, 如果想在 windows 下正常运行的，就必须依赖 cygwin1.dll , 速度上会有些影响.

而用 MinGW 编译出来的程序, 如果源代码里面调用了 unix 环境的 API , 则 MinGW 会把这些对 UNIX 的 API 调用翻译成 win32 下等价的形式. 同时这个程序是不能在 windows 下运行的.

如果你是想在 windows 环境下开发 linux 运行程序, 那么 CygWin 是你的不二之选. 而如果你想开发的是 windows 运行程序, 并且追求速度, 那么二者相比而言, MinGW 是更好的选择.

### WSL

全称 Windows Subsystem for Linux

如今可在 windows 10 应用商店直接下载安装多个版本的 linux .

#### 使用 wslconfig 命令进行管理

1. 设置默认运行的 linux 系统

```sh
wslconfig /setdefault <DistributionName>
```

2. 卸载 linux 系统

```sh
wslconfig /unregister <DistributionName>
```

当系统出现问题，我们可以卸载后重新安装。如：wslconfig /unregeister ubuntu

3. 查看已安装的 linux 系统

```sh
wslconfig /list
```

#### 文件系统互相访问

1. WSL中访问本地文件

在“/mnt”目录下有“c”、“d”、“e”等文件夹, 分别表示本地的 C 盘、D 盘、E 盘.

2. 本地访问 WSL 的根目录

`C:\Users\XXXX\AppData\Local\Packages\CanonicalGroupLimited.UbuntuonWindows_79rhkp1fndgsc\LocalState\rootfs`

不过不建议在外部直接对其文件进行编辑、新建等操作，因为会出现一些问题。

## 安装图形界面

- [Windows linux子系统--入门到GUI](http://csuncle.com/2017/08/08/Windows-linux%E5%AD%90%E7%B3%BB%E7%BB%9F-%E5%85%A5%E9%97%A8%E5%88%B0GUI/)
- [Win10Linux子系统（WSL）图形界面的安装](https://blog.csdn.net/novasliver/article/details/83190269)

## Clion 与 WSL

利用 xServer 与 xLaunch 安装 WSL 的图形界面实际上是在本地利用 ssh 与子系统进行通信, 缺点是反应太慢．于是从常用工具同样是宇宙最强 IDE inteliJ 的开发商开发的 C/C++ IDE Clion 出发考虑其他方案．

- [JetBrains官方指导：Working with WSL Environment](http://www.jetbrains.com/help/clion/how-to-use-wsl-development-environment-in-clion.html)

于是乎 Clion 采用的是子系统中的编译工具链, 完全无需考虑兼容性问题．

### cmder

windows 的终端界面和字体实在太丑, 可下载 cmder 工具．由于 WSL 已提供 bash, 于是下载 mini 版即可．

## 其他参考

- [在Linux的Windows子系统上(WSL)使用Docker（Ubuntu）](https://www.cnblogs.com/xiaoliangge/p/9134585.html)

- [中文文案排版指北（简体中文版）](https://github.com/mzlogin/chinese-copywriting-guidelines)