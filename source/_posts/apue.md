---
title: 《APUE》阅读笔记
comments: true
toc: true
date: 2019-07-30 14:59:54
categories: Reading
tags: apue
---

《UNIX 环境高级编程(Advanced Programming in the UNIX Environment)》

<!--more-->

## 第7章 进程环境

## 第8章 进程控制

### 进程标识

ID 为 0 的进程通常是调度进程（交换进程）, 是内核的一部分, 不执行任何磁盘上的程序, 因此也被成为系统进程;
ID 为 1 的进程通常是 init 进程, 在自举过程结束时由内核调用, 负责在自举内核后启动一个 UNIX 系统, 负责接管所有孤儿进程, 不会终止, 是一个普通的用户进程, 但以超级用户特权运行.

### fork

子进程获得父进程的 数据空间、堆和栈的副本（拷贝地址空间）, 但不共享这些存储空间部分, 由于 fork 之后常常跟随 exec, 所以很多实现并不执行数据段、堆和栈的完全副本, 而使用 Copy-On-Write 技术.
父子进程共享正文段.

clone(2) 作为 fork 的推广形式, 它允许调用者控制哪些部分由父进程和子进程共享.

在 fork 之后, 父子进程的执行顺序是不确定的, 这取决于内核的调度算法.

write() 是不带缓冲的; 而标准 I/O 库（如 printf）是带缓冲的, 如果标准输出连到终端设备, 则它是行缓冲的, 否则（将标准输出重定向到一个文件时）它是全缓冲的, 这种情况下 fork 之前父进程 printf 的缓冲数据 fork 后将会被拷贝到子进程中, 若 printf 则会又打印出来.
`setbuf(stdout, NUll);`可将标准输出的缓冲设置为空, 使其实时输出.

fork 使得 子进程继承了父进程的：
    所有打开文件描述符;
    父子进程每个相同的打开描述符共享一个文件表项;（因此重定向父进程的标准输出时, 子进程的标准输出也被重定向）
    父子进程共享同一个文件偏移量;（因此一般 1. 父进程等待子进程完成; 2. 父子进程各自执行不同的程序段, 不干扰对方使用的文件描述符）
    实际用户 ID、实际组 ID、有效用户 ID、有效组 ID;
    附属组 ID;
    进程组 ID;
    会话 ID;
    控制终端;
    设置用户 ID 标志和设置组 ID 标志;
    当前工作目录;
    根目录;
    文件模式创建屏蔽字;
    信号屏蔽和安排;
    对任一打开文件描述符的执行关闭（close-on-exec）标志;
    环境;
    连接的共享存储段;
    存储映像;
    资源限制;

父子进程的区别如下：
    fork 的返回值不同;
    进程 ID 不同;
    父进程 ID 不同;
    子进程的 tms_utime、tms_stime、tms_cutime 和 tms_ustime 的值设置为 0;
    子进程不继承父进程设置的文件锁;
    子进程的未处理闹钟被清除;
    子进程的未处理信号集设置为空集;

### vfork

vfork 与 fork 的区别：
    1、vfork 创建的新进程的目的是 exec 一个新程序, 不过在子进程调用 exec 或 exit 之前会在父进程的空间运行;（因此并不将父进程的地址空间完全复制到子进程）
    2、vfork 保证子进程先运行, 在它调用 exec 或 exit 后父进程才可能被调度运行;（如果在调用这两个函数之前子进程依赖父进程的进一步动作则会导致死锁）

### exit

进程有 5 种正常终止和 3 中异常终止方式：
    1、main 函数中执行 return, 相当于对用 exit;
    2、调用 exit;（调用终止处理程序, 关闭所有标准 I/O 流等; 但不处理文件描述符、多进程以及作业控制; 由 ISO C 定义）
    3、调用 _exit 或 _Exit;(在 UNIX 系统中两种调用等价, 并不冲洗标准 I/O 流; 处理 UNIX 系统特定的细节; exit 调用 _exit; 由 POSIX.1 说明)
    4、在进程的最后一个线程在其启动例程中执行 return;（进程以终止状态 0 返回）
    5、进程的最后一个线程调用 pthread_exit 函数;
    1、调用 abort;（产生 SIGABRT 信号, 是下一种异常终止的一种特例）
    2、进程接收到某些信号;（信号可由进程自身（调用 abort）、其他进程或内核产生）
    3、最后一个线程对取消请求作出响应;

不论进程如何终止, 最后都会执行内核中的同一段代码, 即关闭描述符、释放存储器.
子进程正常或异常终止时, 内核向其父进程发送 SIGCHLD 信号.

父进程在子进程前终止：在父进程终止的时候, 内核逐个检查所有活动进程, 若是子进程则将其父进程改为 init 进程.
子进程在父进程前终止：子进程需要告知父进程其如何终止（1. 将退出状态（exit status）作为参数传递给 exit、_exit、_Exit; 2. 异常终止时内核而不是进程本身产生一个指示其异常终止原因的终止状态（termination status））,父进程使用 wait 或 waitpid 取得内核为每个终止子进程保存的一定量信息, 这些信息至少包括 进程 ID、进程终止状态、进程使用的 CPU 时间总量.

一个已经终止而其父进程还未对其获取信息及释放资源的进程称为僵死进程.（ps(1) 打印其状态为 Z）

为防止系统中塞满僵死进程, init 进程设计成只要有子进程终止就会调用 wait 获取其终止状态. 如果一个进程 fork 一个子进程, 但不想父进程等待子进程终止, 也不想子进程处于僵死状态直到父进程终止, 方法是调用 fork 两次, 即通过让子进程终止使子子进程被 init 进程接管.

### wait 和 waitpid

waitpid 可等待一个特定的进程, 而 wait 返回任一终止进程的状态;
waitpid 提供了一个 wait 的非阻塞版本;
waitpid 通过 WUNTRACED 和 WCONTINUED 选项支持作业控制.

历史遗留：waitid()、wait3()、wait4().

## 第10章 信号

## 第11章 线程

### 线程概念

多线程作用：
1、通过为每种事件类型分配单独的处理线程, 可以简化处理异步事件的代码;
2、多个进程必须使用操作系统提供的复杂机制才能实现内存和文件描述符放入共享, 而多个线程自动地可以访问相同的存储地址空间和文件描述符;
3、有些问题可以分解从而提高整个程序的吞吐量;
4、交互的程序可以通过多线程改善响应时间;

即使程序运行在单核处理器上, 仍然可以享受多线程模型的好处, 可以改善响应时间和吞吐量.

### 线程标示

进程 ID (pid_t) 在整个系统中是唯一的, 而线程 ID (pthread_t) 只有在它所属的进程上下文中才有意义.

### 线程创建

### 线程终止

进程中任意线程调用 exit、_Exit 或者 _exit, 那么整个进程就会终止.

单个线程可以通过 3 中方式退出：
1、简单地从启动例程中返回, 返回值是线程的退出码 (return);
2、被同一进程中的其他线程取消 (pthread_cancel);
3、线程调用 pthread_exit (pthread_exit);

线程通过调用 return 返回或者调用 pthread_exit 退出, 进程中的其他线程可以通过调用 pthread_join 获得该线程的退出状态. 默认情况下, 线程的终止状态会保存直到对该线程调用 pthread_join.

pthread_exit 和 pthread_create 的 void* 参数可以是一个简单数值, 也可以是一个复杂结构体. 只是这个结构所使用的的内存在调用者完成调用以后必须是有效的, 比如 pthread_join 会试图使用该结构. 因此可以使用全局结构或用 malloc 分配堆空间.

pthread_cancel 并不等待线程终止, 它仅仅提出请求, 对方线程可以选择忽略取消或者控制如何被取消.

使用 pthread_cleanup_push 注册的线程清理函数 rtn 只有当线程采取以下动作时才会被调用：
1、调用 pthread_exit 时;
2、相应取消请求时;
3、用非零参数调用 pthread_cleanup_pop 时;

而简单使用 return 返回时不会执行线程清理程序.

并且线程处理函数注册在栈中, 因此执行顺序与它们注册时相反.

### 线程同步

待更新

| 同步原语 | 进程 | 线程 POSIX| 线程 STL |
|------|-----|-----|-----|
|获取 ID|getpid|pthread_self|std::this_thread::get_id()|
|创建|fork|pthread_create</br>pthread_detach</br>pthread_join</br>|std::thread和其成员函数|
|终止|exit|pthread_exit||
|非正常终止|abort|pthread_cancel||
|终止时清理|atexit|pthread_cleanup_push</br>pthread_cleanup_pop||
|互斥量|N/A|pthread_mutex_init</br>pthread_mutex_destroy</br>pthread_mutex_lock</br>pthread_mutex_trylock</br>pthread_mutex_unlock</br>pthread_mutex_timedlock|std::mutex类</br>std::unique_lock<>模板</br>std::lock_guard<>模板|
|条件变量|N/A|pthread_cond_init</br>pthread_cond_destroy</br>pthread_cond_wait</br>pthread_cond_timedwait</br>pthread_cond_signal</br>pthread_cond_broadcast|std::condition_variable</br>std::condition_variable_any|
|读写锁|N/A|pthread_rwlock_init</br>pthread_rwlock_destroy</br>pthread_rwlock_rdlock</br>pthread_rwlock_wrlock</br>pthread_rwlock_tryrdlock</br>pthread_rwlock_trywrlock</br>pthread_rwlock_unlock</br>pthread_rwlock_timedrdlock</br>pthread_rwlock_timedwrlock||
|自旋锁|N/A|pthread_spin_init</br>pthread_spin_destroy</br>pthread_spin_lock</br>pthread_spin_trylock</br>pthread_spin_unlock</br>||
|屏障|N/A|pthread_barrier_init</br>pthread_barrier_destroy</br>pthread_barrier_wait||

#### 互斥量

如果线程试图对同一个互斥量加锁两次, 那么它自身就会陷入死锁状态.

如果锁的粒度太粗, 就会出现很多线程阻塞等待相同的锁, 这可能并不能改善并发性;
如果锁的粒度太细, 那么过多的锁开销就会使系统性能受到影响;
程序员需要在代码复杂性和性能之间找到正确的平衡.

pthread_mutex_t 静态初始化：PTHREAD_MUTEX_INITIALIZER
pthread_mutex_t 动态初始化：pthread_mutex_init/pthread_mutex_destroy
(条件变量 pthread_cond_t、
读写锁 pthread_rwlock_t、
自旋锁 pthread_spinlock_t、
屏障 pthread_barrier_t 类似)

如果动态分配互斥量（如通过调用 malloc 函数）, 在释放内存前需要调用 pthread_mutex_destroy.
（条件变量 pthread_cond_init/destroy、
读写锁 pthread_rwlock_init/destroy、
自旋锁 pthread_spin_init/destroy、
屏障 pthread_barrier_init/destroy 类似）

pthread_mutex_trylock 不阻塞直接返回：加锁成功返回 0, 失败返回 EBUSY. 用来避免线程阻塞.
pthread_mutex_timedlock 指定线程愿意阻塞等待的绝对时间, 若超时未加锁成功则返回 ETIMEOUT. 用来避免线程永久阻塞.
（条件变量 pthread_cond_timedwait、
读写锁 pthread_rwlock_<try/timed><rdlock/wrlock>、
自旋锁 pthread_spin_trylock 类似）

#### 条件变量

条件变量本身是由互斥量保护的, 线程在改变条件状态之前必须先锁住互斥量, 其他线程在得到互斥量之前不会察觉到这种改变.

传递给 pthread_cond_wait 的互斥量对条件进行保护. 调用者把锁住的互斥量传给函数, 函数然后自动把线程放到等待条件的线程列表上, 对互斥量解锁. 这就关闭了条件检查和线程进入休眠状态等待条件改变这两个操作之间的时间通道, 于是线程不会错过条件的任何变化. pthread_cond_wait 返回时互斥量再次被锁住.

pthread_cond_timedwait 传入的绝对时间参数为 timespec 结构, 可以使用 clock_gettime 获取 timespec 结构表示的当前时间, 或者使用 gettimeofday 获取 timeval 结构表示的当前时间再转换成 timespec 结构.

从 pthred_cond_[timed]wait 调用成功返回时, 线程需要重新计算条件, 因为另一个线程可能已经在运行并改变了条件.

#### 读写锁

读写锁非常适合于对数据结构读的次数远大于写的情况. 与互斥量相比, 读写锁在使用之前必须初始化, 在释放它们底层的内存之前必须销毁.

有的实现可能会对共享模式（读模式）下可获取的读锁的次数进行限制（避免读锁长期占用而使写锁请求得不到满足）, 所以需要检查 pthread_rwlock_rdlock 的返回值.

#### 自旋锁

应用场景：锁的持有时间短, 而且线程不希望在重新调度上花费太多的成本.

当线程等待锁变为可用时, CPU 不能做其他的事情, 这也是自旋锁只能够被持有一小段时间的原因. 事实上, 某些自旋锁的实现在试图获得互斥量的时候会自旋一小段时间, 在自旋计数达到某一阈值的时候才会休眠.

pthread_spin_lock 和 pthread_spin_trylock 对自旋锁加锁, 前者在获取之前一直自旋, 后者不能自旋.

#### 屏障

对某个线程来说, pthread_barrier_wait 的返回值为 PTHREAD_BARRIER_SERIAL_THREAD, 剩下的线程看到的返回值为 0. 这使得一个线程可以作为主线程, 它可以工作在其他所有线程已完成的工作结果上.

一旦达到屏障计数值, 而且线程处于非阻塞状态, 屏障就可以重用. 但是除非连续调用  pthread_barrier_destroy 和 pthread_barrier_init 后, 否则屏障计数不会改变.

## 第15章 进程间通信

管道（匿名、FIFO）、消息队列、共享内存、信号量、套接字、信号

总结：

管道： 单向传输（半双工）, 效率低下（a 传输给 b, 只有等 b 取数据后 a 才能返回）, 不适合频繁通信，优点是简单.

消息队列： 可以解决管道效率低下的问题, 但是仍不适合发送数据多且频繁的场景, 因为拷贝开销过大.

共享内存： 可以解决消息队列拷贝时间的问题, 通过各自一段虚拟地址映射至同一段物理内存.

信号量： 可以解决共享内存多进程竞争问题, 本质是一个计数器

[system-v-ipc-vs-posix-ipc](https://stackoverflow.com/questions/4582968/system-v-ipc-vs-posix-ipc)

|   |System V|     |       |POSIX|     |       |
|----|--------|-----|-------|-----|-----|-------|
||消息队列|信号量|共享内存|消息队列|信号量|共享内存|
|头文件|<sys/msg.h>|<sys/sem.h>|<sys/shm.h>|<mqueue.h>|<semaphore.h>|<sys/mman.h>|
|创建或打开IPC的函数</br>(删除IPC的函数)|msgget|semget|shmget|mq_open</br>mq_close</br>mq_unlink</br>|sem_open</br>sem_close</br>sem_unlink</br>sem_init</br>sem_destroy|shm_open</br>shm_unlink</br>|
|控制IPC操作的函数|msgctl|semctl|shmctl|mq_getattr</br>mq_setattr||ftruncate</br>fstat|
|IPC操作函数|msgsnd</br>msgrcv|semop|shmat</br>shmdt|mq_send</br>mq_receive</br>mq_notify|sem_wait</br>sem_trywait</br>sem_timedwait</br>sem_post</br>sem_getvalue|mmap</br>munmap|

### 管道

历史上是半双工的, 即数据只在一个方向上流动, 现在某些系统提供全双工管道.

固定 fd[0] 为读端, fd[1] 为写端.

常量 PIPE_BUF 规定了内核的管道缓冲区大小, 如果多个进程对同一管道调用 write, 而且要求写的字节数超过 PIPE_BUF, 那么各个进程所写的数据可能会相互交叉.

``` c
#include <unistd.h>
int pipe(int fd[2]);
```

### FIFO

FIFO 是一种文件类型, 使用 open 打开时, 可以指定非阻塞标志（O_NONBLOCK）.

应用程序可以使用 mknod 和 mknodat 函数创建 FIFO（mknod 现已添加在 POSIX.1 的 XSI 扩展中）.

``` c
#include <sys/stat.h>
int mkfifo(const char *path,  mode_t mode);
int mkfifoat(int fd, const char *path, mode_t mode);
```

### POSIX 信号量

POSIX 信号量接口解决了 XSI 信号量接口的几个缺陷：
1、更高性能
2、接口使用更简单：没有信号量集
3、在删除时表现更完美：XSI 信号量被删除时使用这个信号量标识符的操作会失败, 并将 errno 置为 EIDRM; 而使用 POSIX 信号量时, 操作能继续正常工作到该信号量的最后一次引用被释放.

POSIX 信号量包括 命名的和未命名的.

## 第16章 网络 IPC：套接字

### 套接字描述符

套接字描述符在 UNIX 系统中被当做是一种文件描述符, 但不是所有参数为文件描述符的函数都可以接受套接字描述符.

### 字节序

Linux 3.2.0 - 小端
Mac OSX 10.6.8 - 小端
Solaris 10 - 大端
TCP/IP 协议栈 - 大端

### 建立连接

`int bind(int sockfd, struct sockaddr *addr, socklen_t len);`

一般只能将一个套接字端点绑定到一个给定地址上, 尽管有些协议支持多重绑定.

`int connect(int sockfd, const struct sockaddr *addr, socklen_t len);`

使用 UDP (无连接的套接字)同样可以调用 connect, 表示只一对一发送, 如此可以使用 send.

`int listen(int sockfd, int backlog);`

参数 backlog 提供了一个提示, 提示系统该进程所要入队的未完成连接请求数量（对于 TCP, 其默认值为 128）, 一旦队列满系统就会拒绝多余的连接请求.

`int accept(int sockfd, struct sockaddr *restrict addr, socklen_t *restrict len);`

如果没有连接请求在等待, accept 会阻塞直到一个请求到来. 如果 sockfd 处于非阻塞模式, accept 会返回 -1, 并将 errno 设置为 EAGAIN 或 EWOULDBLOCK.
可以使用 poll 或 select 来等待一个请求的到来, 此时一个带有等待连接请求的套接字会以可读的方式出现.

### 数据传输

`ssize_t send(int sockfd, cosnt void *buf, size_t nbytes, int flags);`

send 类似于 write, 但可以通过指定标志来改变处理传输数据的方式.
使用 send 时套接字必须已经连接.
即使 send 成功返回, 也并不表示连接的另一端的进程就一定接受了数据, 只能保证无误地发送到了网络驱动程序上.
对于支持报文边界的协议, 如果尝试发送的单个报文的长度超过协议所支持的最大长度, 那么 send 会失败, 并将 errno 设为 EMSGSIZE;
对于字节流协议, send 会阻塞直到整个数据传输完成.

`ssize_t sendto(int sockfd, cosnt void *buf, size_t nbytes, int flags, const struct sockaddr *destaddr, socklen_t destlen);`

与 send 相比, 区别在于 sendto 可以在无连接的套接字上指定一个目标地址.

`ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);`

使用 msghdr 结构来指定多重缓冲区传输数据, 类似于 writev.

`ssize_t recv(int sockfd, cosnt void *buf, size_t nbytes, int flags);`
`ssize_t recvfrom(int sockfd, cosnt void *buf, size_t nbytes, int flags, const struct sockaddr *destaddr, socklen_t destlen);`
`ssize_t recvmsg(int sockfd, const struct msghdr *msg, int flags);`

类似于 readv.

### 带外数据

TCP 支持带外数据, UDP 不支持.

TCP 将带外数据称为紧急数据, 仅支持一个字节的紧急数据, 但是允许紧急数据在普通数据传递机制数据流之外传输.
为了产生紧急数据, 可以在 3 个 send 函数里指定 MSG_OOB 标志.
如果带 MSG_OOB 标志发送的数据超过一个时, 最后一个字节被视为紧急数据.
如果在接受当前的紧急数据字节之前又有新的紧急数据到来, 那么已有的字节会被丢弃.

### 非阻塞与异步 I/O

通常, 当套接字输出队列没有足够空间来发送消息时, send 函数会阻塞; recv 函数没有数据可用时会阻塞等待. 而在套接字非阻塞模式下, 这些函数不会阻塞而是失败, 将 errno 设为 EAGAIN 或 EWOULDBLOCK, 一般可以使用 poll 或 select 来判断能否接受或传输数据.

一些文献把经典的基于套接字的异步 I/O 机制称为 “基于信号的 I/O”, 区别于 Single UNIX Specification 中的通用异步 I/O 机制.
