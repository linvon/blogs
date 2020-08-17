---
title: "Linux中的hook函数的使用"
date: "2019-11-03"
categories: ["学习笔记"]
---


在 Linux 中处理网络数据包经常需要使用到 hook 钩子，其实就是类似于一种触发点的机制

**hook 函数依赖的结构体**
``` c
struct nf_hook_ops
{
        struct list_head list;           // 链表成员      
        /* 下面这些需要函数编写者填入 */
        nf_hookfn *hook;              // 钩子函数指针     
        struct net_device    *dev; /* 设备 */
        int pf;                            // 协议簇 protocol flags，对于 ipv4 而言，是 PF_INET
        int hooknum;                   //hook 类型       
        int priority;                     // 优先级
};
```

其中：
**hook** 为用户定义的钩子函数，原型为：
``` c
typedef unsigned int nf_hookfn(void *priv,
                   struct sk_buff *skb,
                   const struct nf_hook_state *state);
```
nf_hookfn 执行后需要返回以下返回值：
NF_ACCEPT：  继续正常的报文处理；
NF_DROP：      将报文丢弃；
NF_STOLEN：  由钩子函数处理了该报文，不要再继续传送；
NF_QUEUE：    将报文入队，通常交由用户程序处理；
NF_REPEAT：   再次调用该钩子函数。
**dev** 为网络设备
**pf** 是协议簇 protocol flags，对于 ipv4 而言，是 PF_INET，可选值有
``` c
enum {
    NFPROTO_UNSPEC =  0,
    NFPROTO_INET   =  1,
    NFPROTO_IPV4   =  2,
    NFPROTO_ARP    =  3,
    NFPROTO_NETDEV =  5,
    NFPROTO_BRIDGE =  7,
    NFPROTO_IPV6   = 10,
    NFPROTO_DECNET = 12,
    NFPROTO_NUMPROTO,
};
```
**hooknum** 为 hook 点，可选值有以下几种，对应 Linux 的 netfilter 的 hook 点
``` c
enum nf_inet_hooks {
    NF_INET_PRE_ROUTING,
    NF_INET_LOCAL_IN,
    NF_INET_FORWARD,
    NF_INET_LOCAL_OUT,
    NF_INET_POST_ROUTING,
    NF_INET_NUMHOOKS
};
```
最后一个参数 **priority** 为 hook 优先级，内核定义了以下多种优先级:
``` c
enum nf_ip_hook_priorities       //include/linux/netfilter_ipv4.h
{
　　NF_IP_PRI_FIRST = INT_MIN,
　　NF_IP_PRI_CONNTRACK = -200,
　　NF_IP_PRI_MANGLE = -150,
　　NF_IP_PRI_NAT_DST = -100,
　　NF_IP_PRI_FILTER = 0,
　　NF_IP_PRI_NAT_SRC = 100,
　　NF_IP_PRI_LAST = INT_MAX,
};
```
