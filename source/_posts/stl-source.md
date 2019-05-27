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

- __type_traits<T\>::has_trivial_constructor
- __type_traits<T\>::has_trivial_copy_constructor
- __type_traits<T\>::has_trivial_assignment_operator
- __type_traits<T\>::has_trivial_destructor
- __type_traits<T\>::is_POD_type

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

??? end() 指向空, 不是如 list 一般的空节点.

### associative containers

RB-tree 有自动排序功能, 而 hashtable 没有. 意味着 [multi]<set/map> 有自动排序功能, 而 hash_[multi]<set/map> 没有. 如果表格足够大, 后者可出现排序假象.

#### RB-tree

为了更大的弹性, SGI 将 RB-tree 迭代器实现为两层 (双层节点结构与双层迭代器结构设计), 这种设计理念和 slist 类似.

RB-tree 迭代器属于 Bidrectional Iterator, 其提领操作和成员访问操作与 list 十分相似.

(`p216`) ??? node->parent->parent = node
实现上的技巧: header->parent == root; root->parent == header; header->left == leftmost; header->right == rightmost;
??? RB-tree 的 begin() == leftmost; end() == header;
RB-tree 初始化时产生一个 header 节点空间, 令其为红色, header 左右子节点为自己.

RB-tree 一开始即要求用户必须明确设定所谓的 KeyOfValue 仿函数, 从实值 (value) 取出键值 (key).

(`p224`) !!! insert_unique()
             __rb_tree_rebalance()

#### [multi]<set/map>

set\<T>::iterator 被定义为底层 RB-tree 的 const_iterator, 杜绝写入操作, 也就是说, set 的 iterator 是一种 constant iterator, 相对于 mutable iterator 来说.
map 的键值不可改变, 实值可以改变, 因此 map 的 iterator 既不是 constant iterator 也不是 mutable iterator.

几乎所有的 set、map 操作行为都只是转调 RB-tree 的操作行为而已.

set、map 一定使用 RB-tree 的 insert_unique(), multiset、multimap 才使用 RB-tree 的 insert_equal().

`template <... class Compare = less<key> ...>`: 默认采用递增排序.

面对关联容器, 应该使用其所提供的 find 函数来搜寻元素, 会比使用 STL 的算法 find() 更有效率, 因为后者只是循序搜寻.

#### hashtable

bst 具有对数平均时间的表现, 而 hashtable 具有常数平均时间的表现, 且这种表现是以统计为基础, 不需仰赖输入元素的随机性.

包括有: 线性探测 (linear probing)、二次探测 (quadratic probing)、开链 (separate chaining)
前二者负载系数永远在 0~1 之间, 而开链策略可以大于 1.
SGI STL 的 hashtable 便是采用 开链.

元素的删除必须采用惰性删除, 也就是只标记删除记号, 实际删除操作则待表格重新整理 (rehashing) 时再进行, 这是因为hashtable 中每个元素不仅表述自己, 也关系到其他元素的排列.
二次探测主要用来解决主集团 (primary clustering) 的问题, 但却可能造成次集团 (secondary clustering) 的问题 (两个元素经 hash function 计算出来的位置若相同, 则插入时所探测的位置也相同, 造成某种浪费), 当然消除次集团的办法也有, 例如复式散列 (double hashing). 如果我们假设表格大小为`质数`, 而且永远保持负载系数在 0.5 以下, 那么就可以确定每插入一个新元素所需要的探测次数不多于 2.

在 SGI STL 中采用开链策略的 hashtable 中, buckets 以 vector 完成, bucket 所维护的链表, 并不采用 STL 的 list 或 slist, 而是自行维护 struct __hashtable_node.
迭代器为 Forward Iterator, operator++ 操作为 尝试访问当前节点的 next, 非空即是结果, 否则即是链表的尾端, 就跳至下一个 bucket.
虽然开链法不要求表格大小必须为质数, 但 SGI STL 仍然以质数来设计表格大小, 并且先将 28 个质数 (逐渐呈现约两倍的关系, 53、97、193、389...) 计算好, 以备随时访问.
插入行为可 insert_unique() (对应于非 multi) 与 insert_equal() (对应于 multi).
"表格重建与否"的判断原则为: 拿元素个数 (包括新增元素) 和 bucket vector 的大小比较, 如果前者大于后者就重建表格. 重建时, 先建立一个新 bucket vector, 将每个节点重新计算应该落在哪个 bucket 上后转移到新 bucket vector 上,最后与旧 bucket vector 进行 swap().

hashtable.size() 变化, 而不一定等于 hashtable.bucket_count(), 后者发生重建时变化.

SGI hashtable 无法处理 string (char* 却可以)、double、float 这些型别 (insert() 时才出错), 欲处理这些型别, 用户必须自行为它们定义 hash function.

#### hash_[multi]<set/map>

hash_[multi]<set/map> 默认表格大小为 100 (根据 STL 的设计, 采用质数 193).

## algorithms

`#include <numeric>` `#include <algorithm>`

质变算法, 诸如拷贝 (copy)、互换(swap)、替换(replace)、填写(fill)、删除(remove)、排列组合(permutation)、分割(partition)、随机重排(random shuffing)、排序(sort).
如果将此类算法运用于一个常数区间上, 编译器将会报错.
非质变算法, 诸如查找 (find)、匹配 (search)、计数 (count)、巡访 (for_each)、比较 (equal, mismatch)、寻找极值 (max, min).

所有泛型算法的前两个参数都是一对迭代器 [first, last), 必要条件是能够经由 increment 操作符的反复运用从 first 到达 last, 而编译器本身无法强求这一点, 如果不成立将会导致不可预期的结果.

将无效或者说不匹配的迭代器传给某个算法, 虽然是一种错误, 但却不能保证在编译期就能够被捕捉, 因为所谓"迭代器类型"并不是真正的型别, 它们只是一种型别参数 (type parameters).

## functors / function objects

`#include <functional>`

虽然函数指针可以达到"将整组操作当做算法的参数", 但其不能满足 STL 对抽象性的要求, 且无法和 STL 其他组件 (如 adapter) 搭配, 产生更灵活的变化.

仿函数按操作数 (operand) 个数分为 一元和二元 (STL 不支持三元仿函数), 按功能分为 算术运算 (Arithmetic)、关系运算 (Rational)、逻辑运算 (Logical).

证同 (identity)、选择 (select)、投射 (project) 仿函数并不在 C++标准规格之内, 但它们常常存在于各个实现品中作为内部使用
证同: 直接返回参数. 在 ...set 中用来指定 RB-tree 所需的 KeyOfValue op, 因为 set 的键值即其实值.
选择: 接收 pair, 直接返回 first 或 last. 在 ...map 中用来指定 RB-tree 所需的 KeyOfValue op, 因为 map 的键值即 pair 的 first.
投射: 接收两个参数, 直接返回第一参数或第二参数.

## adapters

container adapters: 包括 queue 和 stack 即是修饰 deque 的接口而成.
iterator adapters: 包括 insert iterator、reverse iterator、iostream iterator.
functor/funciton adapters: 配接操作包括 绑定 (bind)、否定 (negate)、组合 (compose).

所有期望获得配接能力的组件, 本身必须是可配接的 (adaptable). 换句话说, 一元仿函数必须继承自 unary_function, 二元仿函数必须继承自 binary_function, 成员函数必须以 mem_fun 处理过, 一般函数必须以 ptr_fun 处理过. 一个未经 ptr_fun 处理过的一般函数, 虽然也可以函数指针的形式传给 STL 算法使用, 却无法拥有任何配接能力.

insert iterator 有三种: back_insert_iterator、front_insert_iterator、insert_iterator, 其 iterator_category 都为 output_iterator_tag;
reverse_iterator<iterator> 的 5 种相应型别都与其对应的正向迭代器相同;
istream_iterator 的 iterator_category 为 input_iterator_tag;
ostream_iterator 的 iterator_category 为 output_iterator_tag.

![reverse iterator](/images/riterator.png)

container adapter 内藏了一个 container member;
insert iterator   内藏了一个 pointer to container (并因而取得其 iterator);
reverse iterator  内藏了一个 iterator number;
stream iterator   内藏了一个 pointer to stream;
function adapter  内藏了一个 member object, 其型别等同于它所要配接的对象 (这个对象当然是一个 adaptable functor).

源码中经常出现的 pred 一次, 是 predicate 的缩写, 意指会返回真假值 (bool)  的表达式.

## conclusion

容器以 class template 完成;
算法以 function templates 完成;
仿函数是一种将 operator() 重载的 class template;
迭代器是一种将 operator++ 和 operator* 等指针习惯常行为重载的 class template;
配接器中的 container adapter 和 iterator adapter 都是一种 class template, 而 function adapter.
