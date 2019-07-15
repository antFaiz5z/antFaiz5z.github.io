---
title: 《muduo》阅读笔记
comments: true
toc: true
date: 2019-06-28 17:05:28
categories:
tags:
---

原书《Linux 多线程服务端编程：使用 muduo C++ 网络库》.

<!--more-->

## 第三章 多线程服务器的适用场合与常用编程模型

每个进程有自己独立的地址空间, 线程的特点是共享地址空间, 从而高效地共享数据. 多线程的价值在于更好地发挥多核处理器的效能.

"Reactor" 模型 = "non-blocking IO" + "IO multiplexing"

Reactor 模型的优点：
编程不难、效率不错;
不仅可以用于读写 socket、连接的建立, 甚至 DNS 解析都可以用非阻塞方式进行, 以提高并发度和吞吐量, 对于 IO 密集型应用是个不错的选择.

基于事件驱动的编程模型的缺点：
它要求事件回调函数必须是非阻塞的, 对于涉及网络 IO 的请求响应式协议, 它容易割裂业务逻辑, 使其散布于多个回调函数之中, 相对不容易理解与维护.

实用方案：
1、thread-per-connection
2、单线程 reactor
3、reactor + 线程池
4、one loop per thread（reactors）
5、one loop per thread（reactors）+ 线程池

方案间的区别在于 接受新连接的线程、网络 IO 的线程、计算任务的线程 是否为同一个线程. 计算任务的线程基本对应于线程池.
muduo 直接支持后四种.

thread per connection 不适合高并发场景, 其伸缩性不佳.
one loop per thread 的并发度足够大, 且与 CPU 核数成正比.
基于 IO multiplexing 的 一个 event loop 的并发连接数可以轻松达到几千乃至上万.

one loop（或者说 reactor） per thread 优点：
1、线程数目基本固定, 可以在程序启动的时候设置, 不会频繁创建与销毁;
2、可以很方便地在线程间调配负载;
3、IO 事件发生的线程是固定的, 同一个 TCP 连接不必考虑事件并发.

必须使用单线程的场合：
1、程序可能会 fork(). 多线程程序不是不能调用 fork(), 而是这么做会遇到很多麻烦
2、限制程序的 CPU 占用率.

无论是 IO 密集型或是 CPU 密集型的服务,  多线程都没有什么绝对意义上的性能优势. 意思是, 如果用很少的 CPU 负载就能让 IO 跑满, 或者用很少的 IO 流量就能让 CPU 跑满, 那么多线程没什么用处.
虽然多线程不能提高绝对性能, 但能提高平均响应性能.

多线程适用的场景：
提高响应速度, 让 IO 和 "计算" 相互重叠, 降低延迟.

一般多线程程序中的线程主要有三类：IO 线程、计算线程、第三方库所使用的线程如 logging、database connection.

一个多线程的进程和多个单线程的进程的取舍：
如果工作集较大, 那么就用多线程, 避免 CPU cache 换入换出, 影响性能;
否则就用单线程进程, 享受单线程编程的便利.

## 第九章 分布式工程实践

将"随时能重启进程"作为程序设计目标, 要求程序只使用操作系统能自动回收的 IPC, 不使用生命期大于进程的 IPC, 也不使用无法重建的 IPC. 具体来说, 只使用 TCP 作为进程间通信的唯一手段.

### 心跳协议的设计

应用层心跳必不可少：
1、操作系统崩溃导致机器重启, 没有机会发送 FIN 分节
2、服务器硬件故障导致机器重启, 也没有机会发送 FIN 分节
3、并发连接数很高时, 操作系统或进程如果重启, 可能没有机会断开全部连接, 即 FIN 分节可能出现丢包, 但没有机会重试
4、网络故障, 连接双方能得知这一情况的唯一方案是检测心跳超时

为什么 TCP keepalive 不能代替应用层心跳?
心跳除了说明进程还在、网络通畅, 更重要的是表明应用程还能正常工作;
而 TCP keepalive 由操作系统负责探查, 即使进程死锁或阻塞, 操作系统也会如常收发 TCP keepalive 消息, 对方无法得知这一异常.

进程 C 依赖于 S, 则一般 S 为 sender, C 为 receiver.

心跳设计关键：（高置信度与低反应时间不可兼得）
1、通常 sender 的发送周期和 receiver 的检查周期相同, 为 T; timeout 时间一般为 2T
2、T 越小, 开销越大; T 越大, receiver 检测到故障的延迟也就越大
3、心跳信息应该包含`发送方的标识符`; 建议也包含`当前负载`, 便于客户端做负载均衡
4、如果 sender 和 receiver 之间有其他消息中转进程, 那么还应该加上 sender 的`发送时间`, 防止消息在传输过程中堆积而导致假心跳
5、要在工作线程中发送, 不要单独起一个"心跳线程", 防止伪心跳（防止工作线程死锁或阻塞时还在发送心跳）
6、与业务信息用同一个连接, 不要单独用"心跳连接", 防止伪心跳（如果验证的不是收发业务数据的 TCP 连接畅通则意义不大）

### 进程标识

以四元组 ip:port:start_time:pid 作为分布式系统中的 gpid.
进一步,还可以把程序的名称和版本号作为 gpid 的一部分, 起到锦上添花的作用.
如此生成全局唯一的消息 id 字符串也十分简单, 只要在进程内使用一个原子计数器, 用计数器递增的值和 gpid 即可组成每个消息的全局唯一 id.