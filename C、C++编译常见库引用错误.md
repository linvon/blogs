---
title: "C、C++编译常见库引用错误"
date: "2019-08-15"
categories: ["学习笔记"]
---


`symbol lookup error xxxxx , undefined symbol`
这种情况一般是程序依赖的动态库不对，动态库中并没有程序所使用的符号，可以借助 nm 和 ldd 工具分析动态库

`libxxx.so: undefined reference to 'xxxxx'`
这种情况可能有多种原因：
1. 编译的时候动态、静态库没有引用好，导致找不到引用的函数体
2. 头文件中没有定义、导出函数，或者头文件没有正常加载导致找不到函数定义
3. 动态库编译时没有引用头文件导出函数主体，虽然动态库编译成功了，但是引用方找不到函数
4. C、C++ 混编 , 头文件引用、导出有误，详细如下

``` C
// 只要发生 C 和 C++ 混编互相调用，就应该在头文件中加入如下定义
#ifdef __cplusplus    /*C++ 编译器包含的宏，例如用 g++ 编译时，该宏就存在，则下面的语句 extern "C" 才会被执行 */
extern "C" {          /*C++ 编译器才能支持，C 编译器不支持 */
#endif
 
extern int fun();
 
#ifdef __cplusplus
}
#endif

// 如果 C 库的头文件并没有做以上操作，可以在 C++ 程序引用相关头文件时加入该定义
extern "C" {
include "xxxx.h"
}
```
