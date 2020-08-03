---
title: "Linux三剑客使用笔记"
date: "2019-05-05"
tags: ["linux","grep","sed","awk"]
toc: "true"
categories: ["学习笔记", "linux"]
---
## **Grep**

> grep主要用于查找文本或文件中是否含有要查找的 “关键字”

```bash
-H	显示文件名前缀
-h	不显示文件名前缀
-n	显示行号前缀
-l	只显示匹配到的文件的文件名
-L	只显示未匹配到的文件的文件名
-c	只显示匹配到的行数
-o	只显示行中匹配的部分
-q	静默模式. 匹配到返回1，否则返回0
-v	显示未匹配到的行
-r	Recurse
-i	忽略大小写
-w	全词匹配
-x	全句匹配
-F	不使用正则表达式，仅文本匹配
-E	使用扩展表达式
-m N	每个文件最多匹配N次
-A N	输出文本的最后N行
-B N	输出文本的首部N行
-C N	输出文本的前后各N行
-e PTRN	指定正则表达式
-f FILE	从文件中读取正则
```


## Sed

> sed主要是对文本或文件进行修改操作，比如追加、替换、删除等

```bash
sed [-inrE] CMD [FILE]	-i 表示在文件内执行，不输出；-r -E 表示使用扩展正则；-n 只打印匹配到的行
注：不加参数会整体输出执行后结果，但不会对源文件做改变，要修改文件需要加-i参数

-------------------------------------------------------------
sed的命令包含多种参数，参数一般都有如下两种语法：
sed -n 'n1,n2p' file 			数字代表行数,n1 n2可表示从n1到n2行，后接命令参数
sed -n '/pattern/p' file		使用斜线，采用正则表达式匹配行，后接命令参数

p ：显示，将某个选择的数据打印显示。通常 p 会与参数 sed -n 一起执行
sed -n '2p' file 			打印第2行
sed -n '/pattern/p' file 		打印匹配到pattern的行
a ：添加， a 的后面可以接字符串，该字符串会在当前指定行的下一行出现
sed '2a\content' file			在第2行后追加一行content内容
sed '/pattern/a\content' file		在匹配到内容后追加一行content内容
d ：删除，删除指定行后的内容
sed '3d' file				删除第3行内容
sed '1,3d' file				删除1-3行内容
sed '/pattern/d' file			删除匹配到行后内容
c ：更改， c 的后面可以接字符串，该字符串可以取代 n1,n2 之间的行
sed '3c content' file			更改第3行内容为content
sed '1,3c content' file			更改1-3行内容为content
sed '/pattern/c\content' file		更改匹配到行内容为content
i ：插入， i 的后面可以接字符串，该字符串会在当前指定行的上一行出现
sed '2i\content' file 			在第2行前插入一行content内容
sed '/pattern/i\content' file		在匹配到内容前插入一行content内容
s ：替换，使用s做前缀，如 
sed 's/pattern/replace/g' file  	进行正则匹配,g表示多次替换
w ：写入，
sed '/pattern/w /tmpfile' file 		将匹配行内容重定向写入到指定文件
-------------------------------------------------------------——
sed 's/\([0-9]\)[a-zA-Z]\([0-9]\)/\1aaa\2/g' 使用\1 \2可以引用匹配串括号内表达式内容
```

## **Awk**

> awk主要用于对文本或文件进行更加复杂的操作，支持编程，主要进行列处理

```bash
awk -v a="$idx" '{print $a}' file	-v 设定变量，例为输出第idx列
awk -F ' ' '{print $2}' file		-F 设定分隔符，例为空格
awk '{print NR}' file  			输出行数
awk '{print FNR}' file1 file2		输出每个文件重新计算的行数
awk '{print NF}' file  			输出列数
awk '{print $NF}' file  		输出最后一列结果
awk '{$4++}END{print $4}' file		多个程序段
awk '!($1 in a){a[$1]=1;print}' 	对于结果的第一列去重
awk '{sum += $1};END {print sum}' 	对第一列求和
```

## **组合技巧**

```bash
ls -lt /tmp | grep key | head -n 1 |awk '{print $9}'  获取最新的含有key关键字的文件名
```

