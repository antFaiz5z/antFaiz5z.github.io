---
title: 网络库、Socket、IO、多线程 知识点备忘
comments: true
toc: true
date: 2018-10-25 11:36:40
categories: C/C++
tags: 
    - networking
    - io
    - multi-thread
---

## IO模型

- 同步 I/O：将数据从内核缓冲区复制到应用进程缓冲区的阶段，应用进程会阻塞。

- 异步 I/O：不会阻塞。

阻塞式 I/O、非阻塞式 I/O、I/O 复用和信号驱动 I/O 都是同步 I/O，它们的主要区别在第一个阶段。

非阻塞式 I/O 、信号驱动 I/O 和异步 I/O 在第一阶段不会阻塞。

![五大 I/O 模型比较](/images/io_models.png)

## 多路复用IO

select/poll/epoll 都是 I/O 多路复用的具体实现，select 出现的最早，之后是 poll，再是 epoll。

### 功能

select 和 poll 的功能基本相同，不过在一些实现细节上有所不同。

select 会修改描述符，而 poll 不会；
select 的描述符类型使用数组实现，FD_SETSIZE 大小默认为 1024，因此默认只能监听 1024 个描述符。如果要监听更多描述符的话，需要修改 FD_SETSIZE 之后重新编译；而 poll 的描述符类型使用链表实现，没有描述符数量的限制；
poll 提供了更多的事件类型，并且对描述符的重复利用上比 select 高。
如果一个线程对某个描述符调用了 select 或者 poll，另一个线程关闭了该描述符，会导致调用结果不确定。

### 速度

select 和 poll 速度都比较慢。

select 和 poll 每次调用都需要将全部描述符从应用进程缓冲区复制到内核缓冲区。

select 和 poll 的返回结果中没有声明哪些描述符已经准备好，所以如果返回值大于 0 时，应用进程都需要使用轮询的方式来找到 I/O 完成的描述符。

epoll是select/poll的增强版本，其实现和使用方式与select/poll有很多不同，epoll通过一组函数来完成有关任务，而不是一个函数。

epoll之所以高效，是因为epoll将用户关心的文件描述符放到内核里的一个事件表中，而不是像select/poll每次调用都需要重复传入文件描述符集或事件集。比如当一个事件发生（比如说读事件），epoll无须遍历整个被侦听的描述符集，只要遍历那些被内核IO事件异步唤醒而加入就绪队列的描述符集合就行了。

### 可移植性

几乎所有的系统都支持 select，但是只有比较新的系统支持 poll。

### 应用场景

很容易产生一种错觉认为只要用 epoll 就可以了，select 和 poll 都已经过时了，其实它们都有各自的使用场景。

1. select 应用场景

select 的 timeout 参数精度为 1ns，而 poll 和 epoll 为 1ms，因此 select 更加适用于实时性要求比较高的场景，比如核反应堆的控制。

select 可移植性更好，几乎被所有主流平台所支持。

2. poll 应用场景

poll 没有最大描述符数量的限制，如果平台支持并且对实时性要求不高，应该使用 poll 而不是 select。

3. epoll 应用场景

只需要运行在 Linux 平台上，有大量的描述符需要同时轮询，并且这些连接最好是长连接。

需要同时监控小于 1000 个描述符，就没有必要使用 epoll，因为这个应用场景下并不能体现 epoll 的优势。

需要监控的描述符状态变化多，而且都是非常短暂的，也没有必要使用 epoll。因为 epoll 中的所有描述符都存储在内核中，造成每次需要对描述符的状态改变都需要通过 epoll_ctl() 进行系统调用，频繁系统调用降低效率。并且 epoll 的描述符存储在内核，不容易调试。

## epoll

epoll_ctl() 用于向内核注册新的描述符或者是改变某个文件描述符的状态。已注册的描述符在内核中会被维护在一棵红黑树上，通过回调函数内核会将 I/O 准备好的描述符加入到一个链表中管理，进程调用 epoll_wait() 便可以得到事件完成的描述符。

从上面的描述可以看出，epoll 只需要将描述符从进程缓冲区向内核缓冲区拷贝一次，并且进程不需要通过轮询来获得事件完成的描述符。

epoll 仅适用于 Linux OS。

epoll 比 select 和 poll 更加灵活而且没有描述符数量限制。

epoll 对多线程编程更有友好，一个线程调用了 epoll_wait() 另一个线程关闭了同一个描述符也不会产生像 select 和 poll 的不确定情况。

### 工作模式

epoll 的描述符事件有两种触发模式：LT（level trigger）和 ET（edge trigger）。LT是select/poll使用的触发方式，比较低效；而ET是epoll的高速工作方式。

1. LT 模式

当 epoll_wait() 检测到描述符事件到达时，将此事件通知进程，进程可以不立即处理该事件，下次调用 epoll_wait() 会再次通知进程。是默认的一种模式，并且同时支持 Blocking 和 No-Blocking。

2. ET 模式

和 LT 模式不同的是，通知之后进程必须立即处理事件，下次再调用 epoll_wait() 时不会再得到事件到达的通知。

很大程度上减少了 epoll 事件被重复触发的次数，因此效率要比 LT 模式高。只支持 No-Blocking，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

通俗理解（是挺俗的）就是，比如说有一堆女孩，有的很漂亮，有的很凤姐。现在你想找漂亮的女孩聊天，LT就是你需要把这一堆女孩全都看一遍，才可以找到其中的漂亮的（就绪事件）；而ET是你的小弟（内核）将N个漂亮的女孩编号告诉你，你直接去看就好，所以epoll很高效。另外，采用非阻塞方式，小明还需要每隔十分钟回来看一下（select）；如果小明有小弟（内核）帮他守在大门口，女神回来了，小弟会主动打电话，告诉小明女神回来了，快来处理吧！这就是epoll。

## 编程模型

1. 单线程服务器的常用编程模型：Reactor模式和Proactor模式

Reactor模式，即“non-blocking IO + IO multiplexing"，程序的基本结构是一个事件循环（event loop），以事件驱动（event-driven）和事件回调的方式实现业务逻辑。其中 IO multiplexing 指的是IO线程的复用，不是IO的复用，具体的话是用select(2)和poll(2)，epoll(4)来监听感兴趣的fd。但是这个编程模型也有一个本质的缺点，它要求回调函数必须是非阻塞的。

2. 多线程服务器的常用编程模型：non-blocking IO+ one loop per thread模式

one loop per thread：每个IO线程有一个event loop（或者叫Reactor）用作IO multiplexing，配合non-blocking IO和定时器用于处理读写和定时事件。

thread pool用作计算，可以使用生产者消费者队列。

也就是可以有多个IO线程或单个IO线程，每个IO线程最多有一个event loop，使用线程池作为计算线程，每个计算线程使用blocking queue缓存代办任务。

在non-blocking网络编程中应用层实现并使用buffer是必需的，buffer使IO线程只阻塞于IO multiplexing函数上，如select/poll/epoll_wait，而不阻塞read()或者write()

## Tips

1. RAII(资源获取即初始化)，这是C++中用来管理资源的一种方式，即通过栈空间上执行对象的构造和析构实现对资源的获取和释放。书中例子也用到了这种方法来实现互斥锁的加锁解锁，条件变量的使用。

2. 善用shared_ptr实现线程安全的对象释放，但是要注意再多个线程读写同一个shared_ptr对象时，需要加锁。
