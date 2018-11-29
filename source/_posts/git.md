---
title: 常用 git 命令
comments: true
toc: false
date: 2018-08-17 10:32:01
categories: Git
tags: git
---

## 常规

```sh
$ git add <path>/<file>
$ git commit -m "<commit_msg>"
$ git push origin master
```

<!--more-->

## Remote

```sh
$ git remote [-v | --verbose]
$ git remote add [-t <branch>] [-m <master>] [-f] [--[no-]tags] [--mirror=<fetch|push>] <name> <url>
$ git remote rename <old> <new>
$ git remote remove <name>
$ git remote set-head <name> (-a | --auto | -d | --delete | <branch>)
$ git remote set-branches [--add] <name> <branch>…​
$ git remote get-url [--push] [--all] <name>
$ git remote set-url [--push] <name> <newurl> [<oldurl>]
$ git remote set-url --add [--push] <name> <newurl>
$ git remote set-url --delete [--push] <name> <url>
$ git remote [-v | --verbose] show [-n] <name>…​
$ git remote prune [-n | --dry-run] <name>…​
$ git remote [-v | --verbose] update [-p | --prune] [(<group> | <remote>)…​]
```

## 查看日志

```sh
$ git status
# 查看当前分支修改状态
$ git log
# log命令可以显示所有提交过的版本信息
$ git log - -pretty=oneline
# 将只会显示提交的commit id号和对应的注释。
$ git reflog
# 如果在回退以后又想再次回到之前的版本，git reflog 可以查看所有分支的所有操作记录（包括commit和reset的操作），包括已经被删除的commit记录，git log则不能察看已经删除了的commit记录
```

## 版本回退

```sh
#本地分支暂存区回退
$ git rm [-r] <path>/<file>
# 对应于git add，使文件不加入版本管理
$ git checkout -- <file>
# 让<file>这个文件回到最近一次git commit或git add时的状态
# 一种是<file>自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；
# 一种是<file>已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态。

#本地分支提交回退
$ git reset HEAD <file>
# git reset命令既可以回退版本，也可以把暂存区的修改回退到工作区。当我们用HEAD时，表示最新的版本.
$ git reset [--hard] <commit_id>
# hard选项，表示彻底将工作区、暂存区和版本库记录恢复到指定的版本库

#远程分支版本回退
# 远程版本回退可在本地回退后再强制推送至远程分支
$ git push -f origin <branch>
# 本地分支回滚后，版本将落后远程分支，必须使用强制推送覆盖远程分支，否则无法推送到远程分支
```

![reset and checkout](http://osxg0gzju.bkt.clouddn.com/reset_and_checkout.png)

## 分支管理

```sh
$ git branch
# 查看当前分支情况

#本地新建分支
$ git branch <new_branch>
# 以当前分支为源创建<new branch>分支
$ git checkout <new_branch>
# 进入<new branch>分支
#合体
$ git checkout -b <new_branch>
# 以当前分支为源创建新分支<new branch>，并进入<new branch>分支

#远程分支创建
# 本地分支创建后推送至远程仓库即可
$ git push origin <new_branch>:master
# 提交本地<new_branch>分支作为远程的master分支
$ git push origin <new_branch>:<new_branch>
# 提交本地<new_branch>分支作为远程的<new_branch>分支
$ git push origin :<new_branch>
# 刚提交到远程的<new_branch>分支将被删除，但是本地还会保存
```

## 子仓库

```sh
#在仓库当前路径添加子模块
git submodule add <git_url>

#克隆包含子模块的仓库
# 方法一
#克隆项目，子模块目录默认被克隆，但是是空的
$ git clone <git_url>
#初始化子模块：初始化本地配置文件
$ git submodule init
#该项目中抓取所有数据并检出父项目中列出的合适的提交
$ git submodule update
# 方法二
#用--recursive命令，跟方法一样达到效果
$ git clone --recursive <git_url>

#更新子模块
$ git submodule update --remote --merge
# 或者进入子模块目录如普通仓库一样操作

#删除子模块
$ git rm --cached [name]
$ rm -rf [name]
$ rm .gitmodules
$ vim .git/config
# 删除子模块相关内容，例如下面的内容
[submodule "submodule"]
        url = git@gitee.com:xxx.com/submodule.git
        active = true
        
#主仓库推送
$ git push --recurse-submodules=check
# 主仓库推送时，确保子模块的修改已经推送，下面命令会检查子模块修改的内容是否推送，如果没有，主仓库推送也会失败
$ git push --recurse-submodules=on-demand
# 检查到子模块没有推送，会自动推送子模块，然后再推送主模块（如果子模块推送失败，那么主模块也推送失败）
```
