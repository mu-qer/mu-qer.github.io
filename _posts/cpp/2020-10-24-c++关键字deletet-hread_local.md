---
layout: post
title: c++关键字delete/thread_local
date: 2020-10-23 23:30:09
categories: Cpp
description: c++知识点
tags: Cpp
---

## c++11下使用delete关键字

C++ 11 中为不可拷贝类提供了更简单的实现方法，使用 delete 关键字即可:

```c++
template<typename _T>
class Matrix 
{
public:
    Matrix(const Matrix<_T>&) = delete;
    Matrix<_T>& operator = (const Matrix<_T>&) = delete;
}
```


# syscall(SYS_gettid)

在linux下每一个进程都一个进程id，类型pid_t，可以由getpid（）获取。POSIX线程也有线程id，类型pthread_t，可以由pthread_self（）获取，线程id由线程库维护。但是各个进程独立，所以会有不同进程中线程号相同节的情况。那么这样就会存在一个问题，我的进程p1中的线程pt1要与进程p2中的线程pt2通信怎么办，进程id不可以，线程id又可能重复，所以这里会有一个真实的线程id唯一标识，tid。glibc没有实现gettid的函数，所以我们可以通过linux下的系统调用syscall(SYS_gettid)来获得。


# __thread

__thread是GCC内置的线程局部存储设施。_thread变量每一个线程有一份独立实体，各个线程的值互不干扰。

```c++
__thread int count = 0;
int main()
{
    //创建线程A
    //创建线程B
    //此时A和B线程都有一个实体 count，二者并不相同
}
```

## 使用方式

```c++
__thread int i;
extern __thread struct state s;
static __thread char* p;
```

## 注意事项

- __thread 只能修饰POD类型(类似整型指针的标量，不带自定义的构造、拷贝、赋值、析构的类型，二进制内容可以任意复制memset,memcpy,且内容可以复原)
- 不能修饰 class 类型，无法自动调用构造和析构函数.
- 可以用于修饰全局变量, 函数内的静态变量, 不能修饰函数的局部变量或者class的普通成员变量, 且__thread变量值只能初始化为编译器常量。

## 可以使用以下函数设置

```c++
int pthread_once(pthread_once_t *once_control, void (*init_routine)(void));
int pthread_key_create(pthread_key_t *key, void (*destructor)(void*));
void *pthread_getspecific(pthread_key_t key);
int pthread_setspecific(pthread_key_t key, const void *value);
```

c++11引入了线程存储器符号： thread_local


# __builtin_expect 说明

> 这个指令是gcc引入的，作用是允许程序员将最有可能执行的分支告诉编译器。这个指令的写法为：__builtin_expect(EXP, N)。
意思是：EXP==N的概率很大。


一般的使用方法是将__builtin_expect指令封装为likely和unlikely宏。这两个宏的写法如下.

```c++
#define likely(x) __builtin_expect(!!(x), 1) //x很可能为真       
#define unlikely(x) __builtin_expect(!!(x), 0) //x很可能为假
```

## 内核中的 likely()和 unlikely()

首先要明确一点：

```c++
if(likely(value))     //等价于 if(value)
if(unlikely(value))   //等价于 if(value)
```

__builtin_expect() 是 GCC (version >= 2.96）提供给程序员使用的，目的是将“分支转移”的信息提供给编译器，这样编译器可以对代码进行优化，以减少指令跳转带来的性能下降。
__builtin_expect((x),1)表示 x 的值为真的可能性更大；
__builtin_expect((x),0)表示 x 的值为假的可能性更大。
也就是说，使用likely()，执行 if 后面的语句的机会更大，使用 unlikely()，执行 else 后面的语句的机会更大。通过这种方式，编译器在编译过程中，会将可能性更大的代码紧跟着起面的代码，从而减少指令跳转带来的性能上的下降。


例子：
```c++
int x, y;
if(unlikely(x > 0))
    y = 1; 
else 
    y = -1;
```

上面的代码中 gcc 编译的指令会预先读取 y = -1 这条指令,这适合 x 的值大于 0 的概率比较小的情况。
如果 x 的值在大部分情况下是大于 0 的,就应该用 likely(x > 0),这样编译出的指令是预先读取 y = 1 这条指令了。
这样系统在运行时就会减少重新取指了。















