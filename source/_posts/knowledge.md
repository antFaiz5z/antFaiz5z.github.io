---
title: 琐碎
comments: true
toc: true
date: 2018-02-02 11:47:20
categories: C/C++
tags:  C/C++
---

## shared_ptr

### 析构

shared_ptr中的析构函数中不是直接释放资源，而是调用了dispose函数，如果*pn==0了，才会释放资源。

### 多线程安全性

shared_ptr 本身不是 100%线程安全的。它的引用计数本身是安全且无锁的，但对象的读写则不是，因为shared_ptr有两个数据成员，读写操作不能原子化。根据文档，shared_ptr的线程安全级别和内建类型、标准库容器、string一样，即：

- 一个 shared_ptr 实体可被多个线程同时读取；
- 两个的 shared_ptr 实体可以被两个线程同时写入，“析构”算写操作；
- 如果要从多个线程读写同一个 shared_ptr 对象，那么需要加锁。

### shared_ptr和auto_ptr的区别

- shared_ptr有两个变量，一个记录对象地址，一个记录引用次数
- auto_ptr只有一个变量，用来记录对象地址
<br>
- shared_ptr，可以多个shared_ptr拥有一个资源
- auto_ptr，只能一个auto_ptr拥有一个资源
<br>
- shared_ptr可以实现赋值的正常操作，使得两个地址指向同一资源
- auto_ptr的赋值很奇怪，源失去资源拥有权，目标获取资源拥有权
<br>
- shared_ptr到达作用域时，不一定会释放资源
- auto_ptr到达作用域时，一定会释放资源
<br>
- shared_ptr存在多线程的安全性问题，而auto_ptr没有
<br>
- shared_ptr可用于容器中，而auto_ptr一般不可以用于容器中
<br>
- shared_ptr可以在构造函数、reset函数的时候允许指定删除器，而auto_ptr不能

## const

### const函数

1、如果一个成员函数在逻辑上不会修改对象的状态（字段），就应该定义为const函数;

2、如果对象是const，则它只能调用const成员函数。

如果对象是普通的非const对象：

- 调用的某个成员函数是非const函数，则理所当然调用它。

- 调用的某个成员函数是const函数，则当然也可以调用他。（底层const指针可以指向非常量对象）

- 调用的某个成员函数同时存在 const版本和非const版本，则优先调用非const成员函数，编译器总是使用最匹配的版本。

### const指针

- 顶层const：

指针变量本身是常量。（顶层const不适合引用，因为引用天生就是一个常量，始终引用一个对象，直到消亡）

    如   int* const p = &a;

- 底层const：

变量指向（或者引用）的对象被视为常量。(注意这里的用词：被视为，因为对象不一定就是常量，也可能是底层const指针、引用的一厢情愿)

    如    const int*p ;    int const *p;    这二者写法等价

        const int& r;