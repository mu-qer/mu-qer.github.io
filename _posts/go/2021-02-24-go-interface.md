---
layout: post
title: go interface
date: 2021-03-12 23:30:09
categories: Go
description: go interface
tags: Go
---

# interface

我理解的在计算机系统中的 interface 的意思是：某种中间层, 拥有默许的一套使用规则. 只要调用方遵循这个规则，就可以通过它实现一些操作. 就像我们常用的系统调用 open/read/write 等都是接口, 我们使用的时候只需要正常调用即可(接口名, 接口参数类型等配套)。

接口的好处:
> 上下游通过接口解耦, 接口隔离了底层的实现, 让我们将当前重点投入到自身逻辑中.

# go interface

一个常规的go的interface是这样的:

```go
type Ierror interface {
        Error() string
}
```
如果一个类型需要实现 error 接口, 那么它只需实现 Error() string 方法即可:

```go
type MError struct {
        CodeNum int
        Message string
}

func (m *MError) Error() string { 
        return fmt.Sprintf("%s, codeNum:%d", m.Message, m.CodeNum)
}

func NewMError(code int, msg string) Ierror {
        return &MError{  //typecheck 3
                CodeNum: code,
                Message: msg,
        }
}

func getError(e Ierror) Ierror {
        return e
}

func main() {

        var mer Ierror = NewMError(404, "not found") // typecheck 1
        err := getError(mer)	// typecheck 2
        fmt.Println(err)
}
```

Go 语言在编译期间对代码进行类型检查，上述代码总共触发了三次类型检查：
- 将 * NewMError 类型的变量赋值给 Ierror 类型的变量 mer;
- 将 * NewMError 类型的变量 mer 传递给签名中参数类型为 Ierror 的 getError 函数;
- 将 * NewMError 类型的变量从函数签名的返回值类型为 Ierror 的 NewMError 函数中返回;
从类型检查的过程来看，编译器仅在需要时才检查类型，类型实现接口时只需要实现接口中的全部方法



go中的 interface{} 类型不是任意类型(与C中的 void * 不同). 如果我们将类型转换成了 interface{}, 变量在运行期间的类型也会发生变化, 获取变量类型时会得到 interface{}.

```go
func main() {
        type Test struct{}
        v := Test{}
        Print(v)
}

func Print(v interface{}) {
        println(v)
}
```
上述函数不接受任意类型的参数, 只接受 interface{} 类型的值, 在调用 Print 函数时会对参数 v 进行类型转换, 将原来的 Test 类型转换成 interface{} 类型, 本节会在后面介绍类型转换的实现原理.

## go中的指针和接口

在 Go 语言中同时使用指针和接口时会发生一些让人困惑的问题, 接口在定义一组方法时没有对实现的接收者做限制, 所以我们会看到某个类型实现接口的两种方式:

```go
type Duck interface{
        Walk()
        Quack()
}

type Cat struct{}

//1
func (c *Cat) Walk() {
        fmt.Println("cat walk")
}
func (c *Cat) Quack() {
        fmt.Println("miao")
}

//2 和 1 不能同时存在
func (c Cat) Walk() {  //c为非指针对象
        fmt.Println("cat walk")
}
func (c Cat) Quack() {
        fmt.Println("miao")
}

func main() {
        var c Duck = &Cat{}  //
        c.Quack() // 调用1

        c = Cat{}
        c.qUACK() //调用2
}
```
无论上述代码中初始化的变量 c 是 Cat{} 还是 &Cat{}，使用 c.Quack() 调用方法时都会发生值拷贝:

- 对于 &Cat{} 来说, 这意味着拷贝一个新的 &Cat{} 指针, 这个指针与原来的指针指向一个相同并且唯一的结构体, 所以编译器可以隐式的对变量解引用（dereference）获取指针指向的结构体;

- 对于 Cat{} 来说, 这意味着 Quack 方法会接受一个全新的 Cat{}, 因为方法的参数是 * Cat, 编译器不会无中生有创建一个新的指针; 即使编译器可以创建新指针, 这个指针指向的也不是最初调用该方法的结构体;


## nil or non-nil

```go
type TestStruct struct{}

func NilOrNot(v interface{}) bool {
        return v == nil
}

func main() {
        var s *TestStruct
        fmt.Println(s == nil)      // #=> true
        fmt.Println(NilOrNot(s))   // #=> false
}

$ go run main.go
true
false
```
> 调用 NilOrNot 函数时发生了隐式的类型转换, 除了向方法传入参数之外, 变量的赋值也会触发隐式类型转换. 
> 在类型转换时, * TestStruct 类型会转换成 interface{} 类型, 转换后的变量不仅包含转换前的变量, 还包含变量的类型信息 TestStruct, 所以转换后的变量与 nil 不相等. 

>> 注意: * TestStruct 类型会转换成 interface{} 类型, 转换后的变量不仅包含转换前的变量, 还包含变量的类型信息 TestStruct


# 数据结构

go 根据 接口类型是否包含一组方法将接口类型分为两类:
- 使用 runtime.iface 结构体表示包含方法的接口类型
- 使用 runtime.eface 结构体表示不包含任何方法的接口类型

## runtime.eface:

```go
type eface struct { //16字节
        _type   *_type  //该interface的真实类型
        data    unsafe.Pointer
}
```
由于 interface{} 类型不包含任何方法, 所以它的结构只包含指向底层数据和类型的两个指针. 从上述结构我们也能推断出Go 语言的任意类型都可以转换成 interface{}.

- runtime._ type:

```go
type _type struct {
        size       uintptr
        ptrdata    uintptr
        hash       uint32
        tflag      tflag
        align      uint8
        fieldAlign uint8
        kind       uint8
        equal      func(unsafe.Pointer, unsafe.Pointer) bool
        gcdata     *byte
        str        nameOff
        ptrToThis  typeOff
}
```
- size: 类型占用的内存空间, 为内存空间的分配提供信息
- hash: 类型的hash值, 可以快速帮我们在做类型比对时候确定类型是否相等
- equal:  该字段用于判断该类型的多个对象是否相等



## runtime.iface:

```go
type iface struct{
        tab     *itab   //itab是interface的核心部分
        data    unsafe.Pointer
}

type itab struct { // 32 字节
        inter *interfacetype
        _type *_type
        hash  uint32
        _     [4]byte
        fun   [1]uintptr
}
```

- inter和_type都是用来表示该interface{}的类型.
- hash: 是对_type.hash的拷贝, 当我们需要比对类型时候, 用这个更快, 不需要通过_type.hash比对
- fun： 是一个动态大小的数组, 它是一个用于动态派发的虚函数表, 存储了一组函数指针. 虽然该变量被声明成一个大小为1的固定数组, 但真正使用时候这个长度为1的数组里存放的是一个动态数组的地址.



