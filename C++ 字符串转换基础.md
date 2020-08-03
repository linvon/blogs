---
title: "C++ \u5B57\u7B26\u4E32\u8F6C\u6362\u57FA\u7840"
date: "2019-09-22"
toc: false
tags:
- c++
categories:
- c++
- "\u5B66\u4E60\u7B14\u8BB0"
...
--- 在C++中，经常使用到的字符串有三种：char\* 、CString、string  

首先解释一下三种类型：  
**char***是最基础的， 也是在C语言中使用的字符串类型，是最早的类型，其实际上是指向一个char类型存储的指针  
**CString**实际上是char数组，在其之上进行了封装，出现早于string，定义于\<afx.h\>  
**string**是C++标准库实现的类，操作比较方便，相信不用过多介绍，定义于\<string.h\>  


再介绍几个基本知识：  

**\_T宏：**  
\_T()宏在8位字符环境下是如下定义的：  
#define \_T(x) x // 非Unicode版本（non-Unicode version）  
而在Unicode环境下是如下定义的：  
#define \_T(x) L##x // Unicode版本（Unicode version）  
该宏主要用于解决Unicode的兼容性问题，如果需要处理Unicode编码问题，可以对字符串采用此宏处理。  


**LPTSTR、LPCTSTR：**  
我们经常能看到这种类型强制转换，实际上是一种字符串指针类型。其作用是用来表示你的字符是否使用UNICODE, 如果你的程序定义了UNICODE或者其他相关的宏，那么这个字符或者字符串将被作为UNICODE字符串，否则就是标准的ANSI字符串。至于UNICODE和ANSI是两种标准字符编码，对于字符编码可另加搜索，我们常用的UTF-8是UNICODE，而GBK是ANSI的。因此这种强制转换经常用在需要处理中文或者统一编码的代码中。  
（L-Long表示long指针，历史遗留，无实际意义；P-Pointer表示是一个指针；C-Const表示是一个常量；T-表示_T宏，编码相关；STR-表示是一个字符串）  


## 转换

``` c++
/* char* */
    -> CString
        CString.format(_T("%s"), char*);  // 要保证char*是以‘\0’结尾， _T()宏用以处理Unicode
        CString s = char*;     // 将char*直接赋值过去也是可以的
    -> string
        string str(char*);
        
/* CString */
    -> char*
        char* =  (LPCTSTR)CString;    // 使用指针强制转换
        strcpy(char*, CString, sizeof(char));  // 使用字符串拷贝，要注意char*的缓冲区长度
        char* = CString.GetBuffer(); do use char* ;CString.ReleaseBuffer();  // GetBuffer默认参数为0，表示以CString原长度分配定长字符串，否则需要传入要分配的字符串缓冲区长度，其实质上是新分配一块内存存储字符串，将指针给char*，因此在使用后需要调用ReleaseBuffer释放内存。
    -> string
        string  s(CString.GetBuffer());  CString.ReleaseBuffer(); // 实际上，CString到string就是先转换为char*，再转成string 
        
/* string */
    -> char*
        char* = string.c_str();  // 返回带'\0'结尾的字符串。该内存不需要我们管理，只需要处理指针指向即可
        char* = string.data();  // 返回不带'\0'的字符串数据
    -> CString
        CString.format(_T("%s"), string.c_str()); // 实际上也是利用char*做中转，完成转换
```

需要注意的是，向char\*转换时，实际上我们已经开始操作指针了，这时候就需要我们自己进行内存管理，建议使用const char\*，这样更为安全  