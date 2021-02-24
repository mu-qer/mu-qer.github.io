---
layout: post
title: c++关键字__builtin_expect
date: 2020-10-23 23:30:09
categories: Cpp
description: c++知识点
tags: Cpp
---


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















