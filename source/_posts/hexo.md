---
title: 利用 Hexo 搭建个人博客
date: 2017-07-12 10:21:19
comments: true
categories: Node
tags: 
    - hexo
    - node
    - git
toc: true
---

![](http://i.v2ex.co/5bb7J7NT.png)

Windows与Linux环境下方法大同小异。

推荐使用Jetbrains的WebStorm。

<!--more-->

更新

-----

 目前项目存在依赖问题有

```bash
#2017.7
npm WARN deprecated swig@1.4.2: This package is no longer maintained
npm WARN deprecated jade@1.11.0: Jade has been renamed to pug, please install the latest version of pug instead of jade
npm WARN deprecated transformers@2.1.0: Deprecated, use jstransformer
npm WARN prefer global marked@0.3.6 should be installed with -g
npm WARN prefer global node-gyp@3.6.2 should be installed with -g

//固有
npm WARN optional SKIPPING OPTIONAL DEPENDENCY: fsevents@^1.0.0 (node_modules/chokidar/node_modules/fsevents):
npm WARN notsup SKIPPING OPTIONAL DEPENDENCY: Unsupported platform for fsevents@1.1.2: wanted {"os":"darwin","arch":"any"} (current: {"os":"linux","arch":"x64"})

```


## 环境准备

### Node.js

对于Windows，[Node](https://nodejs.org)直接官网下载安装，无需配置环境变量。Node现已自带npm。

### Git与Github

[Git](https://git-scm.com/)可官网下载安装，不过下载速度不太稳定，似乎也无需配置环境变量。Windows下可能会有权限的问题（大部分出错的源头都在这），实在不行就用Git bash，但是安装时也不推荐选择“Git bash only”。

由于我们的博客是放在[Github](https://github.com/) pages上，最后的博客地址为< username\>.github.io,同时也是仓库名，因此需要在本地生成ssh密钥与Github账号进行连接。

> node与git配置参考另一篇：[新机器配置流程](https://antfaiz5z.github.io/2018/09/05/new-machine-init/)

## 正文

1. 创建仓库，< username\>.github.io

2. 创建两个分支：master与hexo(名称自定义)，

    设置hexo为默认分支（保存整个项目文件,方便之后在其他电脑上部署），master分支保存生成的网站的静态文件（Github pages如此）。使用
    
    ``` bash
    $ git clone git@github.com:<username>/<username>.github.io.git
    ```

   拷贝仓库,在本地< username\>.github.io文件夹下通过Git bash依次执行

    ``` bash
    $ npm install hexo -g
    $ hexo init
    $ npm install
    $ npm install hexo-deployer-git -g
    ```

    修改_config.yml中的deploy参数，分支应为master，依次执行

    ``` bash
    $ git add .
    $ git commit -m “…”
    $ git push origin hexo
    ```

    上传整个项目至远程仓库，执行

    ``` bash
    $ hexo generate
    $ hexo server
    ```

    浏览器访问[localhost:4000](http://localhost:4000),可以看到博客主页，执行

    ``` bash
    $ hexo deploy
    ```
    将网站部署到GitHub。

3. 在另一台新电脑上pull已有项目

    ``` bash
    git pull <url>
    ```

    不用执行命令“hexo init”

    ``` bash
    $ npm install
    $ npm install hexo -g
    $ npm install hexo-deployer-git -g
    ```
    若是原有项目安装了新主题，建议将主题目录移出.gitignore,保存原有主题配置属性，否则需要重新clone，不然会出现deploy时“no layout”的提示，导致页面空白。

    ```bash
    $ git clone https://github.com/xxx/yyy.git themes/yyy
    ```
    有些主题需要安装另外的插件，所以建议之前install时添加--save保存至package.json。

    > 至于主题修改、域名绑定、图床、各种plugin的添加，这里就不再赘述。

## 常用命令

- Hexo
    
    ``` bash
    $ npm install hexo -g #安装
    $ npm update hexo -g #升级
    $ hexo init #初始化
    ```

- 简写

    ``` bash
    $ hexo n "我的博客" == hexo new "我的博客" #新建文章
    $ hexo p == hexo publish
    $ hexo g == hexo generate#生成
    $ hexo s == hexo server #启动服务预览
    $ hexo d == hexo deploy#部署
    ```

- 服务器
    
    ``` bash
    $ hexo server #Hexo 会监视文件变动并自动更新，您无须重启服务器。
    $ hexo server -s #静态模式
    $ hexo server -p 5移出更改端口
    $ hexo server -i 192.168.1.1 #自定义 IP
    $ hexo clean #清除缓存 网页正常情况下可以忽略此条命令
    $ hexo g #生成静态网页
    $ hexo d #开始部署
    ```

- 监视文件变动
    
    ``` bash
    $ hexo generate #使用 Hexo 生成静态文件快速而且简单
    $ hexo generate --watch #监视文件变动
    ```

- 完成后部署，两个命令的作用是相同的

    ``` bash
    $ hexo generate --deploy
    $ hexo deploy --generate
    $ hexo deploy -g
    $ hexo server -g
    ```

- 草稿

    ``` bash
    $ hexo publish [layout] <title>
    ```

- 模版
    
    ``` bash
    $ hexo new "postName" #新建文章
    $ hexo new page "pageName" #新建页面
    $ hexo generate #生成静态页面至public目录
    $ hexo server #开启预览访问端口（默认端口4000，'ctrl + c'关闭server）
    $ hexo deploy #将.deploy目录部署到GitHub
    $ hexo new [layout] <title>
    $ hexo new photo "My Gallery"
    $ hexo new "Hello World" --lang tw
    ```
