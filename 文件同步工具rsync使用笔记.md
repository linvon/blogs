---
title: "文件同步工具rsync使用笔记"
date: "2019-07-05"
tags: ["rsync","文件传输"]
categories: ["学习笔记", "零散知识"]
---

最近要用到对Linux上文件进行自动备份传输，将多台设备的备份集中到一台设备。

常见的文件传输方式有很多，但都有一定弊端。类似于wget、curl需要依赖http，不是很方便。scp、rsp等工具都要依赖ssh，当ssh加密发生改变就要修改秘钥。因此选择rsync的理由是其不依赖于ssh，内网里打通直接相当于文件服务器，传输速度很快。

rsync的简单使用命令不再赘述，这里主要说一下rsync启动服务端

使用命令`rsync —daemon`启动服务器，需要先编辑`/etc/rsyncd.conf`，配置如下

```ini
######### 全局配置参数 ##########
port=888    # 指定rsync端口。默认873 
uid = rsync # rsync服务的运行用户，默认是nobody，文件传输成功后属主将是这个uid
gid = rsync # rsync服务的运行组，默认是nobody，文件传输成功后属组将是这个gid
use chroot = no # rsync daemon在传输前是否切换到指定的path目录下，并将其监禁在内
max connections = 200 # 指定最大连接数量，0表示没有限制
timeout = 300         # 确保rsync服务器不会永远等待一个崩溃的客户端，0表示永远等待
motd file = /var/rsyncd/rsync.motd   # 客户端连接过来显示的消息
pid file = /var/run/rsyncd.pid       # 指定rsync daemon的pid文件
lock file = /var/run/rsync.lock      # 指定锁文件
log file = /var/log/rsyncd.log       # 指定rsync的日志文件，而不把日志发送给syslog
dont compress = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2  # 指定哪些文件不用进行压缩传输
 
###########下面指定模块，并设定模块配置参数，可以创建多个模块###########
[test]        # 模块ID
path = /test/ # 指定该模块的路径，该参数必须指定。启动rsync服务前该目录必须存在。rsync请求访问模块本质就是访问该路径。
ignore errors      # 忽略某些IO错误信息
read only = false  # 指定该模块是否可读写，即能否上传文件，false表示可读写，true表示可读不可写。所有模块默认不可上传
write only = false # 指定该模式是否支持下载，设置为true表示客户端不能下载。所有模块默认可下载
list = false       # 客户端请求显示模块列表时，该模块是否显示出来，设置为false则该模块为隐藏模块。默认true
hosts allow = 10.0.0.0/24 # 指定允许连接到该模块的机器，多个ip用空格隔开或者设置区间
hosts deny = 0.0.0.0/32   # 指定不允许连接到该模块的机器
auth users = rsync_backup # 指定连接到该模块的用户列表，只有列表里的用户才能连接到模块，用户名和对应密码保存在secrts file中，这里使用的不是系统用户，而是虚拟用户。不设置时，默认所有用户都能连接，但使用的是匿名连接
secrets file = /etc/rsyncd.passwd # 保存auth users用户列表的用户名和密码，每行包含一个username:passwd。由于"strict modes"默认为true，所以此文件要求非rsync daemon用户不可读写。只有启用了auth users该选项才有效。
[upload]    # 以下定义的是第二个模块
path=/upload/
read only = false
ignore errors
comment = anyone can access
```

需要注意的是:

**pid file是使用系统默认的，最好不要修改**

**每个模块的目录需要手动创建，否则客户端传不上来。**

**如果想不使用密码，就只能账户密码都不设置，客户端匿名登录**