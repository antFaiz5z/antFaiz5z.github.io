---
title: C++ 相关书籍阅读笔记
comments: true
toc: true
date: 2019-04-26 14:23:29
categories: Reading
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

### 线程池

join_threads类的实例(来自于第8章)③用来汇聚所有线程。当然也需要析构函数：仅设置done标志⑪，并且join_threads确保所有线程在线程池销毁前全部执行完成。注意成员声明的顺序很重要：done标志和worker_queue必须在threads数组之前声明，而数据必须在joiner前声明。这就能确保成员能以正确的顺序销毁；比如，所有线程都停止运行时，队列就可以安全的销毁了。

### 附录 A

#### 右值引用

左值：有明确的内存地址，在栈上或堆上分配的命名对象，或者其他对象成员。
右值：文字常量和临时变量，不具名变量。

左值引用只能被绑定在左值上，常左值引用可以被绑定在右值上。
右值引用可以被绑定<del>在左值（作为左值引用）？</del>或右值上。

补充，非原书：

    规则1（引用折叠规则）：如果间接的创建一个引用的引用，则这些引用就会“折叠”。在所有情况下（除了一个例外），引用折叠成一个普通的左值引用类型。一种特殊情况下，引用会折叠成右值引用，即右值引用的右值引用， T&& &&。即
    X& &、X& &&、X&& &都折叠成X&；
    X&& &&折叠为X&&
    （T&& 不一定是右值引用，也可以是引用的引用。）
    规则2（右值引用的特殊类型推断规则）：当将一个左值传递给一个参数是右值引用的函数，且此右值引用指向模板类型参数(T&&)时，编译器推断模板参数类型为实参的左值引用。
    从上述两个规则可以得出结论：如果一个函数形参是一个指向模板类型的右值引用，则该参数可以被绑定到一个左值上（T&& 绑定左值，推导为左值引用）。即在使用右值引用作为函数模板的参数时，与之前的用法有些不同：如果函数模板参数以右值引用作为一个模板参数，当对应位置提供左值的时候，模板会自动将其类型认定为左值引用；当提供右值的时候，会当做普通数据使用。
    规则3：虽然不能隐式的将一个左值转换为右值引用，但是可以通过static_cast显示地将一个左值转换为一个右值。【C++11中为static_cast新增的转换功能】。

    move() 是这样一个函数，它接受一个参数(通常是右值引用或左值)，然后返回一个该参数对应的右值引用。
    move() 执行到右值的无条件转换。就其本身而言，它没有move任何东西。等价于 static_cast<T&&>().
    forward() 函数的作用：它接受一个参数，然后返回该参数本来所对应的类型的引用。
    完美转发实现了参数在传递过程中保持其值属性的功能，即若是左值，则传递之后仍然是左值，若是右值，则传递之后仍然是右值。
    std::move和std::forward本质都是转换。std::move执行到右值的无条件转换。std::forward只有在它的参数绑定到一个右值上的时候，才转换它的参数到一个右值。
    td::move没有move任何东西，std::forward没有转发任何东西。在运行期，它们没有做任何事情。它们没有产生需要执行的代码。

``` cpp
template<class T>
typename remove_reference<T>::type&& move(T&& a)
{
  typedef typename remove_reference<T>::type&& RvalRef;
  return static_cast<RvalRef>(a);
}

template<class T>
T&& forward(typename remove_reference<T>::type& arg)
{
   return static_cast<T&&>(arg);
}
``` 

默认函数包括：构造函数、拷贝构造函数、拷贝赋值操作符、移动构造函数、移动赋值操作符。

``` cpp
template <typename T>
class X{
    T data;
public:
    X();
    X(const X&);
    X& operator=(const X&);
    X(X&& other): data(std::move(other.data)){}
    X& operator=(X&& other){
        data = std::move(other.data);
        return *this;
    }
};
```

#### lambda 函数

``` cpp
[]{ // lambda表达式以[]开始
do_stuff();
do_more_stuff();
}(); // 表达式结束，可以直接调用

//当lambda函数体中有多个return语句，就需要显式的指定返回类型。只有一个返回语句的时候，也可以这样做。
cond.wait(lk,[]()->bool{return data_ready;});

//使用 [=] 捕获本地变量
//使用 [&] 对本地变量进行引用
//使用 [this] 访问类中成员
std::function<int()> f=[=](int p){return i+p;}
std::function<int()> f=[=,&j,&k]{return i+j+k;};
std::function<int()> f=[&,j,k]{return i+j+k;};
std::function<int()> f=[&i,j,&k]{return i+j+k;};
```

并发的上下文中，lambda是很有用的，其可以作为谓词放
在 std::condition_variable::wait() (见4.1.1节)和 std::packaged_task<> (见4.2.1节)中；或是用在线程池中，对小任务进行打包。也可以线程函数的方式std::thread 的构造函数(见2.1.1)，以及作为一个并行算法实现，在parallel_for_each()(见8.5.1节)中使用。

## Essential C++

## More Effective C++
