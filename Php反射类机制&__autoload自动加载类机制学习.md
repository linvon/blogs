---
title: "Php反射类机制&__autoload自动加载类机制学习"
date: "2017-08-22"
toc: "true"
tags: ["PHP"]
categories: ["学习笔记", "PHP"]
---
# 反射类机制
**使用Php反射API可以简单的实现站点的插件。**

比如一个系统内部有着各种各样的信息管理，像用户名、用户组、用户拥有的各类权限等等，而页面上的访问主要是给用户来使用的，站点管理员可能需要一个接口从服务器中快速获取数据，但是服务器的代码和数据又是封闭的，这种时刻反射API就可以很好的服用服务器上部署好的代码。   


我们在服务器上可能已经写好了查询、分析信息的代码，但这些代码都是给我们的用户页面服务的，这个时候通过反射API将数据信息的查询类做一个反射供接口使用，就可以写少量的代码而轻松地得到与前端页面一致的数据。


但是在实现了便利性的同时，反射机制也带来了一些缺点，比如破坏了类的封装性，这些也是在我们编码过程中要考虑到的。

详细内容可以参考：[Php手册](http://www.php.net/manual/zh/book.reflection.php)


通过ReflectionClass，我们可以得到类的以下信息：

- 常量 Contants
- 属性 Property Names
- 方法 Method Names
- 静态属性 Static Properties
- 命名空间 Namespace
- 类是否为final或者abstract


``` php
class Person {


    private $id = 0;

    private $name;

    function __construct($name)
    {
        $this->name = $name;
    }

    public function getName()
    {
        return $this->name;
    }
    public function setName($v)
    {
        $this->name = $v;
    }
    public function getId()
    {
        return $this->id;
    }
    public function setId($v)
    {
        $this->id = $v;
    }
}
```

``` php
//加载反射类
$class = new ReflectionClass("Person");
//实例化一个对象
$instance = $class->newInstance("K");

//调用类内方法
echo $instance->getName()."<br>";
//打印类内方法
print_r($class->getMethods()) ;
echo "<br>";
//调用类内方法的另一种形式
$ec = $class->getMethod("getId");
echo $ec->invoke($instance);
```

# __autoload自动加载类机制

前面介绍了反射类机制，接下来学习__autoload自动加载类机制

根据官方文档  
**__autoload — 尝试加载未定义的类**

通过__autoload的配合，可以让站点插件的实现更加便捷。  
当在使用反射类的时候，反射API的页面自然是没有加载我们所需要使用的类的（在项目的其他地方也可能出现这个问题）。
那么我们就需要将这个类引入当前文件中，最容易想到的便是  

`require_once ("Person.class.php"); `

但是如果项目规模较大，这样肯定会带来编码上的问题，也不便于后续的维护。而autoload便提供了一种加载方式。

``` php
function __autoload($class_name) { 
$path = str_replace('_', '/', $class_name); 
require_once $path . '.php'; 
} 
// 这里会自动加载Http/File/Interface.php 文件 
$a = new Http_File_Interface(); 
```

这种方法的好处就是简单易使用。当然也有缺点，缺点就是将类名和文件路径强制做了约定，当修改文件结构的时候，就势必要修改类名。 

```php
$map = array( 
'Http_File_Interface' => 'C:/PHP/HTTP/FILE/Interface.php' 
); 
function __autoload($class_name) { 
if (isset($map[$class_name])) { 
require_once $map[$class_name]; 
} 
} 
// 这里会自动加载C:/PHP/HTTP/FILE/Interface.php 文件 
$a = new Http_File_Interface(); 
```

这种方法的好处就是类名和文件路径只是用一个映射来维护，所以当文件结构改变的时候，不需要修改类名，只需要将映射中对应的项修改就好了。 


这样一来，我们可以单独用一个文件来维护类名和路径的映射，反射类API在使用的时候只需要调用这个autoload函数来加载需要的类即可，省去了大量的编码内容。


参考资料：

[Php手册](http://php.net/manual/zh/function.autoload.php)  
[autoload机制](http://www.jb51.net/article/31399.htm)  
[从底层分析autoload](http://www.jb51.net/article/31279.htm)