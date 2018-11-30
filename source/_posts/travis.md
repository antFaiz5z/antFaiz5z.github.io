---
title: 利用 Travis CI 自动部署博客
comments: true
toc: true
date: 2018-11-30 12:22:36
categories: CI
tags: 
    - travis
    - hexo
---

> 持续集成（英语：Continuous integration，缩写 CI）是一种软件工程流程，是将所有软件工程师对于软件的工作副本持续集成到共享主线（mainline）的一种举措。

简单来说，我们通过持续集成，能够简化 Hexo 发布博客的步骤，即：将清除缓存 hexo clean，生成静态文件 hexo generate 和部署到 GitHub Pages hexo deploy 这些步骤通过持续集成工具来帮助我们自动执行。

这样我们在本地对博客文件进行修改、新增博文内容或者新增博客文章，只需要通过 git 推送到 GitHub 仓库之后，持续集成工具就可以帮助我们在线构建博客静态文件并直接部署到 GitHub Pages。这之后，我们发布博客内容就不需要本地三步走了。

### Travis CI

我们所利用的持续集成平台是 Travis CI.
首先，我们到 [Travis CI](https://travis-ci.org/) 官网，用自己的 GitHub 账户直接关联登录，并允许 Travis CI 查看自己的公有仓库。

然后我们到 Travis CI 账户页面 开启我们的博客仓库。

### 配置持续集成文件

.travis.yml 是 Travis CI 的部署配置文件，Travis CI 部署时会自动读取我们每次 Commit 中是否包含 .travis.yml，有此文件才会开始部署。

在博客项目源代码分支下创建文件 .travis.yml，并添加如下内容：

```yml
language: node_js

sudo: required

node_js: stable

install:
  - npm install 

branches:
  only:
    - whopro

before_install: 
  - export TZ='Asia/Shanghai'
  - npm install -g cnpm --registry=https://registry.npm.taobao.org
  - cnpm install hexo-cli -g

install:
  - cnpm install

script:
  - hexo clean
  - hexo generate

deploy:
  provider: pages
  skip_cleanup: true
  github_token: $GITHUB_TOKEN
  local_dir: public
  name: $GIT_NAME
  email: $GIT_EMAIL
  keep-history: true
  target-branch: master
  on:
    branch: whopro
```
其中，

- language：编译语言、环境；

- node_js：Node.js 版本；

- sudo：需要管理员权限；

- install：安装环境 npm；

- branches：工作仓库分支（hexo 分支）；

- before_install：配置时区为中国时区东八区（UTC + 8），安装组件 hexo；

- install：安装依赖 npm install；

- script：执行脚本，清除缓存，生成静态文件并放在 public 文件夹下；

- deploy：执行部署。

> 其他文档可能提到了利用 hexo-deployer-git 进行部署，但是由于 Travis CI 本身支持直接部署到 GitHub Pages 的工具，因此无需另行安装 hexo-deployer-git 了；<br>
其他文档也可能提到在 .travis.yml 中加入如下内容，来缓存 node_modules 下的内容，从而加快编译速度。但是经过我的尝试，node_modules 经常会由于没有及时更新，在添加其他组件之后出现「博客生成静态文件步骤」失败的情况，因此建议不进行缓存处理。

```yml
cache:
  directories:
    - node_modules
```

### 在 Travis CI 中配置变量

在配置文件中我们使用了三个变量：

- $GIT_NAME：git 用户名

- $GIT_EMAIL：git 用户邮箱

- $GITHUB_TOKEN：GitHub 通行证 (token) 字符串

最后一项 GitHub 通行证 (token) 我们在 GitHub 中进行申请：

访问 GitHub 账户设置 > Tokens

生成新 Token: Generate new token

填入 Token 描述，并给予 Token 第一项 repo 的全部权限

将生成的 Token 复制，保存（生成 Token 的页面只有一次机会看见，请保存妥当。）

在 Travis CI 仓库配置中，将三个变量填入设置（位于 Settings > Environment Variables 处并保存：

在 Travis CI 仓库配置中，将三个变量填入设置（位于 Settings > Environment Variables 处并保存：

这样，在每次我们将博客的源文件通过 git 推送到 GitHub 的 hexo 分支上后，Travis CI 就会自动检测并主动开始构建我们的博客静态文件，并自动部署到 GitHub Pages 中。