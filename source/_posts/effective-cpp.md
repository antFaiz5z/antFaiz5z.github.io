---
title: C++ 相关书籍阅读笔记
comments: true
toc: true
date: 2019-04-26 14:23:29
categories: C/C++
tags: effective
---

《Effective C++》、《Effective Modern C++》、《C++ Concurrency In Action》、《Essential C++》、《More Effective C++》等.

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

rvalue 不能和 lvalue 引用绑定 (除非是 const 左值引用( lvalue-references-to-const ))

### Item 18-22: 智能指针

auto_ptr 和 unique_ptr 采用的是 ownership (建立所有权) 概念, 对于特定对象, 只能被一个智能指针所拥有, 这样, 只有拥有该对象的智能指针的析构函数才会删除该对象, 然后要注意的是, 赋值操作会转让操作权.
虽然 auto_ptr 和 unique_ptr 都采用该策略, 但是 unique_ptr 的策略更严格. 当出现上述情况时, 程序会编译出错 (这是因为 unique_ptr 内部将拷贝构造函数给 delete 了, 但赋值是允许的), 而 auto_ptr 则会在执行阶段 core dumped.
shared_ptr 则采用 reference counting (引用计数) 的策略, 例如, 赋值时, 计数 +1, 指针过期时, 计数 -1. 只有当计数为 0 时, 即最后一个指针过期时, 才会被析构掉.

要安全的重用 unique_ptr 指针, 可给它赋新值. C++为其提供了 std::move() 方法. 而 auto_ptr 由于策略没有 unique_ptr 严格, 无需使用move方法. 由于 unique_ptr 使用了 C++11 新增的移动构造函数和右值引用, 所以可以区分安全和不安全的用法.

auto_ptr 和 shared_ptr 可以和 new 一起使用, 但不可以和 new[] 一起使用, 但是 unique_ptr 可以和 new[] 一起使用.

weak_ptr 是为配合 shared_ptr 而引入的一种智能指针来协助 shared_ptr 工作, 它可以从一个 shared_ptr 或另一个 weak_ptr 对象构造, 它的构造和析构不会引起引用记数的增加或减少, 没有重载 * 和 -> 但可以使用 lock() 获得一个可用的 shared_ptr 对象.
weak_ptr 的一个重要用途是通过 lock() 获得 this 指针的 shared_ptr, 使对象自己能够生产 shared_ptr 来管理自己, 但助手类 enable_shared_from_this 的 shared_from_this 会返回 this 的 shared_ptr, 只需要让想被 shared_ptr 管理的类从它继承即可.

当多个对象指向同一个对象的指针时, 应选择 shared_ptr.
用 new 申请的内存, 返回指向这块内存的指针时, 选择 unique_ptr 就不错.
unique_ptr 为右值(不准确的说类似无法寻址)时, 可以赋给shared_ptr.

### Item 23: 理解 std::move 与 std::forward

std::move 无条件将它的参数转化为 rvalue, 只作转换, 不作 move, 且建议参数为非 const 的 rvalue reference, 这是由于 const 的参数被转换成 const rvalue, 将会被传递给 copy constructor, 实际上并没有 "move". 这样的行为对于保持const 的正确性是必须的. 从一个对象里 move 出一个值通常会改变这个对象, 所以语言不允许将 const 对象传递给像 move constructor 这样的会改变对象的函数.

std::forward 的情况和 std::move 相类似, 但 std::forward 是一个有条件的转换, 它只把用 rvalue 初始化的参数转换成 rvalue.

注意 std::move 只需要一个函数参数, 而 std::forward 却需要一个函数参数以及一个模板类型参数. std::move 和 std::forward 在运行期都没有做任何事情.

## C++ Concurrency In Action

## Essential C++

## More Effective C++
