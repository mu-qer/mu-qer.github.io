---
layout: post
title: std::function && std::bind
date: 2021-08-15 23:30:09
categories: Cpp
description: std function
tags: Cpp
---

# 可调用对象

可调用对象有以下几种定义：
> - 是一个函数指针
> - 是一个operator()成员函数的类的对象
> - 可被转换成函数指针的类对象
> - 一个类成员函数的指针

不同类型可能有不同的调用形式，如：
```c++
// 普通函数
int add(int a, int b){return a+b;} 

// lambda表达式
auto mod = [](int a, int b){ return a % b;}

// 函数对象类
struct divide{
    int operator()(int denominator, int divisor){
        return denominator/divisor;
    }
};
```
上述三种可调用对象虽然类型不同，但是共享了一种调用形式:
```c++
int(int, int)
```

std::function就可以将上述类型保存起来，如下：
```c++
std::function<int(int ,int)>  a = add; 
std::function<int(int ,int)>  b = mod ; 
std::function<int(int ,int)>  c = divide(); 
```

# std::function

> - std::function是一个可调用对象包装器, 是一个类模板, 可以容纳除了类成员函数指针之外的所有可调用对象, 可以提前绑定并延迟执行.
> - 定义格式： std::function<函数类型>
> - std::function可以取代函数指针的作用, 因为它可以延迟函数的执行, 特别适合作为回调函数使用.

```c++
#include <functional>
#include <iostream>

int foo(int para) {
	return para;
}

int main() {
	//std::cuntion 包装了一个返回值为int, 参数为int的函数
	std::function<int(int)> func = foo;
	std::cout << func(10) << std::endl;
}
```

# std::bind

> - std::bind 是用来绑定函数调用的参数的.
> - 它解决的需求是我们有时候可能并不一定能够一次性获得调用某个函数的全部参数, 通过这个函数, 我们可以将部分调用参数提前绑定到函数身上成为一个新的对象, 然后在参数齐全后, 完成调用。

```c++
int foo(int a, int b, int c) {
	;
}

int main() {
	//将参数1，2绑定到foo上, 使用 std::placeholders::_1对第一个参数进行占位
	auto bindFoo = std::bind(foo, std::placeholders::_1, 1, 2);
	//调用时仅提供第一个参数即可
	bindFoo(1);
}
```

```c++
//std::bind绑定一个成员函数
struct Foo {
	void printSum(int a, int b) {
		std::cout << a+b << std::endl;
	}
	int data = 10;
};

int main() {
	Foo foo;
	auto f = std::bind(&Foo::printSum, &foo, 99, std::placeholders::_1);
	f(1);
}
```

> - bind绑定类成员函数时, 第一个参数表示 '对象的成员函数的指针', 第二个参数表示对象的地址
> - 必须显式的指定 &Foo::printSum, 因为编译器不会将对象的成员函数隐式的转换成函数指针, 所以必须在Foo::printSum前添加&
> - 使用对象成员函数指针时, 必须要只要该指针属于哪个对象, 因此第二个参数是 '对象的地址', &foo








