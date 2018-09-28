---
title: cred, systemd, QEMU, Yocto, AGL
comments: true
toc: true
date: 2017-09-12 17:06:20
categories: Linux
tags: 
    - cred
    - systemd
    - QEMU
    - Yocto
    - AGL
---

## [cred](http://blog.csdn.net/lhj0711010212/article/details/8521377)

## [systemd](https://www.ibm.com/developerworks/cn/linux/1407_liuming_init3/)

Systemd 是 Linux 系统中最新的初始化系统（init），它主要的设计目标是克服 sysvinit 固有的缺点，提高系统的启动速度。systemd 和 ubuntu 的 upstart 是竞争对手，预计会取代 UpStart，实际上在作者写作本文时，已经有消息称 Ubuntu 也将采用 systemd 作为其标准的系统初始化系统。

## QEMU

QEMU是一套由法布里斯·贝拉(Fabrice Bellard)所编写的以GPL许可证分发源码的模拟处理器，在GNU/Linux平台上使用广泛。Bochs，PearPC等与其类似，但不具备其许多特性，比如高速度及跨平台的特性，通过KQEMU这个闭源的加速器，QEMU能模拟至接近真实电脑的速度。
目前，0.9.1及之前版本的qemu可以使用kqemu加速器。在qemu1.0之后的版本，都无法使用kqemu，主要利用qemu-kvm加速模块，并且加速效果以及稳定性明显好于kqemu。

## [Yocto](http://blog.csdn.net/qq_28992301/article/details/52872209)
[如何在 Ubuntu 上用 Yocto 创建你自己的嵌入式 Linux 发行版](https://linux.cn/article-8268-1.html?amputm_medium=rss)

## [AGL](http://docs.automotivelinux.org/docs/devguides/en/dev/)