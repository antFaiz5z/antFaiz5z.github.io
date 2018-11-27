---
title: WSL
comments: true
toc: true
date: 2018-11-27 11:07:09
categories: C/C++
tags: WSL
---

## 心路历程

初学C/C++, 在windows上使用`codeblocks`写写算法甚是不错, `visual studio`宇宙最强IDE自然也是不虚(大概是太强了, 至今没有玩明白...).

后接触linux C开发, 便使用上了`vmware`, 快照回滚、双机调试、镜像备份等特别适合于折腾ubuntu时环境复现、编译内核啥的.

之后看到了deepin, 自带wine与软件商店尤适合于日常使用, 便执着于纯deepin或是deepin+windows双系统.

现觉, linux总是会出现各种兼容性问题比如显卡驱动、办公软件等, 诚然, linux的`斜杠`与`LF`和windows的`反斜杠`与`CRLF`相比, "人类"与"反人类"必定是前者令人心驰神往, 但时常由于工作场景需要在双系统或双机器间来回切换总归令人不快, 便寻摸着如何回归windows...

其实, cmake、git、docker、node等一直都有提供windows平台对应的十分成熟的软件，甚至提供GUI工具, 于是问题便在于可编写跨平台程序的C/C++编译器.

## 几种方案

### CygWin

MinGW与CygWin都能让你在windows下编译unix风格的C/C++代码. MinGW与CygWin的区别:

1. 修改编译器, 让windows下的编译器把诸如fork的调用翻译成等价的形式, 这就是MinGW的做法.
2. 修改库, 让windows提供一个类似unix提供的库,他们对程序的接口如同unix一样, 而这些库, 当然是由win32的API实现的, 这就是CygWin的做法.

CygWin与MinGW的最大区别在于：使用CygWin可以在Windows下调用unix-like的API（如fork,spawn,signals,select,sockets等）. 但是如果调用了unix特有的API函数, 在windows环境下不能正常运行, 如果想在windows下正常运行的，就必须依赖cygwin1.dll, 速度上会有些影响.

而用MinGW编译出来的程序, 如果源代码里面调用了unix环境的API, 则MinGW会把这些对UNIX的API调用翻译成win32下等价的形式. 同时这个程序是不能在windows下运行的.

如果你是想在windows环境下开发linux运行程序, 那么CygWin是你的不二之选. 而如果你想开发的是windows运行程序, 并且追求速度, 那么二者相比而言, MinGW是更好的选择.

### WSL

全称 Windows Subsystem for Linux

如今可在windows10应用商店可安装多个版本的linux.

#### 使用wslconfig命令进行管理

1. 设置默认运行的linux系统

```sh
wslconfig /setdefault <DistributionName>
```

2. 卸载linux系统

```sh
wslconfig /unregister <DistributionName>
```

当系统出现问题，我们可以卸载后重新安装。如：wslconfig /unregeister ubuntu

3. 查看已安装的linux系统

```sh
wslconfig /list
```

#### WSL文件系统与本地文件系统互相访问

1. WSL中访问本地文件

在“/mnt”目录下有“c”、“d”、“e”等文件夹, 分别表示本地的C盘、D盘、E盘.

2. 本地访问WSL的根目录

`C:\Users\XXXX\AppData\Local\Packages\CanonicalGroupLimited.UbuntuonWindows_79rhkp1fndgsc\LocalState\rootfs`

不过不建议在外部直接对其文件进行编辑、新建等操作，因为会出现一些问题。

## 安装图形界面

- [Win10Linux子系统（WSL）图形界面的安装](https://blog.csdn.net/novasliver/article/details/83190269)
- [占位]()

## Clion与WSL

利用xServer与xLaunch安装WSL的图形界面实际上是在本地利用ssh与子系统进行通信, 缺点是反应太慢．于是从常用工具同样是宇宙最强IDE inteliJ的开发商开发的C/C++IDE Clion出发考虑其他方案．

- [JetBrains官方指导：Working with WSL Environment](http://www.jetbrains.com/help/clion/how-to-use-wsl-development-environment-in-clion.html)

于是乎Clion采用的是子系统中的编译工具链, 完全无需考虑兼容性问题．

### cmder

windows的终端界面和字体实在太丑, 可下载cmder工具．由于WSL已提供bash, 于是下载mini版即可．

## 参考

- [在Linux的Windows子系统上(WSL)使用Docker（Ubuntu）](https://www.cnblogs.com/xiaoliangge/p/9134585.html)