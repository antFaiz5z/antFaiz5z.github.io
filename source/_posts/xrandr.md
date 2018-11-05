---
title: Deepin下笔记本外接显示器问题
comments: true
toc: false
date: 2018-11-05 20:04:29
categories: Others
tags:
    - deepin
    - xrandr
---

最近用小米游戏本的HDMI口外接显示器时发现，只有启动时有Deepin logo一闪而过，之后无信号输出。

解决办法如下：

```sh
$ xrandr --listproviders # 如果有多个provider，那么可能是不同显示口在不同显卡上
$ xrandr --setprovideroutputsource 0 1 # 或
$ xrandr --setprovideroutputsource 1 0 # 做下链接，就可以看到所有硬件接口
$ xrandr --output xxx --auto # 设置输出,xxx形如VGA-1、HDMI-1-1等
```

问题是重启失效，于是设置开机自启脚本：

将上设置命令保存到新建文件`10custom_init_xrandr`，保存至`/etc/X11/Xsession.d/`目录中。

`xrandr --output`指令示例：

```sh
$ xrandr --output VGA --same-as LVDS --auto
# 打开外接显示器(--auto:最高分辨率)，与笔记本液晶屏幕显示同样内容（克隆）
$ xrandr --output VGA --same-as LVDS --mode 1280x1024
# 打开外接显示器(分辨率为1280x1024)，与笔记本液晶屏幕显示同样内容（克隆）
$ xrandr --output VGA --right-of LVDS --auto
# 打开外接显示器(--auto:最高分辨率)，设置为右侧扩展屏幕
$ xrandr --output VGA --off
# 关闭外接显示器
$ xrandr --output VGA --auto --output LVDS --off
# 打开外接显示器，同时关闭笔记本液晶屏幕（只用外接显示器工作）
xrandr --output VGA --off --output LVDS --auto
# 关闭外接显示器，同时打开笔记本液晶屏幕 （只用笔记本液晶屏）
```