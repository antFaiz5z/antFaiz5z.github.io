---
title: 《STL 源码剖析》阅读笔记
comments: true
toc: true
date: 2019-04-29 17:01:15
categories: C/C++
tags: STL
---

本书针对于 SGI STL.

<!--more-->

## allocator

## iterator

最常用到的迭代器相应型别有五种:

- value type
- difference type
- pointer
- reference
- iterator category

针对最后一种型别, 根据移动特性和施行措施, 迭代器被分为五类:

- Input Iterator
    (struct input_iterator_tag)
- Output Iterator
    (struct output_iterator_tag)
- Forward Iterator
    (struct forward_iterator_tag)
- Bidirectional Iterator
    (struct bidirectional_iterator_tag)
- Random Access Iterator
    (struct random_access_iterator_tag)

![各迭代器继承关系](/images/iterator.png)

原生指针是一种 Random Access Iterator.

traits 编程技法大量运用于 STL 实现品中. 它利用"内嵌型别"的编程级别与编译器的 template 参数推导功能, 增强 C++ 未能提供的关于型别认证方面的能力, 弥补 C++ 不为强型别 (strong typed) 语言的遗憾.

iterator_traits 负责萃取迭代器的特性, __type_traits 负责萃取型别 (type) 的特性. 此处关注的型别特性是指, 这个型别是否具有 default cto？ 是否具有 non-trivial copy ctor？ 是否具有 non-trivial assignment operator? 是否具有 non-trivial dtor? 等. 如果答案是否定的, 在对这个型别进行构造、析构、拷贝、赋值操作时, 就可以采用内存直接处理操作如 malloc()、memcpy 等, 获得最高效率.

- __type_traits\<T>::has_trivial_constructor
- __type_traits\<T>::has_trivial_copy_constructor
- __type_traits\<T>::has_trivial_assignment_operator
- __type_traits\<T>::has_trivial_destructor
- __type_traits\<T>::is_POD_type

上述式子的返回值不是只是简单的 bool 值, 而是空的结构体: struct `__true_type` 和 struct `__false_type`, 因为如此编译器可以做类型推导. 一般具现体 (general instantiation), 内含对所有型别都必定有效的保守值. 上述各个 has_trivial_xxx 及 is_POD_type 型别都被定义为 `__false_type`, 就是对所有型别都必定有效的保守值. 而对于 C++ 基本型别 (char、int、unsigned int 等) 在 <type_traits.h> 中提供特化版本, 上述每一个成员的值都是 `__true_type`, 表示这些型别可以采用最快速方式 (例如 memcpy) 来进行拷贝或赋值操作.

## container

![SGI STL 的各种容器 (以内缩方式表示基层与衍生层关系)](/images/container.png)

这里所谓的衍生并非派生 (inheritance) 关系, 而是内含 (containment) 关系. 例如
heap 内含一个 vector,
priority_queue 内含一个 heap,
stack 和 queue 内含一个 deque,
set/map/multiset/multimap 都内含一个 RB-tree,
hash_<set/map/multiset/multimap> 都内含一个 hashtable.

### sequence containers

#### vector

vector 默认使用 alloc 作为空间配置器.
vector 的迭代器是普通指针, 也就是 Random Access Iterator.

增加新元素时若超过 capacity, 则 capacity 扩充至两倍 (不同 STL 实现此数字可不同), 若两倍仍不足则继续扩充. 容量的扩充必须经历"重新配置、元素移动、释放原空间"等过程.
重新配置: 如果原大小为 0, 则配置 1 (个元素大小), 不为 0 则配置为原来的两倍.
元素移动: 先将原 vector 从 start 至 position (插入点) 的部分拷贝至新 vector (此时新 vector 的 finish == position), 再将新 vector 从 finish 开始的剩余部分赋初值 x (原插入值), 接着 ++finish (相当于已将 x 插入), 最后将原 vector 从 position 开始的剩余部分拷贝至新 vector.
释放原空间: 调用 destroy() 和 deallocate().

在 push_back() 中因备用空间不够而调用的 insert_aux() 即是采用上述流程, 然而在 push_back() 中实际上插入点 position 已经等于 finish, 所以流程中后半部分的将 position 到 finish 部分进行拷贝是否必要有待商榷, 书中提到了译者侯捷对此也抱有疑问.

在 insert() 中便区分有三种情况: (finish - position) > n、(finish - position) <= n、备用空间不够， 其中 n 为插入个数. 第三种情况即是采用上述流程, 稍微不同的是中间是插入 n 个 x 而不是简单的 ++finish. 第一种情况直接将插入点后的元素向后拷贝 copy_backward(), 第二中情况先在 finish 之后填充 x 再 copy_backward() 再在 finish 之前覆盖 x.

#### list

list 默认使用 alloc 作为空间配置器，并据此定义了一个 list_node_allocator.
list 不仅是一个双向链表, 还是一个环状双向链表, 所以它只需要一个指针便可以完整表现整个链表. 所以 list 提供的是 Bidirectional Iterator.

list 进行 insert() 和 spice() 操作都不会使迭代器失效 (而 vector 的插入操作可能会使内存重新配置而导致所有迭代器失效), 甚至 erase() 操作也只会使指向被删除元素的迭代器失效其他不受影响.

list 初始化时即有一个空白节点, begin() 初始时指向此, end() 始终指向于此.

(`p138`) ??? next = first // 修正区段范围

list 不能使用 STL 算法 sort(), 必须使用自己的 sort() 成员函数 (使用了快排`p142`), 因为前者只接受 RandomAccessIterator.

#### deque

deque 没有所谓容量 (capacity) 概念, 因为它是以分段连续空间组合而成, 随时可以增加一段新的空间并链接起来.

deque 提供 Random Access Iterator, 但它的迭代器并不是普通指针, 其复杂度高 (实现代码分量比 vector 和 list 多得多).

中控器 map (非 STL 中的 map 容器), 一个 map 要管理多个节点, 最少8个, 最多是 "所需节点数加 2"(前后各预留一个, 扩充时可用).

(`p151`) ??? __deque_iterator<> 没有为 (finish -1) 定义运算子.
         ??? // 下行最后有两个 ';', 虽奇怪但符合语法

deque 自行定义了两个专属的空间配置器: data_allocator (每次配置一个元素大小)、map_allocator (每次配置一个指针大小)

deque 的最初状态 (无任何元素时) 保留有一个缓冲区, y因此 clear() 完成后恢复初始状态也一样保留一个缓冲区：若 start 与 finish 在不同缓冲区, 将除头缓冲区与尾缓冲区之外的缓冲区内的所有元素析构并释放缓冲区内存, 再将头尾缓冲区的元素析构并释放尾缓冲区的内存而保留头缓冲区; 若 start 与 finish 在同一缓冲区, 则析构这唯一缓冲区所有元素并保留此缓冲区.

#### stack

SGI STL 默认以 deuqe 作为 stack 的底部结构 (以 list 同样可以做到), 因此其实现非常简单. 往往被归类为容器配接器.

stack 不提供遍历功能, 也不提供迭代器.

#### queue

SGI STL 默认以 deuqe 作为 queue 的底部结构 (以 list 同样可以做到), 因此其实现非常简单. 往往被归类为容器配接器.

queue 不提供遍历功能, 也不提供迭代器.

#### heap

heap 不属于 STL 容器组件, 扮演 priority_queue 的助手. priority_queue 的复杂度最好介于 queue 与 binary search tree 之间, 因此 binary heap 便是这种条件下的适当候选人. 所谓 binary heap 就是一种 complete binary tree.

一种 array 隐式表述 tree 的方式: 将 array 的 #0 元素保留, 那么当某个节点位于 array 的 #i 处, 它的左子节点位于 #2i 处, 右子节点位于 #2i +1处, 父节点位于 #i/2 处.

推荐使用 vector 而不是 array, array无法动态改变大小, 因此不可以对满载的 array 进行 push_heap() 操作.

按照元素排列方式, heap 可分为 max-heap 和 min-heap, STL 提供的是 max-heap.

heap 不提供遍历功能, 也不提供迭代器.

##### push_heap()

percolate up (上溯)：
新加入的元素首先一定放在最下一层作为叶节点, 填补在从左至右第一个空格, 也就是插入到 vector 的 end() 处.
为满足 mxa-heap 的条件 (每个节点的键值都大于等于其子节点键值), 如果键值比父节点大, 就父子对换键值, 直到不需对换或到根节点为止.

该函数接受两个迭代器用来表现 heap 底部容器 vector 的头尾, 并且前提是新元素已插入到 end().

(`p175`) ??? parent = (holeIndex - 1) /2
意味着具体实现并不是如上所述将 #0 保留的隐式表述法?

##### pop_heap()

percoalte down (下溯)：
pop_heap() 操作取走根节点 (具体实现中是首先将根节点与最右下节点键值对换, 最后用 pop_back() 原根节点, pop_heap() 中未调用 pop_back()), 任务是将最右下的元素重新找一个合适的位置, 首先将其填入根节点位置, 再与左右子节点比较键值, 如果其键值较小, 则与较大子节点对换键值, 直到不需对换或下放到叶节点为止.

该函数接受两个迭代器用来表现 heap 底部容器 vector 的头尾.

(`p177`) ??? 未比较 value 与左右子节点键值大小
总是小于右子节点？

##### sort_heap()

多次 pop_heap() 且每次操作范围从后向前缩减一个元素 (pop_heap() 会将最大元素放在底部容器最尾端), 最后便得到一个递增序列. 排序过后, 原来的 heap 就不是一个合法的 heap 了.

该函数接受两个迭代器用来表现 heap 底部容器 vector 的头尾.

##### make_heap()

这个算法用来将一段现有的数据转化为一个 heap, 主要依据前文所述的隐式表述法.

(`p180`) ??? parent = (len -2) /2
最后一层叶子节点在后续的 adjust_heap() 中自然会归于上下有序.

#### priority_queue

priority_queue 默认使用一个 max-heap 完成.

SGI STL 默认以 vector 作为 priority_queue 的底部结构, 再加上 heap 处理规则, 因此其实现非常简单. 往往被归类为容器配接器.

priority_queue 不提供遍历功能, 也不提供迭代器.

构造函数调用 make_heap();
push() 先后调用 push_back()、push_heap();
pop() 先后调用 pop_heap()、pop_back();

#### slist

SGI STL 提供单向链表, 不在标准之内.

slist 和 list 的主要区别在于前者的迭代器属于 Forward Iterator, 而后者为 Bidirectional Iterator.

基于效率考虑, slist 特别提供了 insert_after() 和 erase_after(), 并不提供 push_back(), 只提供 push_front().

??? end() 指向空, 不是 list 一般的空节点.

### associative containers

#### RB-tree

#### set

#### map

#### multiset

#### multimap