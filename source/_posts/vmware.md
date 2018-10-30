---
title: 安装或更新vmware tools时的小问题
comments: true
date: 2015-11-27 23:18:29
categories: vmware
tags: vmware tools
---

最近安装`vmware tools`时遇到了些小问题,根本原因就是安装的是`ubuntu14.04`,挂载`cdrom`的点在`dev/s01 on /media/xxx(用户名)`,而不是要按照官方文档所说的去手动挂载在`/mnt/cdrom`.

由于挂载后此时`vmware tools`是只可读的,所以我们要将其复制到其他文件夹，比如`/tmp`.再用`tar`命令将`vmware tools`的`tar`包进行解压,最后运行解压后的文件夹`distribute`中的`install.pl`文件即可.
