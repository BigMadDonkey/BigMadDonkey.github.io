---
layout: post
title: malloc和calloc的区别
categories: [C]
description: malloc和calloc究竟有什么区别？
keywords: C, memory, system
mathjax: true
---

{:toc}
本文来自于对[一篇博客](https://vorpus.org/blog/why-does-calloc-exist/)的学习。

## 引言

在C库中，用来开辟动态内存空间主要通过`malloc`和`calloc`两个函数完成。

```c
void * buf1 = malloc(size)
void * buf2 = calloc(count, size)
```

从用法和结果上看，他们的区别一个在于参数：`malloc`接收总尺寸，而`calloc`接收内存块的个数和他们的大小，开辟的空间等于二者相乘；另一点在于从`calloc`申请而来的空间总是初始化为0.

但是事实确实如此吗？`calloc`确实只是参数改了一下，并且在`malloc`的基础上`memset`为0吗？

## 区别1: 输入安全性校验

实际上，通过`calloc`进行内存分配有一个好处：其内部会对`count * size`的结果做验证。如果这个结果溢出，则不会分配内存空间。

相应地如果直接在`malloc`调用处传入`count * size`如果计算结果溢出了，对于`malloc`这是不可见的，它不会报告问题，而是开辟一块错误尺寸的空间。

## 区别2: 分配内存时置0操作的代价

本地执行下述代码(gcc, 无额外参数)

```c
#include <stdlib.h>
#include <stdio.h>
#include <time.h>
#include <string.h>

const int LOOPS = 100;

float now()
{
    struct timespec t;
    if (clock_gettime(CLOCK_MONOTONIC, &t) < 0) {
        perror("clock_gettime");
        exit(1);
    }
    return t.tv_sec + (t.tv_nsec / 1e9);
}

int main(int argc, char** argv)
{
    float start = now();
    for (int i = 0; i < LOOPS; ++i) {
        free(calloc(1, 1 << 30));
    }
    float stop = now();
    printf("calloc+free 1 GiB: %0.2f ms\n", (stop - start) / LOOPS * 1000);

    start = now();
    for (int i = 0; i < LOOPS; ++i) {
        void* buf = malloc(1 << 30);
        memset(buf, 0, 1 << 30);
        free(buf);
    }
    stop = now();
    printf("malloc+memset+free 1 GiB: %0.2f ms\n", (stop - start) / LOOPS * 1000);
}
```

观察控制台输出结果：

![两种方式分配内存的时间代价](/images/blog/MallocAndCalloc/MallocAndCalloc-diff2_result.png)

不难发现调用他们的时间成本差别很明显。

造成这种现象的原因，其一在于`memset`的调用是否为必要。当一次申请的内存空间足够大，进程将不得不向OS请求更多的内存资源。当内存资源是由操作系统提供而来时，OS会在将内存分配给进程之前进行内存的置0操作，防止读取到内存上之前保留的数据。显式使用`malloc` + `memset`会不分场合地多置0，而`calloc`由于与内存分配单元结合更好，对内存的来源是有感知的，因此可以避免的多余的内存置0操作。

但是这仍不足以解释为什么`calloc`需要的时间如此之短。如果`calloc`只是全置一遍0而`malloc`会多执行一次，他们的时间应该在两倍左右，但事实上calloc的消耗要小的多。

这是因为OS在分配内存时借助乐虚拟内存的机制：申请大量的置0块，实际上并不会真的将1GB的内存区域全部置0，而是将一个page的内存区域全部置0并设为**写时复制**，然后该进程的页表中新申请项的位置都指向这个page即可。这样置0的时间就不会集中在申请时，而是每次访问不同page时。每当访问了一个没有访问过的page，则将这块全0的page复制一份到新的物理内存page上。换言之，将内存置0的操作延迟到具体使用时执行，按需置0！

然而，对于`malloc`的情况，显示调用memset相当于再上述操作的基础上，再对每一块内存写一遍。因此`calloc` 与 `malloc`在分配内存上的时间差，在于前者只写1个page，后者则是实实在在的都写了一遍！

## 结论

在有条件的情况下，能用`calloc`就尽量使用，它能让内存分配的操作更安全，且时间成本的分配更合理（延迟加载）。
