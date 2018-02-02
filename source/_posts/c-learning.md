---
title: C++学习
comments: true
toc: true
date: 2018-02-02 11:47:20
categories: C++
tags:  C++
---


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