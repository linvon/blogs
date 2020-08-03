---
title: "学习JavaScript回调机制"
date: "2017-08-22"
tags: ["JS"]
categories: ["学习笔记", "JS"]
---
因为对JS的了解不多，还在逐渐学习，最近接触了JS的回调机制，还是总结一下。

> A callback is a function that is passed as an argument to another function and is executed after its parent function has completed.

根据定义可以看出，我们可以简单理解回调函数是它的父级函数执行完后才会执行的函数，以参数的形式传递给它的父级函数。

``` javascript
//定义主函数，回调函数作为参数
function A(callback) {
    callback();  
    console.log('我是主函数');      
}

//定义回调函数
function B(){
    setTimeout("console.log('我是回调函数')", 3000);//模仿耗时操作  
}

//调用主函数，将函数B传进去
A(B);

//输出结果
我是主函数
我是回调函数
```


　定义主函数的时候，我们让代码先去执行callback()回调函数，但输出结果却是后输出回调函数的内容。这就说明了主函数不用等待回调函数执行完，可以接着执行自己的代码。所以一般回调函数都用在耗时操作上面。比如ajax请求，比如处理文件等。

在Javascript编程中回调函数经常以几种方式被使用，尤其是在现代web应用开发以及库和框架中：

- 异步调用（例如读取文件，进行HTTP请求，等等）
- 时间监听器/处理器
- setTimeout和setInterval方法
- 一般情况：精简代码

``` javascript
$.ajax({
    url:"test.json",
    type: "GET",
    data: {username:$("#username").val()},
    dataType: "json",
    beforSend:function(){ 
         // 禁用按钮防止重复提交
        $("#submit").attr({ disabled: "disabled" });
    }, 
    complete:function(msg){ 
        //请求完成后调用的回调函数（请求成功或失败时均调用）
    } , 
    error:function(msg){ 
        //请求失败时被调用的函数 
    } , 
    Sucess:function(msg){ 
        //请求成功后调用的回调函数 
    } 
});
```
这是一个附带回调函数的Ajax请求，可以根据不同的响应进行不同的函数操作，作用很大。

其他参考资料：[深入理解回调函数](http://www.cnblogs.com/gaosheng-221/p/6045483.html)