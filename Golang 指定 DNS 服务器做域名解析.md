---
title: "Golang 指定 DNS 服务器做域名解析"
date: "2020-11-30"
categories: ["学习笔记"]
---

在 Golang 中，官方已经提供了 net 包来做 DNS 的解析处理，但不足之处是其功能是从默认的 `etc/resolve.conf` 中取 DNS 服务器列表再做域名解析，无法手动指定用于解析的 DNS 服务器。

我们可以手动创建一个 Resolver ，固定配置好地址后，便可以通过它来做 DNS 解析，从而实现指定 DNS 服务器地址的功能，具体代码如下：

``` go
r := &net.Resolver{
  PreferGo: true,
  Dial: func(ctx context.Context, network, address string) (net.Conn, error) {
    d := net.Dialer{
      Timeout: 10 * time.Second,
    }
    return d.DialContext(ctx, "udp", "8.8.8.8:53")
  },
}

ips, _ = r.LookupHost(context.Background(), "www.google.com")
```




