---
title: 面试问题整理
comments: true
toc: true
date: 2019-03-21 15:37:13
categories: Others
tags:
---

本文主要针对于 C++ 面试中常见问题进行整理并尽量作口语化而非书面化的回答。

<!--more-->

# 正文

## C/C++基础

- 内存布局
- 内存对齐
- 指针与引用
- new、delete、malloc、free
- 虚函数、虚表、虚表指针
- 智能指针
- static_cast、dynamic_cast、const_cast、reinterpret_cast

### STL

- 容器：分类、复杂度
- vector
- set、map
- allocator
- iterator、traits

## TCP/IP

- TCP 与 UDP
- 三次握手

    首先，接收方调用 listen() 处于 LISTEN 状态，</br>
    发送方调用 connect() 发送一个 SYN 包，转为 SYN_SNET 状态，</br>
    接收方接收到后发送 SYN + 相应的 ACK 包，转为 SYN_RCVD 状态，</br>
    发送方收到后转为 ESTABLISHED 状态，接着再发送相应的 ACK 包，</br>
    接收方收到后也转为 ESTABLISHED 状态，</br>
    至此，全双工通信的可靠连接建立完成。

    * 为什么TCP建立连接要三次握手

    首先得回答三次握手的目的是同步连接双方的序列号和确认号并交换 TCP 窗口大小信息。然后可以回答为什么两次握手不行，两次握手可能因为丢包而出现死锁，假设在两次握手场景中，C向S发送请求，S收到并发送确认请求给C，这时候S认为连接已经建立，并开始发送数据给C，但是那个确认请求丢包了，C不认为请求建立了，C当然会拒绝接受S发送来的数据，并且再去请求连接。这样，一个资源就死锁了。最后回答握手当然可以四次五次一直握下去，但三次已经够了，就没有必要了。总结下来一句话，主要目的防止在网络发生延迟或者丢包的情况下浪费资源。

- 四次挥手

    首先，发送方主动调用 close() 发起连接断开请求，发送一个 FIN 包，进入 FIN_WAIT_1 状态，</br>
    接收方收到后转为 CLOSE_WAIT 状态，</br>
    接收方发送相应的 ACK 包，发送方收到后转为 FIN_WAIT_2 状态，</br>
    接收方调用 close(), 发送一个 FIN 包，转为 LAST_ACK 状态，</br>
    发送方收到后转为 TIME_WAIT 状态，等待 2MSL 时间后关闭连接，</br>
    发送方发送相应的 ACK 包，接收方收到后正式关闭连接。

- 流量控制、拥塞控制、重传
- 编程函数
- 报头

## HTTP[S]

- 版本变化：0.9/1.0/1.1/2.0
- get、post
- 证书、密钥
- 一次完整的访问网页的过程
- 报头
- 状态码

## 操作系统

- 进程、线程、协程
- 同步与互斥
- IPC
- 虚拟内存：高端内存、伙伴、slab、vmalloc
- IO 模型
- EPOLL

## 数据库

### MySQL

- SQL语言
- 索引
- MyISAM 与 InnoDB
- 四个原则
- 四个隔离级别
- 两段锁
- MVVC
- 分区、分表、分库

### Redis

## 分布式

- map_reduce
- 负载均衡
- CDN

# 参考
