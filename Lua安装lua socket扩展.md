---
title: "Lua安装lua socket扩展"
date: "2019-07-02"
tags: ["luasocket","lua"]
categories: ["学习笔记", "lua"]
---

[LuaSocket](http://w3.impa.br/~diego/software/luasocket/introduction.html)是Lua的一个通信库，实现了TCP、UDP等一些基础网络操作，并且实现了HTTP、FTP等应用的封装，还是比较实用的

``` bash
#直接下载源码进行编译
git clone https://github.com/diegonehab/luasocket
cd luasocket
make

#之后执行make install进行安装，默认安装lua5.1版本的支持，执行make install-both会自动检查对应版本进行安装
make install-both


#install时可以看到以下输出，如果需要将编译好的库文件直接拿到其他设备上使用，只需要将这些.lua和.so文件保留相对路径移动到对应的lua引用库即可
install -d /usr/local/share/lua/5.3
install -m644 ltn12.lua socket.lua mime.lua /usr/local/share/lua/5.3
install -d /usr/local/share/lua/5.3/socket
install -m644 http.lua url.lua tp.lua ftp.lua headers.lua smtp.lua /usr/local/share/lua/5.3/socket
install -d /usr/local/lib/lua/5.3/socket
install socket-3.0-rc1.so /usr/local/lib/lua/5.3/socket/core.so
install -d /usr/local/lib/lua/5.3/mime
install mime-1.0.3.so /usr/local/lib/lua/5.3/mime/core.so
```

安装完成之后在Lua内执行以下代码就可以使用了

```lua
require ("socket")

local ftp = require ("socket.ftp") --使用ftp模块
```

