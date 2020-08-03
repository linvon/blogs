---
title: "学习URLencode"
date: "2017-08-22"
tags: ["URLencode"]
categories: ["学习笔记", "Web开发"]
---
URL编码是在日常编码中经常使用而又缺少注意的一个环节。

# 多次编解码会带来问题吗？
我们在Web开发中经常会遇到让用户填入URL的情况，而URL填入了页面，再到存入数据库，再到取出访问，很可能经过你不确定多少次的编解码，这会不会带来问题？

答案是肯定会的

首先在编码方面，URLencode将中文和特殊字符编码成 %xy 的形式，如果执行二次编码，那么“%”会被再次编码，变为 %xyxy ，这样一来就需要两次解码才能恢复到我们需要的内容。

其次在解码方面，如果url是被执行了一次以内的编码，那么多次解码是没有影响的，但如果url进行了多次的编码，或者url的参数中本身含有被编码的参数，那么在执行多次编码的情况下，很可能因为编解码次数不对应而导致无法获得自己最终需要的内容。

综上所述，在Web开发过程中，严格控制url的编解码次数，严格控制数据存储流程是十分必要的。

# 编解码的方式与标准？

- 浏览器、服务器自动编解码  
  我们可以注意到，当你访问一个含有中文的url的时候，将他从浏览器中复制出来，它就已经是被编码过的了。同样，部分服务器在接收并取出url后也会对其进行解码，呈现在你眼前的是一个解码后的url。
  
- JavaScript编解码
  我们可以注意到，在JS中有三种不同的编解码函数。
  escape,encodeURI,encodeURIComponent
  
  >   1、   传递参数时需要使用encodeURIComponent，这样组合的url才不会被#等特殊字符截断。                            
  >
  >   例如：
  >
  >   http://passport.baidu.com/?logout&aid=7&u=+ encodeURIComponent("http://cang.baidu.com/bruce42") 
  >
  >
  >   2、   进行url跳转时可以整体使用encodeURI 
  >
  >   例如：Location.href=encodeURI("http://cang.baidu.com/do/s?word=百度&ct=21"); 
  >
  >   3、   js使用数据时可以使用escape 
  >
  >   例如：搜藏中history纪录。 
  >
  >   
  >
  >   最多使用的应为encodeURIComponent，它是将中文、韩文等特殊字符转换成utf-8格式的url编码，所以如果给后台传递参数需要使用encodeURIComponent时需要后台解码对utf-8支持（form中的编码方式和当前页面编码方式相同） 
  >
  >   escape不编码字符有69个：*，+，-，.，/，@，_，0-9，a-z，A-Z 
  >
  >   encodeURI不编码字符有82个：!，#，$，&，'，(，)，*，+，,，-，.，/，:，;，=，?，@，_，~，0-9，a-z，A-Z 
  >
  >   encodeURIComponent不编码字符有71个：!， '，(，)，*，-，.，_，~，0-9，a-z，A-Z 

- Php、C、C++编解码  
其他后端的编解码效果大致与encoudeURIComponent相同

因为JS编解码方式有三种，所以在处理传输数据的编解码的时候要注意使用正确的函数。