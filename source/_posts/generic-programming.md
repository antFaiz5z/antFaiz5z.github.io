---
title: 泛型编程与模板
comments: true
toc: true
date: 2018-07-12 11:19:23
categories: C/C++
tags: template
---

Template!!!

<!--more-->

# 声明与定义函数模板的两种方式

## 声明与实现都放在.h中

## 声明在.h中，实现在.cpp中

即：类模板内部使用泛型和非泛型类型，类外定义函数成员

优点：简化条理化代码，实现声明和实现分离，高内聚低耦合，对外提供简单的接口，对内高内聚 。

``` c++
template<typename T, int N >
class TinyVector
{
public:
    // 模板类里面的正常函数
    TinyVector( const T& value );

    // 模板类里面含有模板函数 
   template< typename T2 >
    TinyVector<T,N>& operator= ( const TinyVector<T2,N>& v );
}

外部实现：
// 模板类的正常函数
template< typename T, int N >
TinyVector<T,N>::TinyVector( const T& value )
{
    for ( int i = 0; i < N; ++i ) {
        data_[i] = value;
    }
}

// 模板类里面的模板函数
template< typename T, int N >
template< typename T2 >
TinyVector<T,N>& TinyVector<T,N>::operator= ( const TinyVector<T2,N>& v )
{
    for ( int i = 0; i < N; ++i ) {
        data_[i] = v(i);
    }
    return *this;
}

```

即需要在类名后添加类模板的类型参数。

注意：类模板中所有函数成员都需要如此，即使没有用到泛型变量。

## 模板声明和实现分离后的调用原理

模板不能分离的原因：

C++标准明确表示，当一个模板不被用到的时侯它就不该被实例化出来，cpp中没有将模板类实例化，所以实际上.cpp编译出来的t.obj文件中关于模板实例类型的一行二进制代码，于是连接器就傻眼了，只好给出一个连接错误。所以这里将.cpp包含进来，其实还是保持模板的完整抽象，main中可以对完整抽象的模板实例化。

进行分离的方法：

类的声明必须在实现的前面，所以使用的时候包含cpp就可以把.h和.cpp中定义的内容一并包含了，其实是放在同一个文件中一个意思。
这种模板分离的方式缺点：如果这个模板类的cpp有非模板的定义，能够有效实例化，但是会导致重复包含定义而出错所以有些模板的实现和分离放到了.tcc格式或者.inl格式的文件中(这些格式的文件都是来自于txt文本的不能从cpp直接修改得到)，在.h中直接包含进去,代码中就可以直接包含ClassTemplate.h了，避免包含.cpp奇怪的行为。

# 模板参数作容器参数迭代器报错

## vector<T\>::const_iterator

``` c++
std::vector<T>::const_iterator iter;
//error: expected ‘;’ before ‘iter’

//solution:
typename std::vector<T>::const_iterator iter;
```

原因：

 首先类除了可以定义数据成员或函数成员之外，还可以定义类型成员。使用std::vector<T>::const_iterator时，编译器假定这样的名字指定的是数据成员，而不是数据类型成员。如果希望编译器将const_iterator当做类型，则必须显示告诉编译器这样做，这就是我们加typename的原因。

## typename const

``` c++
typename const std::list<T>::iterator end = dest.end();  //错误

const typename std::list<T>::iterator end = dest.end();  //正确
```

原因：

应该是typename后面接的下一个单词须是个类型名，而不应是const。