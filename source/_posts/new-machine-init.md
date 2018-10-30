---
title: 新装系统环境配置流程个人版
comments: true
toc: true
date: 2018-09-05 19:27:04
categories: Others
tags:
---

Personal.

<!--more-->

### ss

```sh
$ sudo apt-get install shadowsocks-qt5
```

设置系统代理->手动:

Socks代理: 127.0.0.1

本地端口: 1080

### zsh

```sh
$ sudo apt-get install -y zsh
```
OhMyZsh

```sh
$ wget https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O - | sh
```

```sh
$ chsh -s /bin/zsh
```

reboot terminal

```sh
$ vim ~/.zshrc
```

```sh
ZSH_THEME="agnoster"
```

为了展示 Agnoster 主题提示符里的三角形，需要 Powerline 字体库的支持。

使用 Git Clone下来字体库仓库，进入仓库根目录，执行 install.sh 安装。

字体库链接：https://github.com/powerline/fonts 或直接

```sh
$ sudo apt-get install fonts-powerline
```

### nvm

访问 https://github.com/creationix/nvm 查看nvm最新release版本

```sh
$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.1/install.sh | bash
```

访问 https://nodejs.org/zh-cn/ 查看node最新LTS版本

```sh
$ nvm install [version]
```

### C/C++

```sh
$ sudo apt-get install build-essential
```

### anacode

https://www.anaconda.com/download/#linux

### blog

https://antfaiz5z.github.io/2017/07/12/hexo/