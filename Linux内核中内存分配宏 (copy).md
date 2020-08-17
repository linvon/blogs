---
title: "Linux内核中内存分配宏"
date: "2019-09-27"
categories: ["学习笔记"]
---


在设备驱动程序或者内核模块中动态开辟内存存在 kmalloc 和 vmalloc 两个函数。其区别和使用选择就不多赘述，网上一搜一大把，这里简单分享一个宏，用以处理不同情况的内存分配。


``` c
#define KMALLOC_MAX_SIZE (4096)      ///< kmalloc的最大值

#define ALLOC(x) (((x) > KMALLOC_MAX_SIZE) ?  \    vmalloc(x):                                     \    (in_interrupt() ? kmalloc(x, GFP_ATOMIC) : kmalloc(x, GFP_KERNEL)))

#define FREE(x)          \do                      \{                       \    if (x != NULL)      \    {                   \        ((unsigned long)(x) >= VMALLOC_START) ? vfree(x) : kfree(x);\        x = NULL;       \    }                   \} while (0)


```