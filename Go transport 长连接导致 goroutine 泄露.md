---
title: "Go transport 长连接导致 goroutine 泄露"
date: "2021-01-19"
toc: "false"
---



记录一个近期发现的线上服务 goroutine 泄露问题，通过监控发现服务存在缓慢的内存泄露，通过 pprof 工具查询其 goroutine 数量，`http://ip:port/debug/pprof/goroutine?debug=1` 发现其存在 goroutine 泄露，泄露的 goroutine 达到 20000+，在 debug 页面中可以看到类似如下的展示

``` go
#	0x42f8b4	internal/poll.runtime_pollWait+0x54		/usr/local/go/src/runtime/netpoll.go:203
#	0x4cb714	internal/poll.(*pollDesc).wait+0x44		/usr/local/go/src/internal/poll/fd_poll_runtime.go:87
#	0x4cc5aa	internal/poll.(*pollDesc).waitRead+0x19a	/usr/local/go/src/internal/poll/fd_poll_runtime.go:92
#	0x4cc58c	internal/poll.(*FD).Read+0x17c			/usr/local/go/src/internal/poll/fd_unix.go:169
#	0x5414be	net.(*netFD).Read+0x4e				/usr/local/go/src/net/fd_unix.go:202
#	0x553dad	net.(*conn).Read+0x8d				/usr/local/go/src/net/net.go:184
#	0x7045b4	net/http.(*persistConn).Read+0x74		/usr/local/go/src/net/http/transport.go:1825
#	0x5c9ae2	bufio.(*Reader).fill+0x102			/usr/local/go/src/bufio/bufio.go:100
#	0x5c9c4e	bufio.(*Reader).Peek+0x4e			/usr/local/go/src/bufio/bufio.go:138
#	0x705247	net/http.(*persistConn).readLoop+0x1a7		/usr/local/go/src/net/http/transport.go:1978
```

将 debug 模式改为2，`http://ip:port/debug/pprof/goroutine?debug=2`，可以看到如下的调用栈

``` go
goroutine 2280278 [IO wait, 435 minutes]:
internal/poll.runtime_pollWait(0x7f36c5cc9aa0, 0x72, 0x18d73e0)
	/usr/local/go/src/runtime/netpoll.go:220 +0x55
internal/poll.(*pollDesc).wait(0xc01a65b698, 0x72, 0xc0313d2000, 0x1000, 0x1000)
	/usr/local/go/src/internal/poll/fd_poll_runtime.go:87 +0x45
internal/poll.(*pollDesc).waitRead(...)
	/usr/local/go/src/internal/poll/fd_poll_runtime.go:92
internal/poll.(*FD).Read(0xc01a65b680, 0xc0313d2000, 0x1000, 0x1000, 0x0, 0x0, 0x0)
	/usr/local/go/src/internal/poll/fd_unix.go:159 +0x1b1
net.(*netFD).Read(0xc01a65b680, 0xc0313d2000, 0x1000, 0x1000, 0xc0002015c0, 0xc020906d98, 0xc020906c20)
	/usr/local/go/src/net/fd_posix.go:55 +0x4f
net.(*conn).Read(0xc0205ea7d0, 0xc0313d2000, 0x1000, 0x1000, 0x0, 0x0, 0x0)
	/usr/local/go/src/net/net.go:182 +0x8e
net/http.(*persistConn).Read(0xc0006e9320, 0xc0313d2000, 0x1000, 0x1000, 0xc020906eb0, 0x46ca40, 0xc020906eb0)
	/usr/local/go/src/net/http/transport.go:1887 +0x77
bufio.(*Reader).fill(0xc01f7f5ce0)
	/usr/local/go/src/bufio/bufio.go:101 +0x105
bufio.(*Reader).Peek(0xc01f7f5ce0, 0x1, 0x2, 0x0, 0x0, 0x0, 0xc02691cfc0)
	/usr/local/go/src/bufio/bufio.go:139 +0x4f
net/http.(*persistConn).readLoop(0xc0006e9320)
	/usr/local/go/src/net/http/transport.go:2040 +0x1a8
created by net/http.(*Transport).dialConn
	/usr/local/go/src/net/http/transport.go:1708 +0xcb7
```

比较困惑的是，goroutine 的调用是由底层源码发起的，可以猜测是某些基础库的使用方式有误，注意到代码是 http 的 lib 相关，能看到明显的 pollWait、persistConn、IO wait 字眼，为了定位是什么位置的代码引用，我们可以去线上查找当前服务进程打开的连接句柄

使用 pidof 或者 ps 命令找到自己服务进程的 pid，之后 `lsof -p pid` 来查看进程打开的句柄，能看到许多异常的建立起的连接

``` shell
srv 17987 work   30u     IPv4 30417287        0t0      TCP localhost:49788->localhost:fmtp (ESTABLISHED)
srv 17987 work   31u     IPv4 30697379        0t0      TCP localhost:61063->localhost:fmtp (ESTABLISHED)
srv 17987 work   32u     IPv4 30565197        0t0      TCP localhost:34199->localhost:fmtp (ESTABLISHED)
srv 17987 work   33u     IPv4 30574610        0t0      TCP localhost:34687->localhost:fmtp (ESTABLISHED)
srv 17987 work   34u     IPv4 29940569        0t0      TCP localhost:47346->localhost:fmtp (ESTABLISHED)
srv 17987 work   35u     IPv4 30708877        0t0      TCP localhost:61533->localhost:fmtp (ESTABLISHED)
srv 17987 work   36u     IPv4 30164366        0t0      TCP localhost:43103->localhost:fmtp (ESTABLISHED)
srv 17987 work   37u     IPv4 30563301        0t0      TCP localhost:34663->localhost:fmtp (ESTABLISHED)
srv 17987 work   38u     IPv4 30709910        0t0      TCP localhost:61559->localhost:fmtp (ESTABLISHED)
srv 17987 work   39u     IPv4 30238360        0t0      TCP localhost:58203->localhost:fmtp (ESTABLISHED)
srv 17987 work   40u     IPv4 30342583        0t0      TCP localhost:35954->localhost:fmtp (ESTABLISHED)
srv 17987 work   41u     IPv4 30838726        0t0      TCP localhost:42024->localhost:fmtp (ESTABLISHED)
srv 17987 work   42u     IPv4 30415497        0t0      TCP localhost:49098->localhost:fmtp (ESTABLISHED)
srv 17987 work   43u     IPv4 30840173        0t0      TCP localhost:42478->localhost:fmtp (ESTABLISHED)
srv 17987 work   44u     IPv4 30416471        0t0      TCP localhost:49762->localhost:fmtp (ESTABLISHED)
srv 17987 work   45u     IPv4 30839661        0t0      TCP localhost:42500->localhost:fmtp (ESTABLISHED)
srv 17987 work   46u     IPv4 31288225        0t0      TCP localhost:39302->localhost:fmtp (ESTABLISHED)
srv 17987 work   47u     IPv4 30997795        0t0      TCP localhost:27359->localhost:fmtp (ESTABLISHED)
srv 17987 work   48u     IPv4 31009992        0t0      TCP localhost:27841->localhost:fmtp (ESTABLISHED)
srv 17987 work   49u     IPv4 30559437        0t0      TCP localhost:33999->localhost:fmtp (ESTABLISHED)
```

如果 grep -c 统计一下，可以发现其数量和泄露的 goroutine 数量几乎一致，可以判断这就是我们要找的泄露连接。使用 `lsof -P -p pid`打印完整端口号，找出连接对应的服务，在我们的例子中，是存在大量连接到 consul 的连接。

为什么 Transport 会导致泄露呢？ 我们跟踪进源码，在 Transport 的注释中可以看到

``` go
// Transports should be reused instead of created as needed.
// Transports are safe for concurrent use by multiple goroutines.
```

意为 Transports 应该被重复使用而不是重复创建，其在多个 goroutine 之间使用是并发安全的。而实际上 Transport 维护的是一个简单的长连接，goroutine 完成后，其所使用的 fd 并不能被释放，因此便会导致 goroutine 产生泄露。

再回到业务代码中，在对应位置查找对于 Transport 的使用，发现有明显的重新创建行为，而正是该行为导致了大量的 goroutine 泄露，我们只需要将使用到 Transport 的初始化为全局变量，创建一次后重复使用便可以避免该问题了。