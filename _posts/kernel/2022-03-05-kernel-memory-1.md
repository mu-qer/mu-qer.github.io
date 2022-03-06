---
layout: post
title: memory management
date: 2022-03-05 23:30:09
categories: Kernel
description: memory manage/malloc/free/merge
tags: Kernel
---

# 虚拟内存

## 概述,虚拟内存做了哪些事

- 给每个进程分配完全独立的虚拟空间，每个进程终于有只属于自己的活动场地了
- 进程使用的虚拟空间最终还要落到物理内存上，因此设置了一套完善的虚拟地址和物理地址的映射机制
- 引入缺页异常机制实现内存的惰性分配，啥时候用啥时候再给
- 引入swap机制把不活跃的数据换到磁盘上，让每块内存都用在刀刃上
- 引入OOM机制在内存紧张的情况下干掉那些内存杀手
- ......


## 虚拟内存的引入

引入虚拟内存之前, cpu访问的是物理内存，具体过程如下：
```
	cpu <---(call)---  物理内存(main memory)  <---(load/swap)--->  磁盘
```

引入虚拟内存之后, CPU并不再直接和物理内存打交道，而是把地址转换的活外包给了MMU，MMU是一种硬件电路，其速度很快，主要工作是进行内存管理，地址转换只是它承接的业务之一。
```
	cpu -----> (va到pa的转换) ------> mmu
```

### mmu && pageTable && TLB 

每个进程都会有自己的页表Page Table，页表存储了进程中 <虚拟地址> 到 <物理地址> 的映射关系，所以就相当于一张地图，MMU收到CPU的虚拟地址之后开始查询页表，确定是否存在映射以及读写权限是否正常，如图:

![mmu](https://mu-qer.github.io/assets/img/kernel/2022-03-05-mmu.JPG)

当机器的物理内存越来越大，页表这个地图也将非常大，于是问题出现了:
- 对于4GB的虚拟地址且大小为4KB页，一级页表将有2^20个表项，页表占有连续内存并且存储空间大
- 多级页表可以有效降低页表的存储空间以及内存连续性要求，但是多级页表同时也带来了查询效率问题

我们以2级页表为例，MMU要先进行两次页表查询确定物理地址，在确认了权限等问题后，MMU再将这个物理地址发送到总线，内存收到之后开始读取对应地址的数据并返回。

![multi-pagetable-search](https://mu-qer.github.io/assets/img/kernel/2022-03-05-multi-pagetable-search.JPG)

MMU在2级页表的情况下进行了2次检索和1次读写.
> 当页表变为N级时，就变成了N次检索 + 1次读写

可见，页表级数越多查询的步骤越多，对于CPU来说等待时间越长，效率越低，这个问题还需要优化才行. 于是快表 TLB 出现了.

![tlb](https://mu-qer.github.io/assets/img/kernel/2022-03-05-tlb.JPG)

可以认为TLB是ptagetable在mmu中的缓存. 当CPU给MMU传新虚拟地址之后, MMU先去问TLB那边有没有, 如果有就直接拿到物理地址发到总线给内存。 没有的话 MMU还有保底的老武器页表 Page Table，在页表中找到之后MMU除了把地址发到总线传给内存，还把这条映射关系给到TLB，让它记录一下刷新缓存。

![cpu-read-mem](https://mu-qer.github.io/assets/img/kernel/2022-03-05-cpu-read-mem.JPG)


### page fault

假如目标内存页在物理内存中没有对应的页帧或者存在但无对应权限，CPU 就无法获取数据，这种情况下CPU就会报告一个page fault.
由于CPU没有数据就无法进行计算, CPU罢工了用户进程也就出现了缺页中断，进程会从用户态切换到内核态，并将缺页中断交给内核的 Page Fault Handler 处理. 过程如图：

![page-fault](https://mu-qer.github.io/assets/img/kernel/2022-03-05-page-fault.JPG)

缺页中断会交给PageFaultHandler处理，其根据缺页中断的不同类型会进行不同的处理：

> - Hard Page Fault
>> 也被称为Major Page Fault，翻译为硬缺页错误/主要缺页错误，这时物理内存中没有对应的页帧，需要CPU打开磁盘设备读取到物理内存中，再让MMU建立VA和PA的映射。

> - Soft Page Fault
>> 也被称为Minor Page Fault，翻译为软缺页错误/次要缺页错误，这时物理内存中是存在对应页帧的，只不过可能是其他进程调入的，发出缺页异常的进程不知道而已，此时MMU只需要建立映射即可，无需从磁盘读取写入内存，一般出现在多进程共享内存区域.

> - Invalid Page Fault
>> 翻译为无效缺页错误，比如进程访问的内存地址越界访问，又比如对空指针解引用内核就会报segment fault错误中断进程直接挂掉。

总结如图：

![page-fault-handler](https://mu-qer.github.io/assets/img/kernel/2022-03-05-page-fault-handler.JPG)


常见的产生page fault的原因如下：
- 访问越界: 比如空指针解引用或者权限问题等都会出现缺页错误
- 使用malloc申请内存： malloc机制是延时分配内存，当使用malloc申请内存时并未真实分配物理内存，等到真正开始使用malloc申请的物理内存时发现没有才会启动申请，期间就会出现Page Fault
- 访问数据被swap换出： 物理内存是有限资源，当运行很多进程时并不是每个进程都活跃，对此OS会启动内存页面置换将长时间未使用的物理内存页帧放到swap分区来腾空资源给其他进程，当存在于swap分区的页面被访问时就会触发Page Fault从而再置换回物理内存


## 虚拟内存的分配

虚拟机制下每个进程都有独立的地址空间，并且地址空间被划分为了很多部分，如图为32位系统中虚拟地址空间分配：

![process-memory-struct](https://mu-qer.github.io/assets/img/kernel/2022-03-05-process-memory-struct.JPG)

各段特点和联系：

- text段存放程序的二进制代码，所以又被称为代码段，在32位和64位系统中代码段的起始地址都是确定的，并且大小也是确定的。
- data段存放的是已经初始化的全局变量和静态变量, 和text段紧挨着，中间无空隙，因此起始地址也是固定的，大小也是确定的。
- bss段存储未初始化的全局变量，和data段紧挨着，中间没有空隙，因此起始地址也是固定的，大小也是确定的。
- heap段和bss段并不是紧挨着的，中间会有一个随机的偏移量，heap段的起始地址也被称为start_brk，由于heap段是动态的，顶部位置称为program break brk.
- 在heap段上方是内存映射段，该段是mmap系统调用映射出来的，该段的大小也是不确定的，并且夹在heap段和stack段中间，该段的起始地址也是不确定的。
- stack段算是用户空间地址最高的一部分了，它也并没有和内核地址空间紧挨着，中间有随机偏移量，同时一般stack段会设置最大值RLIMIT_STACK(比如8MB)，在之下再加上一个随机偏移量就是内存映射段的起始地址了.

总结：
> 总结：
>> - 进程虚拟空间的各个段，并非紧挨着，也就是有的段的起始地址并不确定，大小也并不确定
>> - 随机的地址是为了防止黑客的攻击，因为固定的地址被攻击难度低很多

我把heap段、stack段、mmap段再细化一张图:

![process-memory-struct-2](https://mu-qer.github.io/assets/img/kernel/2022-03-05-process-memory-struct-2.JPG)


## 内存的组织方式

从前面可以看到进程虚拟空间就是一块块不同区域的集合，这些区域就是我们上面的段，每个区域在Linux系统中使用vm_area_struct这个数据结构来表示的。

内核为每个进程维护了一个单独的任务结构task_strcut，该结构中包含了进程运行时所需的全部信息，其中有一个内存管理(memory manage)相关的成员结构mm_struct：

```c++
struct mm_struct    *mm;
struct mm_struct    *active_mm;
```

结构mm_strcut的成员非常多，其中gpd和mmap是我们需要关注的:
- pgd指向第一级页表的基地址，是实现虚拟地址和物理地址的重要部分
- mmap指向一个双向链表，链表节点是vm_area_struct结构体，vm_area_struct描述了虚拟空间中的一个区域
- mm_rb指向一个红黑树的根结点，节点结构也是vm_area_struct

![mm_struct](https://mu-qer.github.io/assets/img/kernel/2022-03-05-mm_struct.JPG)

看下vm_area_struct的结构体定义:

![vm_area_struct](https://mu-qer.github.io/assets/img/kernel/2022-03-05-vm_area_struct.JPG)

vm_area_start作为链表节点串联在一起，每个vm_area_struct表示一个虚拟内存区域，由其中的vm_start和vm_end指向了该区域的起始地址和结束地址，这样多个vm_area_struct就将进程的多个段组合在一起了。

![vm_area_list](https://mu-qer.github.io/assets/img/kernel/2022-03-05-vm_area_list.JPG)

我们同时注意到vm_area_struct的结构体定义中有rb_node的相关成员，不过有的版本内核是AVL-Tree，这样就和mm_struct对应起来了：

![vm_area_rb_node](https://mu-qer.github.io/assets/img/kernel/2022-03-05-vm_area_rb_node.JPG)

这样vm_area_struct通过双向链表和红黑树两种数据结构串联起来，实现了两种不同效率的查找，双向链表用于遍历vm_area_struct，红黑树用于快速查找符合条件的vm_area_struct。


# 内存分配器

有内存分配和回收的地方就可能有内存分配器。
以glibc为例，我们先捋一下：
- 在用户态层面，进程使用库函数malloc分配的是虚拟内存，并且系统是延迟分配物理内存的，由缺页中断来完成分配
- 在内核态层面，内核也需要物理内存，并且使用了另外一套不同于用户态的分配机制和系统调用函数

从而就引出了，今天的主线图：

![mem_alloc_view1](https://mu-qer.github.io/assets/img/kernel/2022-03-05-mem_alloc_view1.JPG)


从图中我们来阐述几个重点：

- 伙伴系统和slab属于内核级别的内存分配器，同时为内核层面内存分配和用户侧面内存分配提供服务，算是终极boss
- 内核有自己单独的内存分配函数kmalloc/vmalloc，和用户态的不一样
- 用户态的进程通过库函数malloc来玩转内存，malloc调用了brk/mmap这两个系统调用，最终触达到伙伴系统实现内存分配
- 内存分配器分为两大类：用户态和内核态，用户态分配和释放内存最终还是通过内核态来实现的，用户态分配器更加贴合进程需求

## 常见的用户态分配器

分配器响应进程的内存分配请求，向操作系统申请内存，找到合适的内存后返回给用户程序，当进程非常多或者频繁内存分配释放时，每次都找内核老大哥要内存/归还内存，可以说十分麻烦。 于是分配器决定自己搞管理！

- 分配器一般都会预先分配一块大于用户请求的内存，然后管理这块内存
- 进程释放的内存并不会立即返回给操作系统，分配器会管理这些释放掉的内存从而快速响应后续的请求

常见的用户态分配器：
- 1. ptmalloc2 : 它也是目前glibc中使用的默认分配器, 不过后续各自都有不同的修改, 因此ptmalloc2和glibc中默认分配器也并非完全一样.

- 2. tcmalloc, 全称是 thread-caching malloc, 所以 tcmalloc 最大的特点是带有线程缓存，tcmalloc 非常出名，目前在 Chrome、Safari 等知名产品中都有所应有。
tcmalloc 为每个线程分配了一个局部缓存，对于小对象的分配，可以直接由线程局部缓存来完成，对于大对象的分配场景，tcmalloc 尝试采用自旋锁来减少多线程的锁竞争问题。

- 3. jemalloc,它是一个通用的 malloc 实现，侧重于减少内存碎片和提升高并发场景下内存的分配效率，其目标是能够替代 malloc.
jemalloc 应用十分广泛，在 Firefox、Redis、Rust、Netty 等出名的产品或者编程语言中都有大量使用

## glibc malloc原理分析

我们在使用malloc进行内存分配，malloc只是glibc提供的库函数，它仍然会调用其他函数从而最终触达到物理内存，所以是个很长的链路。

我们先看下malloc的特点：

- malloc 申请分配指定size个字节的内存空间，返回类型是 void* 类型, 但是此时的内存只是虚拟空间内的连续内存, 无法保证物理内存连续
- mallo并不关心进程用申请的内存来存储什么类型的数据，void* 类型可以强制转换为任何其它类型的指针, 从而做到通用性

继续我看下 malloc是如何触达到物理内存的：

```c++
#include <unistd.h>
int brk (void *addr);
void* sbrk(intptr_t increment);
```

- brk函数将break指针直接设置为某个地址，相当于绝对值
- sbrk将break指针从当前位置移动increment所指定的增量，相当于相对值
- 本质上brk和sbrk作用是一样的都是移动break指针的位置来扩展内存

```c++
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
```

- mmap和munmap是一对函数，一个负责申请，一个负责释放
- mmap有两个功能：实现文件映射到内存区域 和 分配匿名内存区域，在malloc中使用的就是匿名内存分配，从而为程序存放数据开辟空间

### malloc数据结构

malloc维护了一个含有128个bin的数组, 数组名‘bin_array’. 每一个bin是一个双向链表，很多个相同大小的chunk块被连接在同一个bin上。

1. 通用的bins

![malloc_bins](https://mu-qer.github.io/assets/img/kernel/2022-03-05-malloc_bins.JPG)

- bins[0]目前没有使用
- bins[1]的链表称为unsorted_list，用于维护free释放的chunk。
- bins[2,63]总计长度为62的区间称为small_bins, 用于维护＜512B的内存块, 其中每个bin中对应的链表中的chunk大小相同, 相邻bin的大小相差8字节, 范围为16字节到504字节.
- bins[64,126]总计长度为63的区间称为large_bins, 用于维护大于等于512字节的内存块, 每个元素对应的链表中的chunk大小不同, 数组下标越大链表中chunk的内存越大，large bins中的每一个bin分别包含了一个给定范围内的chunk，其中的chunk按大小递减排序，最后一组的largebin链中的chunk大小无限制，该bins的使用频率低于small bins。

![malloc_bins_internal](https://mu-qer.github.io/assets/img/kernel/2022-03-05-malloc_bins_internal.JPG)

2. 特殊的bins

> - fast bin
>> - malloc对于释放的内存并不会立刻进行合并，如何将刚释放的两个相邻小chunk合并为1个大chunk，此时进程分配仍然是小chunk则可能还需要分割大chunk，来来回回确实很低效，于是出现了fast bin。 
>> - fast bin存储在fastbinY数组中，一共有10个，每个fast bin都是一个单链表，每个单链表中的chunk大小是一样的，多个链表的chunk大小不同，这样在找特定大小的chunk的时候就不用挨个找，只需要计算出对应链表的索引即可，提高了效率. 多个fast bin链表存储的chunk大小有16, 24, 32, 40, 48, 56, 64, 72, 80, 88字节总计10种大小。
>> - fast bin是除tcache外优先级最高的，如果fastbin中有满足需求的chunk就不需要再到small bin和large bin中寻找。当在fast bin中找到需要的chunk后还将与该chunk大小相同的所有chunk放入tcache，目的就是利用局部性原理提高下一次内存分配的效率。
>> - 对于不超过max_fast的chunk被释放后，首先会被放到 fast bin中，当给用户分配的 chunk 小于或等于 max_fast 时，malloc 首先会在 fast bin 中查找相应的空闲块，找不到再去找别的bin。

> - unsorted bin
>> - 当用户释放的内存大于max_fast或者fast bins合并后的chunk都会首先进入unsorted bin上，unsorted bin中的chunk大小没有限制，相当于malloc给了最近被释放的内存被快速二次利用的机会，在内存分配的速度上有所提升。 
>> - 在进行 malloc 操作的时候，如果在 fast bins 中没有找到合适的 chunk，则malloc 会先在 unsorted bin 中查找合适的空闲 chunk. unsorted bin里面的chunk是最近回收的，但是并不能全部再被快速利用，因此在遍历unsorted bins的过程中会把不同大小的chunk再分配到small bins或者large bins。

> - top chunk
>> - top chunk不属于任何bin，它是始终位于堆内存的顶部。
当所有的bin里的chunk都无法满足分配要求时，malloc会从top chunk分配内存，如果大小不合适会进行分割，剩余部分形成新的top chunk。
如果top chunk也无法满足用户的请求，malloc只能向系统申请更多的堆空间，所以top chunk可以认为是各种bin的后备力量，尤其在分配大内存时，large bins也无法满足时大哥就得顶上了。

> - last remainder chunk
>> - 当unsorted bin只有1个chunk，并且这个chunk是上次刚刚被使用过的内存块，那么它就是last remainder chunk。当进程分配一个small chunk，在small bins中找不到合适的chunk，这时last remainder chunk就上场了。 如果last remainder chunk大于所需的small chunk大小，它会被分裂成两个chunk，其中一个chunk返回给用户，另一个chunk变成新的last remainder chunk。这种特殊chunk主要用于分配内存非常小的情况下，当fast bin和small bin都无法满足时，还会再次从last remainder chunk进行分配，这样就很好地利用了程序局部性原理。

### malloc内存分配流程

![malloc_flow_chart](https://mu-qer.github.io/assets/img/kernel/2022-03-05-malloc_flow_chart.JPG)

在上图中有几个点需要说明：

- 内存释放后，size小于max_fast则放到fast bin中，size大于max_fast则放到unsorted bin中，fast bin和unsorted bin可以看作是刚释放内存的容器，目的是给这些释放内存二次被利用的机会。

- fast bin中的fast chunk会定期合并到unsorted bin中。

- unsorted bin很特殊，可以认为是个中间过渡bin，在large bin分割chunk时也会将下脚料chunk放到unsorted bin中等待后续合并放入large bin中 或者 再分配到small bin中。

- 由于small bin和large bin链表很多并且大小各不相同，遍历查找合适chunk过程是很耗时的，为此引入binmap结构来加速查找，binmap记录了bins的是否为空等情况，可以提高效率。

当用户申请的内存比较小时，分配过程会比较复杂，我们再尝试梳理下该情况下的分配流程：

![mallocsmallchunk_flow_chart](https://mu-qer.github.io/assets/img/kernel/2022-03-05-mallocsmallchunk_flow_chart.JPG)

- <1>. 将进程需要分配的内存转换为对应空闲内存块的大小, 记做chunk_size
- <2>. 当chunk_size小于等于max_fast，则在fast bin中搜索合适的chunk，找到则返回给用户，否则跳到第3步
- <3>. 当chunk_size<=512字节，那么可能在small bin的范围内有合适的chunk，找到合适的则返回，否则跳到第4步
- <4>. 在fast bin和small bin都没有合适的chunk，那么就对fast bin中的相邻chunk进行合并，合并后的更大的chunk放到unsorted bin中，跳转到第5步
- <5>. 如果chunk_size属于small bins，unsorted bin 中只有一个 chunk，并且该 chunk 大于等于需要分配的大小，此时将该 chunk 进行切割，一部分返回给用户，另外一部分形成新的last remainder chunk分配结束，否则将 unsorted bin 中的 chunk 放入 small bins 或者 large bins，进入第6步
- <6>. 现在看chunk_size属于比较大的，因此在large bins进行搜索，满足要求则返回，否则跳到第7步
- <7>. 至此fast bin和另外三组bin都无法满足要求，就轮到top chunk了，在top chunk满足则返回，否则跳到第8步
- <8>. 如果chunk_size大于等于mmap分配阈值，使用mmap向内核伙伴系统申请内存，chunk_size小于mmap阈值则使用brk来扩展top chunk满足要求

> 特别地，搜索合适chunk的过程中，fast bins 和small bins需要大小精确匹配，而在large bins中遵循“smallest-first，best-fit”的原则，不需要精确匹配，因此也会出现较多的碎片。

## 内存回收

内存回收就是释放掉比如缓存和缓冲区的内存，通常他们被称为文件页page cache，对于通过mmap生成的用于存放程序数据而非文件数据的内存页称为匿名页。

- 文件页 有外部的文件介质形成映射关系
- 匿名页 没有外部的文件形成映射关系

### 文件页回收

page cache常被用于缓冲磁盘文件的数据，让磁盘数据放到内存中来实现CPU的快速访问。

page cache中有非常多page frame，要回收这些page frame需要确定这些物理页是否还在用，为了解决这个问题出现了反向映射技术。
正向映射是通过虚拟地址根据页表找到物理内存，反向映射就是通过物理地址找到哪些虚拟地址使用它，也就是当我们在决定page frame是否可以回收时，需要使用反向映射来查看哪些进程被映射到这块物理页了，进一步判断是否可以回收。

找到可以回收的page frame之后内核使用LRU算法进行回收，Linux采用的方法是维护2个双向链表，一个是包含了最近使用页面的active list，另一个是包含了最近不使用页面的inactive list。

- active_list 活跃内存页链表，这里存放的是最近被访问过的内存页，属于安全区。
- inactive_list 不活跃内存页链表，这里存放的是很少被访问的内存页，属于毒区。

### 匿名页回收

匿名页没有对应的文件形成映射，因此也就没有像磁盘那样的低速备份。
在回收匿名页的时候，需要先保存匿名页上的内容到特定区域，这样才能避免数据丢失保证后续的访问。

匿名页在进程中是非常普遍的，动态分配的堆内存都可以说是匿名页，Linux为回收匿名页，特地开辟了swap space来存储内存上的数据。内核倾向于回收page cache中的物理页面，只有当内存很紧张并且内核配置允许swap机制时，才会选择回收匿名页。
回收匿名页意味着将数据放到了低速设备，一旦被访问性能损耗也很大，因此现在大内存的物理机器经常关闭swap来提高性能。

### kswaped线程和waterMark

NUMA架构下每个CPU都有自己的本地内存来加速访问避免总线拥挤，在本地内存不足时又可以访问其他Node的内存，但是访问速度会下降。

![numa-mem](https://mu-qer.github.io/assets/img/kernel/2022-03-05-numa-mem.JPG)

每个CPU加本地内存被称作Node，一个node又被划分为多个zone，每个zone有自己一套内存水位标记，来记录本zone的内存水平，同时每个node有一个kswapd内核线程来回收内存。

Linux内核中有一个非常重要的内核线程kswapd，负责在内存不足的情况下回收页面，系统初始化时，会为每一个NUMA内存节点创建一个名为kswapd的内核线程。

在内存不足时内核通过wakeup_kswapd()函数唤醒kswapd内核线程来回收页面，以便释放一些内存，kswapd的回收方式又被称为background reclaim。

Linux内核使用水位标记（watermark）的概念来描述这个压力情况。

![mem-free](https://mu-qer.github.io/assets/img/kernel/2022-03-05-mem-free.JPG)

他们所标记的分别含义为：

- 水位线在high以上表示内存剩余较多，目前内存使用压力不大，kswapd处于休眠状态

- 水位线在high-low的范围表示目前虽然还有剩余内存但是有点紧张，kswapd开始工作进行内存回收

- 水位线在low-min表示剩余可用内存不多了压力山大，min是最小的水位标记，当剩余内存达到这个状态时，就说明内存面临很大压力。

- 水位线低于min这部分内存，就会触发直接回收内存。

