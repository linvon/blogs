---
title: "Shell语法笔记"
date: "2018-12-04"
tags: ["shell"]
toc: "true"
categories: ["学习笔记", "shell"]
---
# **Shell中三种引号的用法**

- 单引号

```bash
#所见即所得
var=123 
var2='${var}123'
echo var2 var2  #结果为${var}123
```

- 双引号

```bash
#输出引号中的内容，若存在命令、变量等，会先执行命令解析出结果再输出

var=123 
var2="${var}123"
echo var2 var2  #结果为123123
```

- 反引号（键盘tab键上面一个键）

```bash
#命令替换:root用户登录系统
var=`whoami`
echo $var var  #结果为执行whoami命令的结果 显示root

备注：反引号和$()作用相同
```

> 参考：https://www.cnblogs.com/lqcjlu/p/6226405.html



# **Shell中函数返回值的问题**
在shell函数return的值是赋值给$?的，而变量直接接收的值是函数中的echo的输出值。同时要注意$?接受的是上一条命令执行的返回值，即紧邻的上一条任何命令

```bash
#!/bin/bash
function fun1()
{
    local out=`123`
    return $out
}

function fun2()
{
    local out=`223`
    echo $out
    return $out
}

ret=`fun1`
echo "ret : "$ret    #ret :

ret=`fun2`
echo "ret : "$ret    #ret :223

fun2
ret=$?				
echo "ret : "$ret    #ret :223

ret=$?				
echo "ret : "$ret    #ret :0
```



# **Shell中条件判断参数含义**

- 文件判断

```bash
[ -a FILE ]  如果 FILE 存在则为真。  
[ -b FILE ]  如果 FILE 存在且是一个块特殊文件则为真。  
[ -c FILE ]  如果 FILE 存在且是一个字特殊文件则为真。  
[ -d FILE ]  如果 FILE 存在且是一个目录则为真。  
[ -e FILE ]  如果 FILE 存在则为真。  
[ -f FILE ]  如果 FILE 存在且是一个普通文件则为真。  
[ -g FILE ]  如果 FILE 存在且已经设置了SGID属性则为真。 
[ -h FILE ]  如果 FILE 存在且是一个符号连接则为真。  
[ -k FILE ]  如果 FILE 存在且已经设置了粘制位则为真。  
[ -p FILE ]  如果 FILE 存在且是一个名字管道(F如果O)则为真。  
[ -r FILE ]  如果 FILE 存在且是可读的则为真。  
[ -s FILE ]  如果 FILE 存在且大小不为0则为真。  
[ -t FD ]    如果文件描述符 FD 打开且指向一个终端则为真。  
[ -u FILE ]  如果 FILE 存在且设置了SUID (set user ID)则为真。  
[ -w FILE ]  如果 FILE 存在且是可写的则为真。  
[ -x FILE ]  如果 FILE 存在且是可执行的则为真。  
[ -O FILE ]  如果 FILE 存在且属有效用户ID则为真。  
[ -G FILE ]  如果 FILE 存在且属有效用户组则为真。  
[ -L FILE ]  如果 FILE 存在且是一个符号连接则为真。  
[ -N FILE ]  如果 FILE 存在且上一次读取后被修改过则为真。  
[ -S FILE ]  如果 FILE 存在且是一个套接字则为真。  
[ FILE1 -nt FILE2 ]  如果 FILE1 比 FILE2 新, 或者 FILE1 存在且 FILE2 不存在则为真。 
[ FILE1 -ot FILE2 ]  如果 FILE1 比 FILE2 旧, 或者 FILE2 存在且 FILE1 不存在则为真。 
[ FILE1 -ef FILE2 ]  如果 FILE1 和 FILE2 指向相同的设备和节点号则为真。  
```

- 字符串判断

```bash
[ -z STRING ]  如果“STRING” 的长度为零则为真。  
[ -n STRING ] or [ STRING ]  如果“STRING” 的长度为非零则为真。  
[ STRING1 == STRING2 ]  如果两个字符串相同则为真。  
[ STRING1 != STRING2 ]  如果两个字符串不相等则为真。 
[ STRING1 < STRING2 ]  如果 “STRING1” 小于 “STRING2” 的字典序则为真。  
[ STRING1 > STRING2 ]  如果 “STRING1” 大于 “STRING2” 的字典序则为真。  
```

- 数字判断
  
```bash
int1 -eq int2　　　　两数相等为真 
int1 -ne int2　　　　两数不等为真 
int1 -gt int2　　　　int1大于int2为真 
int1 -ge int2　　　　int1大于等于int2为真 
int1 -lt int2　　　　int1小于int2为真 
int1 -le int2　　　　int1小于等于int2为真
```

- 逻辑判断
  
```bash
-a     与 
-o     或 
!      非
```

> 参考：https://www.cnblogs.com/liupuLearning/p/6206415.html


# **Shell中变量的替换**
假设我们定义了一个变量为：
`file=/dir1/dir2/dir3/my.file.txt`

```sh
可以用${ }分别替换得到不同的值：
${file#*/}	删掉第一个   '/' 及其左边的字符串：dir1/dir2/dir3/my.file.txt
${file##*/}	删掉最后一个 '/' 及其左边的字符串：my.file.txt
${file#*.}	删掉第一个   '.' 及其左边的字符串：file.txt
${file##*.}	删掉最后一个 '.' 及其左边的字符串：txt
${file%/*}	删掉最后一个 '/' 及其右边的字符串：/dir1/dir2/dir3
${file%%/*}	删掉第一个   '/' 及其右边的字符串：(空值)
${file%.*}	删掉最后一个 '.' 及其右边的字符串：/dir1/dir2/dir3/my.file
${file%%.*}	删掉第一个   '.' 及其右边的字符串：/dir1/dir2/dir3/my

${file:0:5} 提取最左边的 5 个字节          ：/dir1
${file:5:5} 提取第 5 个字节右边的连续5个字节：/dir2

也可以对变量值里的字符串作替换：
${file/dir/path}  将第一个dir 替换为path：/path1/dir2/dir3/my.file.txt
${file//dir/path} 将全部dir   替换为path：/path1/path2/path3/my.file.txt
${#var} 可计算出变量值的长度
```

记忆的方法为：

> '#'是去掉左边（键盘上#在 $ 的左边）
> '%'是去掉右边（键盘上% 在$ 的右边）
> 单一符号是最小匹配；两个符号是最大匹配

|   变量配置方式   |    str 没有配置    |   str为空字符串    | str已配置非为空字符串 |
| :--------------: | :----------------: | :----------------: | :-------------------: |
| var=${str-expr}  |      var=expr      |        var=        |       var=$str        |
| var=${str:-expr} |      var=expr      |      var=expr      |       var=$str        |
| var=${str+expr}  |        var=        |      var=expr      |       var=expr        |
| var=${str:+expr} |        var=        |        var=        |       var=expr        |
| var=${str=expr}  | str=expr var=expr  |    str不变var=     |   str不变 var=$str    |
| var=${str:=expr} | str=expr var=expr  | str=expr var=expr  |   str不变 var=$str    |
| var=${str?expr}  | expr 输出至 stderr |        var=        |       var=$str        |
| var=${str:?expr} | expr 输出至 stderr | expr 输出至 stderr |       var=$str        |

> 参考：https://blog.csdn.net/shmilyringpull/article/details/7631106	