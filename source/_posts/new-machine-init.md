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

### git/github

1. 查看电脑上是否已经存在SSH密钥：

    ``` bash
    $ cd ~/.ssh
    ```

    若显示“No such file or directoy”,则需要创建新的ssh key

2. 创建新的ssh key:

    ``` bash
    $ ssh-keygen -t rsa -C "<youremail>"
    ```

    执行这条命令会提示文件保存路径，可以直接Enter，默认保存在当前用户目录。然后提示输入passphrase，可以直接Enter，即无密码.
 
3. 复制ssh key到Github

    用编辑器打开.ssh目录下的id_rsa.pub文件，复制里面的全部内容，add ssh key

4. 测试ssh连接Github：

    ``` bash
    $ ssh -T git@github.com
    ```

    期间会提示输入密码，若无则无

5. 设置自己的git信息：

    ``` bash   
    $ git config --global user.name "username" 
    （此处username可修改也不是用于登录github的登录名）
    $ git config --global user.email "<youremail>"
    ```

    设置自己的git信息即完成安装和设置，可以输入

    ``` bash
    $ git config --list
    ```

    查看自己的git信息

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

### nvm/node/npm

访问 https://github.com/creationix/nvm 查看nvm最新release版本

```sh
$ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.1/install.sh | bash
```

访问 https://nodejs.org/zh-cn/ 查看node最新LTS版本

```sh
$ nvm install [version]
```

npm换源：

1. 临时使用
    
    ``` bash
    $ npm --registry https://registry.npm.taobao.org install express
    ```

2. 持久使用

    ``` bash
    $ npm config set registry https://registry.npm.taobao.org
    ```

    配置后可通过下面方式来验证是否成功 

    ``` bash
    $ npm config get registry
    ```

3. 通过cnpm使用

    ``` bash
    $ npm install -g cnpm --registry=https://registry.npm.taobao.org
    ```

    npm命令都可用cnpm代替。

### C/C++

```sh
$ sudo apt-get install cmake gcc clang gdb build-essential
```

### anacode

https://www.anaconda.com/download/#linux

### blog

https://antfaiz5z.github.io/2017/07/12/hexo/