---
layout: post
title: malloc和calloc的区别
categories: Blog
description: malloc和calloc究竟有什么区别？
keywords: C, memory, system
---

{:toc}
本文来自于对[一篇博客](https://vorpus.org/blog/why-does-calloc-exist/)的学习。

## 引言
在C库中，用来开辟动态内存空间主要通过`malloc`和`calloc`两个函数完成。
```C
void * buf1 = malloc(size)
void * buf2 = calloc(count, size)
```
从用法和结果上看，他们的区别一个在于参数：`malloc`接收总尺寸，而`calloc`接收内存块的个数和他们的大小，开辟的空间等于二者相乘；另一点在于从`calloc`申请而来的空间总是初始化为0.

但是事实确实如此吗？`calloc`确实只是参数改了一下，并且在`malloc`的基础上`memset`为0吗？

## 区别1: 输入安全性校验

实际上，通过`calloc`进行内存分配有一个好处：其内部会对`count * size`的结果做验证。如果这个结果溢出，则不会分配内存空间。

相应地如果直接在`malloc`调用处传入`count * size`如果计算结果溢出了，对于`malloc`这是不可见的，它不会报告问题，而是开辟一块错误尺寸的空间。

## 区别2: 分配内存的具体方式

本地执行下述代码(gcc, 无额外参数)
```C
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