---
title: 《Effective C++》等阅读笔记
comments: true
toc: true
date: 2019-04-26 14:23:29
categories: C/C++
tags: effective
---

C++ 书籍阅读笔记

<!--more-->

## Effective C++

## Effective Modern C++

### Item 3: 理解 decltype

decltyp e大多数情况下总是不加修改地产出变量和表达式的类型, 对于 T 类型的 lvalue 表达式, decltype 总是产出 T&.

C++14 支持 decltype(auto), 和auto一样, 他从一个初始化器中推导类型, 只不过使用的是 decltype 类型推导规则. 这一开始看起来可能有点矛盾的东西确实完美地结合在一起：auto明确了类型需要被推导, decltype 明确了推导时使用 decltype 推导规则.

在 C++11 中，也许 decltype 最主要的用途就是用来声明 返回值类型依赖于参数类型的 函数模板.

``` c++
Widget w;

cosnt Widget& cw = w;

auto myWidget1 = cw;
//myWidget1的类型是Widget

decltype(auto) myWidget2 = cw
//myWidget2的类型是const Widget&

decltype(auto) f1()
{
    int x = 0;
    ...
    return x;   //decltypex(x)是int，所以f1返回int
}

decltype(auto) f2()
{
    int x = 0;
    ...
    return (x); //decltype((x))是int&，所以f2返回int&
}
//记住f2不仅仅是返回值和f1不同，它还返回了一个局部变量。这是一种把你推向未定义行为的陷阱代码
```

rvalue 不能和 lvalue 引用绑定(除非是 const 左值引用( lvalue-references-to-const ))

### Item 23: 理解 std::move 与 std::forward

std::move 无条件将它的参数转化为 rvalue, 只作转换, 不作 move, 且建议参数为非 const 的 rvalue reference, 这是由于 const 的参数被转换成 const rvalue, 将会被传递给 copy constructor, 实际上并没有 "move". 这样的行为对于保持const 的正确性是必须的. 从一个对象里 move 出一个值通常会改变这个对象, 所以语言不允许将 const 对象传递给像 move constructor 这样的会改变对象的函数.

std::forward 的情况和 std::move 相类似, 但 std::forward 是一个有条件的转换, 它只把用 rvalue 初始化的参数转换成 rvalue.

注意 std::move 只需要一个函数参数, 而 std::forward 却需要一个函数参数以及一个模板类型参数. std::move 和 std::forward 在运行期都没有做任何事情.

## C++ Concurrency In Action

## Essential C++

## More Effective C++

