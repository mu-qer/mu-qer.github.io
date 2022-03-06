---
layout: post
title: kernel list
date: 2021-03-06 23:30:09
categories: Kernel
description: Kernel list
tags: Kernel
---

# list

linux内核中的list数据结构的用法和我们平时学习到的list是优点不同的，例如:

```c
//平时学习遇到的
struct fox {
	unsigned int	fid;
	unsigned int	age;

	struct fox 		*next, *prev;
};

//kernel list:
struct fox {//这里将 fox称为 list的owner
	unsigned int	fid;
	unsigned int 	age;

	struct list_head list;
};

struct list_head {
	struct list_head	*next, *prev;
};
```

这样做的好处是： struct list_head可以在多种自定义结构体中实现代码复用.
以下分析的均为 kernel list.


# kernel list 如何找到owner对象的地址

在 linux/include/linux/list.h 中, 发现全部操作都是针对 list_head的, 但是我们知道了 list_head的地址, 怎么找到其对应的 owner的地址呢？因为平时使用时候都是用的owner的地址，而不是里边的list的地址！！

list_head作为 fox的一个成员, 当我们知道了 list_head的地址, 可以通过宏 list_entry 来获取它父结构的地址：

```c
#define list_entry(ptr, type, member) container_of(ptr, type, member)

#define container_of(ptr, type, member) ({ \
			const typeof( ((type*)0)->member ) *__mptr=(ptr);
			(type*)( (char*)__mptr - offsetof(type, member));})

#define offsetof(TYPE, MEMBER)	((size_t)&((TYPE*)0)->MEMBER)
```

宏 offsetof：
```c
#define offsetof(TYPE, MEMBER)	((size_t)&((TYPE*)0)->MEMBER)
/*
分解一下后半句: (size_t)&((TYPE*)0)->MEMBER, 按符号优先级进行拆分：
1. (TYPE*)0: 把0(地址,即 NULL)强转成TYPE*, 你把0换成NULL也是可以的.
2. ((TYPE*)0)->MEMBER: 在这个临时构建出的地址结构中(以NULL地址为首地址构建的), 找到MEMBER成员
3. 对找到的这个MEMBER成员取地址： &((TYPE*)0)->MEMBER
4. 把这个地址转成 size_t类型： (size_t)&((TYPE*)0)->MEMBER

可以简单理解为：在0地址的地方构建了一个临时的TYPE对象, 并通过返回MEMBER成员的地址来确定该MEMBER成员的地址距离0(TYPE对象的起始地址)的距离, 即Offset.
*/
```
宏 container_of：
```c
#define container_of(ptr, type, member) ({ \
			const typeof( ((type*)0)->member ) *__mptr=(ptr);
			(type*)( (char*)__mptr - offsetof(type, member));})
/*
首先理解一下全貌, 可以看到ptr被转换成一个临时对象 __mptr, 并最终使用 __mptr来计算当前member距离type对象起始地址的offset, 即地址。 如果ptr是一个普通的ptr, 而不是表达式, 比如: ptr++, ++ptr等, 那么宏 container_of将改写为:
*/

#define container_of(ptr, type, member) ( (type*)( (char*)ptr - offsetof(type,member)) )

/*
ptr之所以要转换成char*, 因为地址是以字节为单位的, 而char的长度就是一个字节.
__mptr是结构体type中的list_head的地址, offsetof求的是member在type结构体中的偏移量, 那么 __mptr减去offsetof得到的就是 member的地址了！！
*/
```

# list interface

## 创建链表

```c
#define LIST_HEAD_INIT(name) { &(name), &(name) }
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)

/**
 * INIT_LIST_HEAD - Initialize a list_head structure
 * @list: list_head structure to be initialized.
 *
 * Initializes the list_head to point to itself.  If it is a list header,
 * the result is an empty list.
 */
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	WRITE_ONCE(list->next, list);
	list->prev = list;
}
```
可以通过 LIST_HEAD(mylist)进行初始化一个链表, mylist的prev 和 next指针都指向自己。
这只是一个空链表而已, 并且不具有有效字段. 我们可以这样初始化一个含有有效字段的链表：

```c
struct fox f = {
	.id = 1,
	.age = 2,
	.list_head = LIST_HEAD_INIT(f.list_head)
};
```
这样就行了。


具体内核list代码：https://github.com/torvalds/linux/blob/master/include/linux/list.h
具体多余的分析：https://blog.csdn.net/wanshilun/article/details/79747710

