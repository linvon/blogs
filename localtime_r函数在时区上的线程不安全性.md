---
title: "localtime_r 函数在时区上的线程不安全性"
date: "2019-10-27"
categories: ["学习笔记"]
---


``` C

struct tm *localtime(const time_t *timep);
struct tm *localtime_r(const time_t *timep, struct tm *result);

```

在 Linux C/C++ 中，localtime、localtime_r 是常用的将时间戳转向时间结构体的方法 . 
我们都知道 localtime 是不可重入的，也就是非线程安全的。他只会返回一个指向转换后时间结构体的指针，当函数重入后，上一次的值会被改写成当前的结果。
而 localtime_r 函数需要我们自己申请内存，函数会在得出结果后将结果立马保存到我们申请的结构体中，保证结果存储，实现线程安全。

其实参考 linux 的 [manpage](https://linux.die.net/man/3/localtime_r) 页我们可以看到，这类函数其实是实现时间戳向本地时间的转换，而该类转换必然会依赖于本机的时区设置

> The localtime() function converts the calendar time timep to broken-down time representation, expressed relative to the user's specified timezone. **The function acts as if it called tzset(3) and sets the external variables tzname with information about the current timezone**. The localtime_r() function does the same, but stores the data in a user-supplied struct. **It need not set tzname, timezone, and daylight.**

manpage 中提到 localtime 会调用 tzset 去设置时区，而 localtime_r 不需要设置时区。
** 实际上 localtime_r 也会隐式的调用 tzset()， 而 tzset() 修改 timezone, altzone, daylight, and tzname 时是线程不安全的。**

我们可以做一个实验，在程序中循环获取本地时间，并使用 localtime_r 进行时间转换，打印出当前的时间格式

``` c

time_t curr_time = time(NULL);
struct tm       curr_tm;
localtime_r(&curr_time, &curr_tm)
printf("hour %s min %s\n", curr_tm.tm_hour, curr_tm.tm_min);

```

在过程中修改本机的时区设置，会发现进程中读取到的时间信息并没有随着时区变更发生变化，只有重启进程后才能重新识别。其原因也很简单，这里的 localtime 并没有使用新时区：

> localtime 等涉及到本地所在时区的函数在调用的时候会先调用 tzset() 这个函数，这一点可以通过 tzset 函数的 manpage 看出来。 tzset 完成的工作是把当前时区信息（通过 TZ 环境变量或者 /etc/localtime）读入并缓冲。事实上 tzset 在实现的时候是通过内部的 tzset_internal 函数来完成的，显式的调用 tzset 会以显式的方式告知 tzset_internal，而单独调用 localtime 的时候是以隐式的方式告知 tzset_internal，前者将强制 tzset 不管何种情况一律重新加载 TZ 信息或者 /etc/localtime，而后者则是只有在 TZ 发生变化，或者加载文件名发生变化的时候才会再次加载时区信息。因此，如果只是 /etc/localtime 的内容发生了变化，而文件名 " /etc/localtime" 没有变化，则不会再次加载时区信息，导致 localtime 函数调用仍然以老时区转换 UTC 时间到本地时间。

所以如果在使用 localtime_r 时涉及到了时区变更操作，亦会带来线程不安全性，如果其他线程强制刷新了时区缓存，可能会导致自身时区变更，之后导致时间计算错误。

解决办法：
建议在使用 localtime_r 之前手动调用 tzset() 函数，强制刷新时区。同时对于时区变更的情况下做必要的异常处理。