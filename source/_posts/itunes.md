---
title: 关于iTunes更新后无法显示已连接iOS设备的问题
date: 2017-08-02 23:54:46
comments: true
categories: iTunes
tags: iTunes
---

1. 右键点按“设备管理器”中的 Apple iPhone、Apple iPad 或 Apple iPod 条目，然后从快捷菜单中选取更新驱动程序。

2. 点按“浏览计算机以查找驱动程序软件”。

3. 点按“从计算机的设备驱动程序列表中选择”。

4. 点按“从磁盘安装”按钮。（如果“从磁盘安装”选项未显示，请选取“移动电话”或“存储设备”等设备类别（如果列出），然后点按“下一步”。“从磁盘安装”按钮随即显示。）在“从磁盘安装”对话框中，点按“浏览”按钮。
使用此窗口导航到下列文件夹：
C:\Program Files\Common Files\Apple\Mobile Device Support\Drivers。
注：如果使用的是 64 位版本的 Windows，此文件夹可能存储在 C:\Program Files （x86）\Common Files\Apple\Mobile Device Support\Drivers。
连按此文件夹中列出的 usbaapl.inf 文件。（如果您拥有 64 位版本的 Windows，则此文件为 usbaapl64.inf。）

5. 点按“从磁盘安装”对话框上的“确定”。

6. 点按“下一步”并完成驱动程序安装步骤。

7. 打开 itunes 确认系统识别设备。