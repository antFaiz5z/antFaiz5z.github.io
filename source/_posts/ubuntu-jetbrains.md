---
title: 在Deepin系统（Debian系）下安装软件
date: 2017-08-01 16:45:00
comments: true
categories: Linux
tags: deepin
---

从Deepin Store中直接安装软件，有时会导致普通用户启动软件存在权限不足的问题，所以干脆使用命令行自行安装。

以下以Jetbrains的WebStorm为例。

首先在官网直接下载或使用`wget`命令下载tar包，解压。将解压后的目录移动到`/usr/share/`（似乎是约定俗成的安装目录？）

<!--more-->

```bash
sudo mv <下载目录>/WebStorm-xxx/ /usr/share/[<重命名目录名>]
```

>需注意目标目录最后一个斜杠的有无的区别

创建一个软链接，**重要目录** `/usr/local/bin/`

```bash
sudo ln -s /usr/share/<上文目录名>/bin/webstorm.sh /usr/local/bin/<自定义链接名>
```

之后，可以通过自定义的链接名命令来启动应用程序