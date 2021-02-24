---
layout: post
title: c++关键字_enable_shared_from_this
date: 2020-10-23 23:30:09
categories: Cpp
description: c++知识点
tags: Cpp
---


首先要说明的一个问题是:
> 如何安全地将 this 指针返回给调用者? 

一般来说, 我们不能直接将 this 指针返回. 想象这样的情况, 该函数将 this 指针返回到外部某个变量保存, 然后这个对象自身已经析构了, 但外部变量并不知道, 那么此时如果外部变量使用这个指针, 就会使得程序崩溃.
那么使用智能指针 shared_ptr 呢? 当然, 使得 c++ 指针安全目前来说最有效也使用最多的办法就是使用智能指针 shared_ptr. 首先假设你已经理解shared_ptr 的工作原理。然后我们来看下面一份代码：

```c++
#include <iostream>
#include <memory>

class Bad {
public:
    Bad()  { std::cout << "Bad()" << std::endl; }
    ~Bad() { std::cout << "~Bad()" << std::endl; }
    std::shared_ptr<Bad> getPtr() {
        return std::shared_ptr<Bad>(this);  //返回shared_ptr
    }
};

int main(int argc, char const *argv[])
{
    std::shared_ptr<Bad> bp1(new Bad);
    std::shared_ptr<Bad> bp2 = bp1->getPtr();
    std::cout << "bp2.use_count: " << bp2.use_count() << std::endl;
    return 0;
}

//运行结果：
Bad()
bp2.use_count: 1
~Bad()
~Bad()
a(822,0x7fff737b5300) malloc: *** error for object 0x7fe74bc04b50: pointer being freed was not allocated
```

我们可以看到, 对象只构造了一次, 但却析构了两次. 并且在增加一个指向的时候 shared_ptr的计数并没有增加. 也就是说, 这个时候 bp1 和 bp2 都认为只有自己才是这个智能指针的拥有者。其实也就是说, 这里建了两个智能指针同时处理 this 指针. 每个智能指针在计数为0的时候都会调用一次Bad对象的析构函数. 所以会出问题.

其实现在问题就变成了, 如何在对象中获得一个指向当前对象的 shared_ptr 对象. 如果我们能够做到这个, 那么直接将这个shared_ptr 对象返回, 就不会造成新建 shared_ptr 的问题了.

enable_shared_from_this 是一个以其派生类为模板类型实参的基类模板，继承它，派生类的this指针就能变成一个 shared_ptr。先来看下面一份代码:

```c++
#include <iostream>
#include <memory>

class Good: public std::enable_shared_from_this<Good> {  //注意模板类参数的填写，Good
public:
    Good() { std::cout << "Good()" << std::endl; }
    ~Good() { std::cout << "~Good()" << std::endl; }

    std::shared_ptr<Good> getPtr() {
        return shared_from_this();    // shared_ptr
    }
};

int main(int argc, char const *argv[])
{
    std::shared_ptr<Good> bp1(new Good);
    std::shared_ptr<Good> bp2 = bp1->getPtr();
    std::cout << "bp2.use_count: " << bp2.use_count() << std::endl;
    return 0;
}

//执行结果：
Good()
bp2.use_count: 2
~Good()
```

# enable_shared_from_this 的工作原理

```c++
template<class T> class enable_shared_from_this
{
protected:

    enable_shared_from_this() BOOST_NOEXCEPT {}
    enable_shared_from_this(enable_shared_from_this const &) BOOST_NOEXCEPT  {}

    enable_shared_from_this & operator=(enable_shared_from_this const &) BOOST_NOEXCEPT {
        return *this;
    }
    ~enable_shared_from_this() BOOST_NOEXCEPT {} // ~weak_ptr<T> newer throws, so this call also must not throw


public:

    shared_ptr<T> shared_from_this()
    {
        shared_ptr<T> p( weak_this_ );
        BOOST_ASSERT( p.get() == this );
        return p;
    }

    shared_ptr<T const> shared_from_this() const
    {
        shared_ptr<T const> p( weak_this_ );   //每次调用shared_from_this()时候，就会根据 weak_ptr来构造一个临时的shared_ptr
        BOOST_ASSERT( p.get() == this );
        return p;
    }

public: // actually private, but avoids compiler template friendship issues

    // Note: invoked automatically by shared_ptr; do not call
    template<class X, class Y> void _internal_accept_owner( shared_ptr<X> const * ppx, Y * py ) const
    {
        if( weak_this_.expired() )
        {
            weak_this_ = shared_ptr<T>( *ppx, py );
        }
    }

private:

    mutable weak_ptr<T> weak_this_;  //注意这里是使用一个weak_ptr对象来构造对外的 shared_ptr对象。
    
};
```

这里会有疑问, shared_from_this()返回的 shared_ptr[107行] 和 之前的[26行]有什么区别？以及,为什么 enable_shared_from_this不直接保存一个 shared_ptr成员？

> - 任何继承自 enable_shared_from_this类的子类A，都会生成一个A类的weak_ptr对象: weak_this_. 而每次调用shared_from_this(), 都是通过同一个 weak_ptr对象来构建出的 shared_ptr对象.
> - 如果enable_shared_from_this类使用的是shared_ptr来保存对象指针, 而不是weak_ptr. 那么该shared_ptr指向自身，那么这个shared_ptr的引用计数最少会是1, 也就是这个对象将永远不能被析构, 而使用weak_ptr不会引起shared_ptr计数的增加。

>> enable_shared_from_this 的成员变量在 enable_shared_from_this 构造的时候是没有指向任何对象的, 在第一次调用 shared_ptr的时候, 由 shared_ptr 调用 boost::detail::sp_enable_shared_from_this 然后调用enable_shared_from_this 的 internal_accept_owner来对其进行初始化。看下面的过程就明白了:

```c++
template<class Y>
explicit shared_ptr( Y * p ): px( p ), pn( p ) 
{   //这里使用了 boost::detail::sp_enable_shared_from_this
    boost::detail::sp_enable_shared_from_this( this, p, p );
}
```

```c++
template< class X, class Y, class T >
inline void sp_enable_shared_from_this( boost::shared_ptr<X> const * ppx,
                                        Y const * py, 
                                        boost::enable_shared_from_this< T > const * pe )
{
    if( pe != 0 )
    {   //而在这里面又调用 enable_shared_from_this 的 _internal_accept_owner
        pe->_internal_accept_owner( ppx, const_cast< Y* >( py ) );
    }
}
```

```c++
template<class X, class Y> void _internal_accept_owner( shared_ptr<X> const * ppx, Y * py ) const
{
    if( weak_this_.expired() ) // 成员函数expired()的功能等价于use_count()==0
    {
        weak_this_ = shared_ptr<T>( *ppx, py );
    }
}
```

> 从这个构造过程可以看出，当 weak_ptr 第一次指向一个对象后，它就担起了观察者的角色，一直观察着同一个对象


# 什么场景下需要在成员函数中获取对象自己的智能指针？

- 声明周期保护. 比如, 我们设计了一个回调, 当回调被调用时在成员函数中做一些对象的清理操作(或其他任何操作)

```c++
#inlcude <memory>

class A : public std::enable_shared_from_this<A> {
public:
	void Test() {
		//下面一行代码保护对象生命周期
		auto self = shared_from_this();
		{
			std::lock_guard<std::mutex> lock(mutex_);
			//该函数被回调时候，如果该对象已经被上层释放，那么程序就会crush
			delegate_->OnComplete(); //do something
		}
	}
}
```

- lambda 表达式捕获 this指针, 如果this指向的对象被析构了, 那么再次使用this时候就会crush

```c++
#include <memory>
class Apple {
 public:
  void Test() {
    //lambda 捕获this 指针
    http_client_->Request([this] {
      // Do Something
    });
  }

 private:
  std::unique_ptr<HttpClientt> http_client_;
}

//改进:
class Apple : public std::enable_shared_from_this<Apple> {
public:
  Apple() {

  }
 void Test() {
   auto weak_self = weak_from_this();
   //这里捕获weak，所以如果Apple 对象被析构了。 weak_self.lock将返回一个null
   http_client_->Request([weak_self] {
     if (auto self = weak_self.lock()) {
       //Do Something
     }
   });
 }

 private:
  std::unique_ptr<HttpClientt> http_client_;
};
```


# 注意点

- 不能在构造函数中使用
- 不能在析构函数中使用 (进入到对象的析构函数，说明强引用计数已经变为0了，这个时候转换成shared_ptr肯定是不行的)

















