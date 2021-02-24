---
layout: post
title: c++关键字noncopyable
date: 2020-10-23 23:30:09
categories: Cpp
description: c++知识点
tags: Cpp
---


# boost::noncopyable 类

在c++中拷贝有等号拷贝 和 构造拷贝之分：

```c++
Foo foo, foo2;
Foo foo2 = foo; //等号拷贝
Foo foo3(foo);	//构造拷贝
```

等号拷贝是显式的，总得有个等号 = 在那才行。构造拷贝是隐式的，除了上面示例代码里那种直接写出构造函数的情况，在使用以值传递作为参数和以值返回的函数时，都会发生构造拷贝：

```c++
/// 传入 foo 时，发生了一次实参到形参的构造拷贝
Foo func(Foo foo) {
    Foo foo2;
    // do somthing
    return foo2;
}
/// 返回时，发生了一次 foo2 到返回值的的构造拷贝
```

类的等号拷贝和构造拷贝都是可以重载的。如果不重载，默认的拷贝模式是对每个类成员依次执行拷贝。

## 什么时候需要不可拷贝

考虑一种情况，我们要实现一个含有动态数组成员的类，其中动态数组成员在构造函数中 new 出来，在析构函数中 delete 掉。比如说这样一个矩阵类：

```c++
template<typename _T>
class Matrix {
public:
   int w;
   int h;
   _T* data;
   
   // 构造函数
   Matrix(int _w, int _h): w(_w), h(_h){
      data = new  _T[w*h];
   }

   // 析构函数
   ~Matrix() {
       delete [] data;
   }
}
```

这里我们没有重载拷贝函数，那么如果拷贝这个类，会发生什么呢？

```c++
/// 测试1：等号拷贝
void copy(){
    Matrix<double> m1(3,3);
    Matrix<double> m2(3,3);
    m1 = m2; 
}

/// 测试2：传参和返回中的构造拷贝
template<typename _T>
Matrix<_T> copy(Matrix<_T> cpy) {
    Matrix<_T> ret(cpy); 
    return ret;
} 
```

[1]. 上面的测试 1 中，我们先构造了 m1 和 m2 两个 Matrix 实例, 这意味着他们各自开辟了一块动态内存来存储矩阵数据。然后我们使用 = 将 m2 拷贝给 m1, 这时候 m1 的每个成员（w，h，data）都被各自使用 = 运算符拷贝为和 m2 相同的值。m1.data 是个指针，所以就和 m2.data 指向了同一块的内存。
于是这里就会出现两个问题：
- 1> 发生拷贝前 m1.data 指向的动态内存区在拷贝后不再拥有指向它的有效指针, 无法被释放, 于是发生了内存泄露
- 2> 在 copy() 结束后, m1 和 m2 被销毁, 各自调用析构函数, 由于他们的 data 指向同一块内存, 于是发生了双重释放。

[2]. ret随着函数结束而被销毁, 返回值由于拷贝自 ret, 所以其矩阵数据也将被销毁.


 因此，对于像 Matrix 这样的类，我们不希望这种拷贝发生。一个解决办法是重载拷贝函数，每次拷贝就开辟新的动态内存：

```c++
Matrix<_T>& operator = (const Matrix<_T>& cpy) {
    w = cpy.w;
    h = cpy.h;
    delete []  data;
    data = new _T[w*h];
    memcpy(data, cpy.data, sizeof(_T)*w*h);
    return *this;
}

Matrix(const Matrix<_T>& cpy):w(cpy.w), h(cpy.h) {
    data = new _T[w*h];
    memcpy(data, cpy.data, sizeof(_T)*w*h);
}
```

## 不可拷贝的类

使用 boost::noncopyable

```c++
class Matrix : boost::noncopyable
{
    // 类实现
}
```

## boost::noncopyable实现

直接把拷贝函数声明为私有:

```c++
private:  // emphasize the following members are private
    noncopyable( const noncopyable& );	//拷贝构造函数
    noncopyable& operator=( const noncopyable& );	//赋值运算符重载函数
```

上述的 matrix类可以改写：

```c++
template<typename _T>
class Matrix 
{
private: //设置为 private
    Matrix(const Matrix<_T>&);
    Matrix<_T>& operator = (const Matrix<_T>&);
}
```