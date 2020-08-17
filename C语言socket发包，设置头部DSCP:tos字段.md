---
title: "C语言socket发包，设置头部DSCP/tos字段"
date: "2019-10-11"
categories: ["学习笔记"]
---



在需要指定 Linux 的 socket 发出包的头部 tos 字段时，可以直接使用内核中的函数，样例如下：
其中 m_dscp 是待设置的值


``` c

include <sys/socket.h>

//IPv4
setsockopt(m_fd, IPPROTO_IP, IP_TOS, &m_dscp, sizeof (m_dscp))

//IPv6
setsockopt(m_fd, IPPROTO_IPV6, IPV6_TCLASS, &m_dscp, sizeof (m_dscp))

```

注意此处的设置会完全使用头部的 8 位，因此如果是 DSCP 值，需要自己手动左移 2 位。
