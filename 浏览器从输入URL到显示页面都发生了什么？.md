---
title: "浏览器从输入URL到显示页面都发生了什么？"
date: "2017-03-30"
tags: ["Web", "浏览器"]
categories: ["学习笔记", "Web开发"]
---
前几天接一个面试电话，上来就问了这么一个问题，虽说懵懵懂懂知道一点，但远不够让人满意，还是要学习一下。

总体上分为三大步骤：  

- 浏览器解析URL并与服务器建立连接 
- 服务器响应请求并返回HTML文件
- 浏览器渲染HTML页面  

### 建立连接  
首先浏览器需要将输入的URL解析成服务器的IP地址，在这个过程会先后经历  

>  1. 查找浏览器缓存  
>  2. 查找系统本地缓存  
>  3. 查找路由器缓存  
>  4. 查找ISP（网络提供商） DNS缓存  
>  5. 递归查找IP地址

*<font size=2>关于DNS还有一些细节的优化内容，详情可以在下面的参考链接中学习</font>*  

找到IP地址后，浏览器会向服务器发送一个HTTP请求，HTTP是建立在TCP/IP基础之上的，所以会产生一个基于socket的链接。


### 服务器响应请求  
服务器上的相对应web软件会解析收到的请求。主要是通过解析出路径来找到相对应的文件以及处理一些包含在cookie中的请求等等。最终服务器会生成一个HTML响应返回给浏览器。404，500等错误也会发生在这一过程中。~~（MVC模式的形成）~~

### 浏览器渲染页面
浏览器接收到服务器传来的HTML响应并开始渲染HTML页面，其过程主要如下：  

> <font size='2'>首先，开源浏览器一般以8k每块下载html页面。  
> 然后解析页面生成DOM树，遇到css标签或JS脚本标签就新起线程去下载他们，并继续构建DOM。下载完后解析CSS为CSS规则树，浏览器结合CSS规则树和DOM树生成Render Tree。  
> 注意：构建CSS Object Model（CSSOM)会阻塞JavaScript的执行。JavaScript的执行也会阻塞DOM的构建。JavaScript下载后可以通过DOM API修改DOM，通过CSSOM API修改样式作用域Render Tree。  
> 每次修改会造成Render Tree的重新布局和重绘。只要修改DOM或修改了元素的形状或大小，就会触发Reflow，单纯修改元素的颜色只需Repaint一下（调用操作系统Native GUI的API绘制）。  
> 作者：陈金
> 链接：https://www.zhihu.com/question/30218438/answer/84704484  
> 来源：知乎
> </font>

其他参考链接：  
[百度面试题：从输入url到显示网页，后台发生了什么？](https://www.cnblogs.com/rollenholt/archive/2012/03/23/2414345.html)  
[浏览器输入url到整个页面显示出来经历的过程](https://www.cnblogs.com/lichenghan/p/4019370.html)