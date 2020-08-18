---
title: "iptables详解"
date: "2019-06-24"
toc: "true"
tags: ["linux","iptables"]
categories: ["学习笔记","linux"]
---

本文主要记录对于iptables的学习内容

# iptables详解

## 什么是iptables

 Linux内核中有一个netfilter框架，是一个包处理模块。借助这个模块及其框架，Linux完成了防火墙功能，而iptables就是该防火墙的配置工具。通过iptables，我们可以查看并配置Linux防火墙的规则。

## iptables工作方式

数据包通过Linux设备时流程如图:

![img](../../img/iptables/process.png)

### 链

之前提到防火墙是对数据包进行过滤处理，而为了更好的在各个流程处理数据包，netfilter在内核中设置了5个挂载点，当数据包到达这些挂载点的时候就会触发相应的处理，这五个挂载点就是iptables的五条链（chain）。  
**当数据包经过这些挂载点时，netfilter便会按照iptables中定义好的规则去匹配数据包，并对匹配到的数据包执行相应的动作**，一个挂载点自然不会只有一条规则，因此多条规则处在同一挂载点上，也正是为什么这些挂载点会被称为链。

- **PRE_ROUTING**:数据包进入路由表之前
- **INPUT**:通过路由表后目的地为本机
- **FORWARD**:通过路由表后目的地不为本机
- **OUTPUT**:由本机产生向外转发
- **POST_ROUTING**:发送到网卡接口之前

因此，在Linux上常见的数据包流向一般包括:

- 到本机应用程序的数据包:PRE_ROUTING -> INPUT->应用程序
- 经过本机进行转发的数据包:PRE_ROUTING -> FORWARD -> POST_ROUTING
- 本级应用程序对外发出的数据包:应用程序-> OUTPUT -> POST_ROUTING

### 表

iptables将功能类似的规则统一划分到一个表里进行管理，目前其提供了四个表:

 **raw**

是自1.2.9以后版本的iptables新增的表，主要用于决定数据包是否被状态跟踪机制处理，一般在高并发时有可能会用到，来设置一些连接不跟踪状态，防止ip_conntrack文件被撑爆。

 **mangle**

 主要用于修改数据包的TOS（Type Of Service，服务类型）、TTL（Time To Live，生存周期）指以及为数据包设置Mark标记，以实现Qos(Quality Of Service，服务质量)调整以及策略路由等应用，由于需要相应的路由设备支持，应用并不广泛。

 **nat**

 主要用于修改数据包的IP地址、端口号等信息（网络地址转换，如SNAT、DNAT、MASQUERADE、REDIRECT），一般在作为网关服务器上会用到，如果有搭建过openvpn的话，一般也会用到。

 **filter**

 主要用于对数据包进行过滤，有DROP、ACCEPT、REJECT、LOG等，一般在INPUT 和FORWARD 链用得最多，也是出现频率最高的。

### 表与链的关系

通过上面的图我们也能看出，表和链之间存在一定的关系。实际上某些功能的规则（即某个表）只需要在对应的位置生效即可，不需要应用于全部的链上，可以减少每个链上规则的数目，加速匹配过程。

如下表所示，表明了每个链上存在哪些表（或者是哪些表存在于哪些链）

|             | raw  | mangle | nat  | filter |
| :---------- | ---- | ------ | ---- | ------ |
| prerouting  | Y    | Y      | Y    |        |
| input       |      | Y      |      | Y      |
| output      | Y    | Y      | Y    | Y      |
| forward     |      | Y      |      | Y      |
| postrouting |      | Y      | Y    |        |

**可见一条链上是可以有多个表的，因此表与表之间也存在优先级，raw>mangle >nat> filter**

### 规则

之前提到过netfilter对数据包进行规则的匹配，如果匹配成功就执行相应动作，因此规则是由**匹配条件和执行动作**构成的。值得一提的是，**对于规则的匹配是从上到下的，如果匹配到了就会执行对应的动作而不会继续向下匹配了，因此写在前面的规则优先级较高**

**匹配条件**

  匹配条件分为基本条件和拓展条件，基本条件就是常用的匹配数据包条件，类似于源IP目的IP等等

  拓展条件通常都是使用netfilter的其他拓展模块的功能，比如连接跟踪模块等

**执行动作**

  执行动作类似于ACCEPT、DROP、REJECT等等

### 一条完整的iptables配置

```bash
iptables -t filter -I INPUT -s 192.168.1.1 -j DROP
```

该配置为filter表 INPUT链 指定源IP为192.168.1.1的数据包都采用DROP动作

# iptables命令使用

## 链管理

**-P :设置默认策略开或关**

  `iptables -P INPUT (DROP|ACCEPT)`

**-F: FLASH，清空规则链**

  `iptables -t nat -F PREROUTING` 清空nat表PREROUTING链

**-N:NEW 支持用户新建一个链**

  `iptables -N new_chain`

**-X: 用于删除用户自定义的空链**

  使用方法与-N相同，但是在删除之前必须要将里面的链清空

**-E:用来Rename chain主要是用来给用户自定义的链重命名**

  `iptables -E oldname newname`

**-Z:清空链，及链中默认规则的计数器的（有两个计数器，被匹配到多少个数据包，多少个字节)**

## 规则管理

**-A:追加，在当前链的最后新增一个规则**

**-I num : 插入，把当前规则插入为第几条**

  -I 3 :插入为第三条

**-R num:replace 替换/修改第几条规则**

  iptables -R 3

**-D num:删除，指定删除第几条规则**

## 配置查看

-L参数，用以查看当前配置,附加参数:

- -n:直接显示ip，如果不加则会将ip反向解析成主机名
- -v:显示详细信息，v越多越详细
- -x:在计数器上显示精确值，不做单位换算
- --line-numbers : 显示规则的行号
- -t:显示指定的表的所有规则
- 尾接链名:显示制定的链的规则

## 匹配参数

**-p :协议**

**-s:源地址**

**-d:目的地址**

**-i:数据包入口**

**-o:数据包出口**

**-m:使用拓展模块**

>  **-m tcp:使用tcp扩展:** 
>
> `--dport ,--sport (21|21-23)` :指定端口（连续端口）
>
> `--tcp-flags ALL SYN,ACK`匹配TCP报文的标志位
>
> `--syn`匹配新建连接的TCP报文
>
> **-m multiport :使用多端口扩展**
>
> `--dports ,--sports 22,80` :指定多端口（不连续端口）
>
> **-m iprange:连续地址范围扩展**
>
> `--src-range 192.168.1.127-192.168.1.146`
>
> **-m string:应用层数据字节序列扩展**
>
> `--algo bm --string "str"` algo指定算法（bm|kmp） string指定内容
>
> **-m icmp:icmp报文扩展**
>
> `--icmp-type 0 ` 匹配icmp报文的具体类型
>
> **-m state:状态模块扩展**

## 执行动作

**DROP:丢弃**

**REJECT:明示拒绝**

**ACCEPT:接受**

**SNAT:源地址转换**

`-j SNAT --to-source 172.16.100.1` 将数据包源IP转换为172.16.100.1

**MASQUERADE:动态伪装，将数据包源地址转换为出口网卡的IP**
`-j MASQUERADE`

**DNAT:目的地址转换，一般用于端口映射**

`-dport 80 -j DNAT --to-destination 192.168.1.1`将对80端口访问的目的地址转换到192.168.1.1

**REDIRECT:重定向:主要用于实现端口重定向**

**MARK:给数据包打防火墙标记**

**RETURN:返回，在自定义链执行完毕后使用返回，来返回原规则链**

**LOG:记录报文相关信息到日志**

**自定义链名:引用自定义链**

# 参考资料

[Linux iptables原理--数据包流向](https://www.cnblogs.com/zejin2008/p/5919550.html)

[朱双印的博客](http://www.zsythink.net/archives/tag/iptables/)

[iptables命令、规则、参数详解](https://www.cnblogs.com/zclzhao/p/5081590.html)

