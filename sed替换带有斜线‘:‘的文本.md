---
title: "sed替换带有斜线‘/‘的文本"
date: "2019-07-04"
tags: ["linux","sed"]
categories: ["学习笔记", "零散知识"]
---

我们知道 sed 替换命令常写为  
`sed s/pattern/replace/ file`   
但是如果要匹配带有 ‘/’ 的文本就比较麻烦，好在 sed 有个特性：紧跟在替换命令s后面的字符就会被认为是分隔符，也就是说可以写为  
`sed s#pattern#replace# file`  
这样就可以方便的替换 ‘/’ 字符了  