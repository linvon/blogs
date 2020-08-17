---
title: "解决 Linux 驱动导出接口的引用依赖限制"
date: "2019-11-11"
categories: ["学习笔记"]
---



假设有情景：驱动 A 要调用驱动 B 的函数，那么需要驱动 B 来导出接口，也就是符号。这样一来就会产生驱动之间的依赖性，如果驱动 B 没有挂载，那么驱动 A 找不到符号也无法挂载。

为了解决这个问题，我们可以采用类似于动态链接库的实现方式，将接口的定义放在公共基础 ko 中，B 来实现接口，A 在调用时先去判断接口是否已实现，这样可以避免 A 和 B 之间的直接依赖。

```  c

// 公共文件，声明并导出接口
int (*my_fn)(int args) __read_mostly = NULL;
EXPORT_SYMBOL(my_fn);

```

``` c

// B 文件 实现接口

extern int (*my_fn)(int args);  // 先声明

int real_fn(int args)
{
    do something;
}

static int my_init(void){
    rcu_assign_pointer(my_fn, real_fn);   // 初始化时将接口实现
}

static int my_exit(void){
    rcu_assign_pointer(my_fn, NULL);;    // 退出时将接口置 NULL
}

module_init(my_init);
module_exit(my_exit);

```

``` c

// A 文件 调用接口

extern int (*my_fn)(int args);  // 先声明

typeof(my_fn) do_fn = rcu_dereference(my_fn); // 定义接口的实现

if (do_fn)      // 如果有实现就调用
{
    do_fn(args);
}

```