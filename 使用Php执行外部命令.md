---
title: "使用Php执行外部命令"
date: "2017-08-22"
toc: "true"
tags: ["PHP"]
categories: ["学习笔记", "PHP"]
---
在编写服务器代码的时候经常会遇到这样的情况：配置文件被修改，某些服务需要重启

这个时候我们必须执行shell命令来继续下一步操作，而这些操作我们都可以在Php里面完成。

# 配置  

>查看php.ini中配置是否打开安全模式，主要是以下三个地方  
>safe_mode =  (这个如果为off下面两个就不用管了)  
>disable_functions =   
>safe_mode_exec_dir=  

# 使用  

>由于PHP基本是用于WEB程序开发的，所以安全性成了人们考虑的一个重要方面。于是PHP的设计者们给PHP加了一个门：安全模式。如果运行在安全模式下，那么PHP脚本中将受到如下四个方面的限制：  
>① 执行外部命令  
>② 在打开文件时有些限制  
>③ 连接MySQL数据库  
>④ 基于HTTP的认证  
>在安全模式下，只有在特定目录中的外部程序才可以被执行，对其它程序的调用将被拒绝。这个目录可以在php.ini文件中用 safe_mode_exec_dir指令，或在编译PHP是加上--with-exec-dir选项来指定，默认是/usr/local/php /bin。  
>如果你调用一个应该可以输出结果的外部命令（意思是PHP脚本没有错误），得到的却是一片空白，那么很可能你的站点管理员已经把PHP运行在安全模式下了。  

# 执行  
在PHP中调用外部命令，可以用如下三种方法来实现：

## 用PHP提供的专门函数  
PHP提供共了4个专门的执行外部命令的函数：exec(), system(), passthru(), shell_exec()   

- system() 
  原型：string system (string command [, int return_var])   
  system()函数很其它语言中的差不多，它执行给定的命令，输出和返回结果。第二个参数是可选的，用来得到命令执行后的状态码。  
  例子：
  
``` php
system("pwd");
```

- exec()
  原型：string exec (string command [, string array [, int return_var]])  
  exec() 函数与system()类似，也执行给定的命令，但不输出结果，而是返回结果的最后一行。虽然它只返回命令结果的最后一行，但用第二个参数array可以得到完整的结果，方法是把结果逐行追加到array的结尾处。所以如果array不是空的，在调用之前最好用unset()最它清掉。只有指定了第二个参数时，才可以用第三个参数，用来取得命令执行的状态码。   
  例子： 
  
```php
exec("/bin/ls -l");
exec("/bin/ls -l", $res);
//$res是一个数据，每个元素代表结果的一行
exec("/bin/ls -l", $res, $rc);
//$rc的值是命令/bin/ls -l的状态码。成功的情况下通常是0
```

- passthru()  
  原型：void passthru (string command [, int return_var])  
  passthru() 只调用命令，不返回任何结果，但把命令的运行结果原样地直接输出到标准输出设备上。所以passthru()函数经常用来调用象pbmplus（Unix 下的一个处理图片的工具，输出二进制的原始图片的流）这样的程序。同样它也可以得到命令执行的状态码。  
  例子：
  
```php
  header("Content-type: image/gif");
  passthru("./ppmtogif hunte.ppm");
```

- shell_exec()  
  原型: string shell_exec ( string $cmd )  
  说明: 直接执行命令$cmd
  例子：
  
```php
  <?php
  $output = shell_exec('ls -lart');
  echo "<pre>$output</pre>";
  ?>
```


## 反撇号  
原型: 反撇号`（和~在同一个键）执行系统外部命令  
说明: 在使用这种方法执行系统外部命令时，要确保shell_exec函数可用，否则是无法使用这种反撇号执行系统外部命令的。

```php
<?php
echo `dir`;
?>
```


## 用popen()函数打开进程  
原型: resource popen ( string $command , string $mode )  
说明: 能够和命令进行交互。之前介绍的方法只能简单地执行命令，却不能与命令交互。有时须向命令输入一些东西，如在增加系统用户时，要调用su来把当前用户换到root用户，而su命令必须要在命令行上输入root的密码。这种情况下，用之前提到的方法显然是不行的。  
popen( )函数打开一个进程管道来执行给定的命令，返回一个文件句柄，可以对它读和写。返回值和fopen()函数一样，返回一个文件指针。除非使用的是单一的模式打开(读or写)，否则必须使用pclose()函数关闭。该指针可以被fgets(),fgetss(),fwrite()调用。出错时，返回FALSE。

```php
<?php
error_reporting(E_ALL);
/* Add redirection so we can get stderr. */
$handle = popen('/path/to/executable 2>&1', 'r');
echo "'$handle'; " . gettype($handle) . "\n";
$read = fread($handle, 2096);
echo $read;
pclose($handle);
?>
```



参考资料：
[PHP中调用外部命令的方法](http://www.cnblogs.com/acetaohai123/p/6571414.html)