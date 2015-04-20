---
layout: post
title: "CPU Cache Line Theory"
date: 2015-04-20 22:16:24 +0800
comments: true
categories: 
description: 
    - Tech
keywords: CPU Cache Line Theroy 设计
redirect_from: /p/20150420/
---

### CPU Cache的意义

CPU的运算能力指数级增长，以至于无法再与内存的读写速率匹配，因为CPU直接读写内存运算时会使性能大大折扣，因此在CPU上设计一层Cache来缓存是用频率较高的数据和指令可以环节与物理内存速率不匹配导致的性能问题。

<!-- more -->

###Cache实现方案

Cache又有`L1 L2 L3`逐层速度变慢，先把Cache简单的看作是比物理内存更快的内存，其作用是为了更快的与CPU打交道，但是Cache容量相对物理内存很小，因此如何有效和合理的利用这宝贵的空间就需要动下脑筋了。

CPU总是在不停的运算，不停的使用各种数据，在Cache中需要知道读取某个地址内存应该在Cache中如何“寻址”，Cache中某个的数据是不是真正要用的内存数据。因此Cache会有一个基本的数据结构，其最小的单位叫做Cache Line，说白了就是把`Cache`等分后数据单元，大小跟系统的位数相关。`Cache Line` 的结构是

```C
struct Cache_line
{
	TAG;
	Data BLock;
	Flag Bits;
};
```

这里用一个结构体来表示他的结构.`data bolck`存放`Cache line`中缓存过来的物理内存的数据，`Tag`表示当前保存的数据在物理内存中这一段物理内存地址的前缀，`flag bits`表示Cache操作过程中涉及到的标志位。
####1.全关联(full associative cache)

这种“寻址”方式的设计思路为，将内存地址看作是`TAG+OFFSET`，高`N`位作为`TAG`，低`M`位作为`offset`，在`Cache line`中`TAG`就表示了该地址的高N位，通过低`M`位作为`offset`找到在该`Cache line`中的偏移，这样子设计其优点是总是可以尽最大可能的找到没有利用的Cache存取数据，只要遍历所有`Cache line`总是能把所有`Cache`利用起来，缺点是“寻址”过程靠的是枚举Cache Line会导致时间成本变高(时间代价`O(n)`)，实现详情见[full associative cache][]。
要是能够让内存地址片段能够一一对应到指定“地址”的`Cache Line`就可以解决CPU 遍历`Cache Line` 带来的时间见代价，于是有一个另一个极端`直接关联型`Cache`

####2.直接关联(direct mapped cache)

简单的理解起来直接关联就像是一个map，`n Mb` 的CPU cache 对应`m Mb`大小的物理存储，总是有固定的对应关系。其表现就是需要 `l-k`这段地址上的数据，那他一定是在CPU cache的`a-b`上。这个实现简单一点可以直接把物理内存分段，平均分为`n` 个`cache line` 份，CPU在Cache中“寻址”时，直接计算地址就能找到该物理内存在Cache的位置了。其优点就是“寻址”很快，`O（1）`的速度，缺点也很明显，一段内存抢夺一个`Cache Line`，而且物理内存通常都是连续使用的，相同一段内存总是会抢来抢去，造成Cache失效，导致效率变低，实现详情见[direct mapped cache][]。两个极端过后自然就会出现一个折中的方案：`N路关联型cache`。

####3.N路关联(N-ways associative cache)

这里把`N`个`Cache Line`作为一个`Set`，这种关联方式其实就是既能通过`O(1)`的速度大致定位到CPU cache上的一个`Set`(即CPU cache的一段区间)，然后又通过`O(n)`的方式在这个`set`中去找需要的内存`TAG`，最后通过`Offset`定位到需要的数据。于是把内存地址看作三段：高`l`位作为`TAG`，中间`m`位作为`set`，低`n`位作为`offset`。先通过`set`的值找到`Cache Line`的区间，然后通过遍历这段`Set`中的`TAG`位比较内存的高`l`位的`TAG`匹配，最后通过`offset`找出该地址的数据，实现详情见[Set Associative Cache][]。

###总结

CPU是一个很复杂的电路结构，这只是其中的冰山一角，不待一幅图，很白话的写了一下CPU Cache 的原理，这些实现Cache的逻辑同样也是可以用到其他的上层业务逻辑里面去的，理念上都是相通的。

[full associative cache]: http://www.cs.umd.edu/class/sum2003/cmsc311/Notes/Memory/fully.html
[direct mapped cache]: http://www.cs.umd.edu/class/sum2003/cmsc311/Notes/Memory/direct.html
[Set Associative Cache]: http://www.cs.umd.edu/class/sum2003/cmsc311/Notes/Memory/set.html