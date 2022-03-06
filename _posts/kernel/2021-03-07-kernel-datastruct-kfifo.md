---
layout: post
title: kernel kfifo
date: 2021-03-07 23:30:09
categories: Kernel
description: Kernel kfifo
tags: Kernel
---

# kfifo

kfifo是内核里面的一个环形队列, 它提供一个无边界的字节流服务, 并且使用并行无锁编程技术, 即当它用于只有一个入队线程和一个出队线程的场情时, 两个线程可以并发操作, 而不需要任何加锁行为, 就可以保证kfifo的线程安全。

```c
struct kfifo {
    unsigned char	*buffer;    /* the buffer holding the data */
    unsigned int	size;    	/* the size of the allocated buffer */
    unsigned int	in;			/* data is added at offset (in % size) */
    unsigned int	out;    	/* data is extracted from off. (out % size) */
    spinlock_t 		*lock;    	/* protects concurrent modifications */
};

/*
buffer: 用于存放数据的缓存
size:	buffer空间的大小, 在初始化时, 将它扩展成2的幂
in,out:	和buffer一起构成一个循环队列, in指向buffer中的队头, out指向buffer中的队尾
lock:	如果使用不能保证任何时间最多只有一个读线程和写线程, 需要使用该lock实施同步
*/

关系如图：
+--------------------------------------------------------------+
|            |<----------data---------->|                      |
+--------------------------------------------------------------+
             ^                          ^                      ^
             |                          |                      |
            out                        in                     size
/*
in,out的类型都是 unsigned int, 2^32, 也是2的幂
以下讲解分为两部分：
1. out <= in < unsigned int
2. out > in, in的大小超过unsigned int的大小,溢出并环回到0+x的位置

以下所有情况均假设是: out <= in < unsigned int
注意上图： in始终在out右边, 即out的值始终小于in. 因此 (in - out)就是有效数据空间的大小, size是总空间大小,那么剩余空间大小就是： size - (in - out) = size - in + out

in表示可以写入数据的位置(offset), out表示可以从该out处读取数据.
in和out都是虚拟索引, 因为in和out的大小都有可能超越size, 类似缓冲区溢出的样子.
但 kfifo在读写数据时候使用取模 kfifo->in & (kfifo->size - 1) 或者 kfifo->out & (kfifo->size - 1)来获取数据真正的写入和读取位置。
*/
```
kfifo只提供了两个操作:
1. __kfifo_put, 入队操作
2. __kfifo_get, 出队操作


kfifo的读写要求:
1. 只支持一个读者和一个写者并发操作
2. 无阻塞的读写操作, 如果空间不够, 则返回实际访问空间???

# kfifo 的初始化

```c
struct kfifo *kfifo_alloc(unsigned int size, gfp_t gfp_mask, spinlock_t *lock)
{
    unsigned char *buffer;
    struct kfifo *ret;

    /*
     判断size是否为2的幂,如果不是,则将size自动向上扩展成2的幂的大小。
     kernel中将数据向上扩展到2的幂是惯用法,为了简化取模运算.
     */
    if (size & (size - 1)) {
        BUG_ON(size > 0x80000000);
        size = roundup_pow_of_two(size);
    }

    buffer = kmalloc(size, gfp_mask);
    if (!buffer)
        return ERR_PTR(-ENOMEM);

    ret = kfifo_init(buffer, size, gfp_mask, lock);

    if (IS_ERR(ret))
        kfree(buffer);

    return ret;
}
```

# kfifo 的入队和出队

```c
/*
    经过 kfifo_alloc这样之后, buffer的size已经是2的幂大小. 这时候在 % 和 & 之间的转换有一个公式可以使用：
    a % b = a & (b - 1) //b是2的幂次方, &计算要更快

    使用 %, 取模 4%64, 一亿次计算， 150ms
    使用 &, 取模 4 & (64-1), 一亿次计算， 50ms
*/
```


kfifo_put是入队操作, 它先将数据放入buffer里, 最后才修改in参数;
kfifo_get的出队操作, 它先将数据从buffer中移走, 最后才修改out.

## 入队列

```c
unsigned int __kfifo_put(struct kfifo *fifo, unsigned char *buffer, unsigned int len)
{
    unsigned int l;
    // fifo->size - fifo->in + fifo->out: 用来得到剩余可写空间的大小
    len = min(len, fifo->size - fifo->in + fifo->out);

    /* Ensure that we sample the fifo->out index -before- we
     * start putting bytes into the kfifo.
     */
    smp_mb();

    /* first put the data starting from fifo->in to buffer end 
     * 计算fifo可写入的空间长度和len的最小值作为拷贝的长度,防止溢出
     * fifo->in & (fifo->size - 1)得到的结果就是in真正需要写入数据的偏移offset或index, 假设叫real_in 
     * real_in = fifo->in & (fifo->size - 1) 
     */
    l = min(len, fifo->size - (fifo->in & (fifo->size - 1)));
    /*
    * 从buffer中拷贝l字节长度的数据到 fifo->buffer中, 从real_in位置写入
      这个长度为l的数据写在 in位置之后
    */
    memcpy(fifo->buffer + (fifo->in & (fifo->size - 1)), buffer, l);

    //这个 len - l 长度数据写在 fifo->buffer的开头处, 这里必定至少有 len-l长度的空闲空间
    memcpy(fifo->buffer, buffer + l, len - l);

    /* Ensure that we add the bytes to the kfifo -before-
     * we update the fifo->in index.
     */
    smp_wmb();

    /*
        这一步很关键, 每次调用该函数, fifo->in都会加上len长度, 那么很自然随后某一时刻必定会超越 size大小,
        因此, in就是一个虚拟的写索引位置, 真正的写索引位置是: fifo->in & (fifo->size -1) 
    */
    fifo->in += len;
    return len;
}
```

## 出队列

```c
unsigned int __kfifo_get(struct kfifo *fifo, unsigned char *buffer, unsigned int len)
{
    unsigned int l;

    //计算有效数据的长度 in - out
    len = min(len, fifo->in - fifo->out);

    /*
     * Ensure that we sample the fifo->in index -before- we
     * start removing bytes from the kfifo.
     */

    smp_rmb();

    /* right_len = fifo->size - (fifo->out & (fifo->size -1)): 即为从 out开始向右到size的长度
       因此 l <= right_len, l = min(len, right_len) 
    */
    l = min(len, fifo->size - (fifo->out & (fifo->size - 1)));
    //先从real_out处拷贝长度为l的数据写入buffer中
    memcpy(buffer, fifo->buffer + (fifo->out & (fifo->size - 1)), l);

    //如果 len - l > 0,说明fifo->buffer开头处还有长度为 len-l 的数据需要拷贝
    memcpy(buffer + l, fifo->buffer, len - l);

    /*
     * Ensure that we remove the bytes from the kfifo -before-
     * we update the fifo->out index.
     */

    smp_mb();

    fifo->out += len;
    return len;
}
```

## unsigned int回绕特性的利用

kfifo每次入队或出队, kfifo->in/out只是简单地kfifo->in/out += len，并没有对kfifo->size 进行取模运算. 因此kfifo->in和kfifo->out总是一直增大, 直到unsigned in最大值时, 又会绕回到0这一起始端. 
当 kfifo->in 回绕到了0的那一端时候, 代码中计算长度的性质仍然是正确的.

```c
//一般情况：
+------------------------+ 
|        |<---data--->|  | 
+------------------------+ 
         ^            ^ 
         |            | 
        out          in

//当有数据入队时, 那么in的值可能超过kfifo->size的值, 那么我们使用另一个虚拟的方框来表示in变化后, 在buffer内对kfifo->size取模的值:

//真实的buffer长度          |//虚拟的buffer长度, 方便查看in对kfifo->size取模后在buffer的下标 
+------------------------+ +----------------------+
|        |<----------data--------->|              |
+------------------------+ +----------------------+
         ^                         ^ 
         |                         | 
        out                       in

/*如果fifo->in超过了unsigned int的最大值时,而回绕到0这一端,上述的计算公式还正确吗? 答案是正确的。

因为fifo->size的大小是2的幂, 而unsigned int空间的大小是2^32, 后者刚好是前者的倍数. 
如果从上述两个图的方式来描述, 则表示unsigned int空间的数轴上刚好可以划成若干数量个kfifo->size大小方框, 没有长度多余. 因此 fifo->in/fifo->out对fifo->size取模后, 刚好落在原buffer的对应位置上。
*/

//现在假设继续向 kfifo中写入数据, 使得满足 kfifo->in < kfifo->out：
+-------------------------------+ 
|--------->|         |<----data-| 
+-------------------------------+ 
           ^         ^
           |         | 
          in        out
/*
    假设 kfifo中数据的长度为 data_len, 那么 kfifo->in 和 kfifo->out的关系：
    kfifo->in = kfifo->out + data_len && kfifo->in < kfifo->out
    这说明in回绕到0的一端了, 但, in - out依然等于 data_len.
    此时的可用空间为: size - data_len = fifo->size - (fifo->in - fifo->out)
    因此, 无论 in和out谁大谁小, 
    1. 计算kfifo剩余空间的大小的公式：fifo->size - fifo->in + fifo->out都是正确的.
    2. 计算有效数据空间的大小的公式： fifo->in - fifo->out都是正确的.
*/
```




## kfifo的无锁编程技术

kfifo使用 in 和 out 两个指针来描述写入和读取索引, 对于写入操作只更新in, 读取只更新out, 两者互不干扰. 为了避免读写out看到写者in预计写入但还未写入的数据空间的问题, 这两个接口都是在最后更新索引值 fifo->in 和 fifo->out的。