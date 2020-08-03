---
title: "TCP\u72B6\u6001\u8BE6\u89E3"
date: "2019-10-05"
toc: true
tags: ~
categories:
- "\u5B66\u4E60\u7B14\u8BB0"
- "\u7F51\u7EDC"
...
--- 
# TCP连接状态

**LISTEN**

监听来自他方的连接请求：服务端状态，应用程序打开监听端口，处理来自客户端的TCP连接请求，状态置位LISTEN

**SYN_SENT**

客户端状态，在发送连接请求后等待匹配的连接请求：客户端通过应用程序connect()连接时，客户端TCP发送SYN标记主动建立连接，之后状态置为SYN_SENT

**SYN_RECV**

服务端状态，在收到并发送一个连接请求后，等待对方连接请求的确认：服务端状态，当收到客户端SYN封包后，服务端会发送一个SYN及ACK确认到客户端，再等待对方连接确认，这时状态为SYN_RECV

如果发现有很多SYN_RCVD状态，可能受到了SYN FLood的Dos攻击

**ESTABLISHED**

代表一个打开的连接：当客户端回复正确的ACK值后，就建立一个打开的连接，客户端和服务端就都进入ESTABLISHED状态，此时便可以PSH数据

如果客户端出现异常，没有发送FIN包就断开了连接，那么此时服务端还是会显示ESTABLISHED状态。而如果客户端再恢复运行，重新建立新连接，并不断重复这种异常正常的切换，那么就会造成服务端出现大量ESTABLISHED假连接，影响服务端的服务效率。

**FIN_WAIT1**

等待远程TCP连接中断请求或者中断请求的确认：客户端调用close()关闭连接后，TCP发出FIN标记主动关闭连接，然后进入FIN_WAIT1状态，等待远程TCP连接中断或者请求的确认。

**CLOSE_WAIT**

对端关闭连接，等待本地的关闭请求：被动关闭端TCP接收到FIN后，就发送ACK回应客户端的FIN包，然后进入CLOSE_WAIT状态，等待自身的应用程序关闭连接。

**FIN_WAIT2**

等待远端的TCP关闭请求：半关闭状态，主动关闭端(也就是客户端调用close()后)接收到ACK确认后，进入FIN_WAIT2状态。该状态下，客户端应用程序依然能接收数据，但不能再发送数据。

值得注意的是：FIN_WAIT_2 状态是没有超时的（不像TIME_WAIT 状态），这种状态下如果对方不关闭连接，那这个 FIN_WAIT_2 状态将一直保持到系统重启，越来越多的FIN_WAIT_2 状态会导致内核crash。

**LAST_ACK**

等待发向主动端的断开连接请求的确认：被动关闭端没有数据传输之后，应用程序调用close关闭连接。发送FIN包，然后进入LAST_ACK状态，等待对端的中断确认。

**TIME_WAIT**

等待足够的时间以确保远程TCP接收到连接中断请求的确认：在主动关闭端接收到FIN后，就发送ACK确认，并进入TIME-WAIT状态。TIME_WAIT状态下的TCP连接会等待2\*MSL，然后会回到CLOSED 可用状态。

> MSL：Max Segment Lifetime，最大分段生存期，指一个TCP报文在Internet上的最长生存时间。每个具体的TCP协议实现都必须选择一个确定的MSL值，RFC 1122建议是2分钟，但BSD传统实现采用了30秒，Linux可以cat /proc/sys/net/ipv4/tcp_fin_timeout看到本机的这个值

为什么TIME_WAIT状态还需要等2MSL后才能返回到CLOSED状态？

- 一方面是可靠的实现TCP全双工连接的终止，也就是当最后的ACK丢失后，被动关闭端会重发FIN，因此主动关闭端需要维持状态信息，以允许它重新发送最终的ACK。
- 另一方面，TCP在2MSL等待期间会定义这个连接(4元组)不能再使用，任何收到的其他报文都会丢弃，用来防止延迟的旧报文扰乱新的连接。

**CLOSED**

TCP连接关闭，被动关闭端在接收到ACK包后，进入CLOSED状态关闭TCP连接

**CLOSING**

CLOSING状态一般较少出现，这种是客户端和服务端同时发起了FIN主动关闭。如果客户端发送FIN主动关闭，但是没有收到服务端发来的ACK确认，而是先收到了服务端发来的FIN关闭连接，这样就会进入CLOSING状态。只要收到了对方对自己发送的FIN的ACK确认就会进入TIME_WAIT状态，因此，如果RTT(Round Trip Time TCP包的往返延时)处在一个可接受的范围内，发出的FIN会很快被ACK从而进入到TIME_WAIT状态，CLOSING状态持续的时间就特别短，因此很难看到这种状态

# TCP状态转化

![p18.png](/img/tcpstate/state.png)

**服务端常见状态迁移流程：**

 CLOSED->LISTEN->SYN收到->ESTABLISHED->CLOSE_WAIT->LAST_ACK->CLOSED

**客户端常见状态迁移流程：**

CLOSED->SYN_SENT->ESTABLISHED->FIN_WAIT_1->FIN_WAIT_2->TIME_WAIT->CLOSED

**其他可能的状态迁移：**

- LISTEN->SYN_SENT：服务端打开连接发送请求。
- SYN_SENT->SYN_RECV：服务器和客户端在 SYN_SENT 状态下如果收到 SYN 数据报，则都需要发送SYN的ACK数据报并进入 SYN_RECV 状态，准备进入 ESTABLISHED 状态。
- SYN_SENT->CLOSED：在发送超时的情况下，会返回到 CLOSED 状态。 
- SYN_RECV->LISTEN：如果收到 RST 包，会返回到 LISTEN 状态。 
- SYN_RECV->FIN_WAIT_1：可以跳过 ESTABLISHED 状态直接跳转到FIN_WAIT_1状态并等待关闭。

# TCP三次握手与四次挥手

![p16.png](/img/tcpstate/syn.png)
![p17.png](/img/tcpstate/fin.png)

其实从连接的建立与关闭过程来看TCP的状态转换会更加直观，一些状态只会出现在一个连接中的客户端或者服务端中的一端。

其实TCP的建立与断开过程都是：请求-确认请求-应答-确认应答的过程。而不同的只是建立过程是三次握手，而断开过程是四次挥手。而为什么会有这样的差别呢？主要原因是TCP是全双工的，也就是双方都可以同时接受和发送数据。在连接建立时，被动端的确认请求和应答可以放在一次交互中传输。而在断开时，主动方断开连接，被动方可能并不想断开连接，仍有数据需要传输。

# TCP同时打开与同时关闭

虽然两个应用程序同时执行主动打开的情况发生的可能性较低，但是也是可能的。每一端都发送一个SYN给对方，且每一端都使用对端所知的端口作为本地端口。TCP协议在遇到这种情况时，只会打开一条连接。这个连接的建立过程需要4次数据交换，而一个典型的连接建立只需要3次交换（即3次握手）。但多数伯克利版的TCP/IP实现并不支持同时打开。

如果两个应用程序同时发送FIN，则在发送后会首先进入FIN_WAIT_1状态。在收到对端的FIN后，回复一个ACK，会进入CLOSING状态。在收到对端的ACK后，进入TIME_WAIT状态。这种情况称为同时关闭。同时关闭也需要有4次报文交换，与典型的关闭相同。

# TCP标记位

在TCP头部有一个FLAGS字段，这个字段可以表示以下几个值：SYN, FIN, ACK, PSH, RST, URG. 

**URG**   
紧急数据标志，如果URG为1，表示本数据包中包含紧急数据。
此时紧急数据指针表示的值有效，该指针表示的是紧急数据之后的第一个字节的偏移值（即紧急数据的总长度）。  
**ACK**  
确认标志位。如果ACK为1，表示数据包中的确认号有效，用于确认数据包的接受。  
**PSH**  
推送标志，表示强迫数据传输，不能和紧急标志同时使用。用以确保数据被给予应得的优先级，并在发送或接收端处理。这个特定的标志在数据传输的开始和结束时被非常频繁地使用，影响数据在两端处理的方式。  
**RST**  
复位标志，当到达的数据包不用于当前连接时，用来复位一条连接。当RST=1时，表示出现严重错误，必须释放连接，然后再重新建立。  
**SYN**  
同步标记，用来建立连接，如果SYN=1而ACK=0,表明它是一个连接请求；如果SYN=1且ACK=1，则表示同意建立一个连接。  
**FIN**  
表示数据已经发送完毕，希望释放连接。  

# TCP KeepAlive

其实KeepAlive的原理就是TCP内嵌的一个心跳包，以服务端为例，如果当前服务端检测到超过一定时间（默认是 7,200,000 milliseconds，也就是2个小时）没有数据传输，那么会向客户端发送一个keep-alive packet（该keep-alive packet就 是ACK和当前TCP序列号减一的组合），此时客户端应该为以下三种情况之一：
         
- 客户端仍然存在且网络连接状况良好。此时客户端会返回一个ACK。服务端接收到ACK后重置计时器（复位存活定时器），在2小时后再发送探测。如果2小时内连接上有数据传输，那么在该时间基础上再向后推延2个小时。
- 客户端异常关闭或是网络断开。在这两种情况下，客户端都不会响应。服务器没有收到对其发出探测的响应，便会在一定时间（系统默认为1000 ms）后重复发送keep-alive packet，并且重复发送一定次数。
- 客户端曾经崩溃，但已经重启。这种情况下，服务器将会收到对其存活探测的响应，但该响应是一个复位，从而引起服务器对连接的终止。

对于应用程序来说，2小时的空闲时间太长。因此需要手工开启Keepalive功能并设置合理的参数。
全局设置可更改/etc/sysctl.conf，加上: 

```
net.ipv4.tcp_keepalive_intvl = 20 
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_time = 60
```

# 参考链接
[tcp中11种状态详解](https://segmentfault.com/a/1190000019620421?utm_source=tag-newest)  
[TCP标志位详解（TCP Flag）](https://blog.csdn.net/chenvast/article/details/77978367)  
[TCP连接的状态详解以及故障排查](https://blog.csdn.net/hguisu/article/details/38700899)  
