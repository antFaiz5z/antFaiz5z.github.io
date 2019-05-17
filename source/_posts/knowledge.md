---
title: (Always-Update) 琐碎
comments: true
toc: true
date: 2018-02-02 11:47:20
categories: C/C++
tags: always-update
---

- 对于 STL 的 map 容器，使用 operator[] 则会初始化实值， 若是 int 则初始化为 0，这是不安全的且取决于编译器，使用 at() 可以避免预期外的初始化。

- 
    * TODO: + 说明：
    如果代码中有该标识，说明在标识处有功能代码待编写，待实现的功能在说明中会简略说明。

    * FIXME: + 说明：
    如果代码中有该标识，说明标识处代码需要修正，甚至代码是错误的，不能工作，需要修复，如何修正会在说明中简略说明。

    * XXX: + 说明：
    如果代码中有该标识，说明标识处代码虽然实现了功能，但是实现的方法有待商榷，希望将来能改进，要改进的地方会在说明中简略说明。

- 二叉查找树（BST）节点排序考虑中序遍历，二叉树序列化及反序列化考虑先序遍历

- constexpr

- enum 与 enum class

- std::function 与 std::bind

- delete 与 default

- noexcept

- std::shared_ptr

    - 析构

        shared_ptr中的析构函数中不是直接释放资源，而是调用了dispose函数，如果*pn==0了，才会释放资源。

    -  多线程安全性

        shared_ptr 本身不是 100%线程安全的。它的引用计数本身是安全且无锁的，但对象的读写则不是，因为shared_ptr有两个数据成员，读写操作不能原子化。根据文档，shared_ptr的线程安全级别和内建类型、标准库容器、string一样，即：

        * 一个 shared_ptr 实体可被多个线程同时读取
        * 两个的 shared_ptr 实体可以被两个线程同时写入，“析构”算写操作
        * 如果要从多个线程读写同一个 shared_ptr 对象，那么需要加锁

    - shared_ptr和auto_ptr的区别

        - shared_ptr有两个变量，一个记录对象地址，一个记录引用次数
        - auto_ptr只有一个变量，用来记录对象地
        - shared_ptr，可以多个shared_ptr拥有一个资源
        - auto_ptr，只能一个auto_ptr拥有一个资源
        - shared_ptr可以实现赋值的正常操作，使得两个地址指向同一资源
        - auto_ptr的赋值很奇怪，源失去资源拥有权，目标获取资源拥有权
        - shared_ptr到达作用域时，不一定会释放资源
        - auto_ptr到达作用域时，一定会释放资源
        - shared_ptr存在多线程的安全性问题，而auto_ptr没有
        - shared_ptr可用于容器中，而auto_ptr一般不可以用于容器中
        - shared_ptr可以在构造函数、reset函数的时候允许指定删除器，而auto_ptr不能

- const

    - const函数

        1、如果一个成员函数在逻辑上不会修改对象的状态（字段），就应该定义为const函数;

        2、如果对象是const，则它只能调用const成员函数。

        如果对象是普通的非const对象：

        - 调用的某个成员函数是非const函数，则理所当然调用它。

        - 调用的某个成员函数是const函数，则当然也可以调用他。（底层const指针可以指向非常量对象）

        - 调用的某个成员函数同时存在 const版本和非const版本，则优先调用非const成员函数，编译器总是使用最匹配的版本。

    - const指针

        - 顶层const：

            指针变量本身是常量。（顶层const不适合引用，因为引用天生就是一个常量，始终引用一个对象，直到消亡）

            如 `int* const p = &a;`

        - 底层const：

            变量指向（或者引用）的对象被视为常量。(注意这里的用词：被视为，因为对象不一定就是常量，也可能是底层const指针、引用的一厢情愿)

            如 `const int*p ;` `int const *p;`这二者写法等价

    - const引用

        `const int& r;`

- 数组初始化是否为0

    ```c++
    int dp[n];//error, not 0
    int dp[n]{};//ok
    int dp[n]{0};//ok， 此方式可定义前几个元素，后面默认为0
    int *dp = new int[n];//ok       //LeetCode: error, not 0
    int *dp = new int[n]();//ok， recommend
    int *dp = new int[n]{};//ok
    int *dp = new int[n]{0};//ok， 此方式可定义前几个元素，后面默认为0
    ```

- `int p` 、`int *p`、`int *&p`

    int *p是定义一个指针

    int *&p是定义一个指针的引用

    指针当参数时，只能改名指针指向的内容，不能改变指针本身你。

    指针的引用当参数是，既可以改变指针指向的内容，又可以修改指针本身。