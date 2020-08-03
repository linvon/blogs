---
title: "Go defer\u7684\u4F7F\u7528"
date: "2019-09-17"
toc: false
tags: ~
categories:
- go
- "\u5B66\u4E60\u7B14\u8BB0"
...
--- 本文参考自[这里](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.4.html"%3Ehttps://tiancaiamao.gitbooks.io/go-internals/content/zh/03.4.html)

defer是Go语言中一个非常便利的特性，它可以在函数结束退出时自动执行，帮助我们去处理资源释放等问题。  
defer语句类似于一个栈，按照代码顺序入栈，在退出时出栈执行，即越靠后的defer会越先执行  

在使用defer的时候，需要注意一个问题：函数的return并不是一个原子指令，它包括一个赋值操作和一个return操作，而defer运行在赋值和return之间。  
我们来详细解释一下：  
熟悉过Go函数的会知道，Go函数可以指定出参为函数内的某个变量，在return时可以给该出参赋值，以下面函数为例  

``` go
func f() (result int) { // 指定了函数的出参为result
    defer func() {      // defer内容为result++
        result++
    }()
    return 0            // 此处指定返回为0，即将0赋值给result返回
}
```
上面这个函数的实际返回值为1，为什么呢？  
因为上面说到的问题，实际上return是先给result赋值，在执行return操作，而defer在这个两个操作的中间，即：  

```
0 -> result  
result ++
return 
```

因此我们需要在使用defer时注意这个问题：  
如果**函数指定了出参并且defer函数操作了出参**，那么出参的值可能会与你预想的不一样，需要加以检查  
因此在实际编写代码时，尽量避免这种耦合操作，比如不需要指定出参，直接return参数即可  
