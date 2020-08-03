---
title: "认识与了解OpenWrt"
date: "2019-05-15"
tags: ["openwrt"]
toc: "true"
categories: ["学习笔记", "openwrt"]
---
# **什么是OpenWrt？**
OpenWrt是一个为嵌入式设备开发的高扩展度的GNU/Linux发行版，是一种主流的路由器固件  
开源、支持多种架构（ARM Mips x86等）、高可扩展性、有大量软件工具包  
官网：[openwrt.org](http://openwrt.org)  
[Github](https://github.com/openwrt/openwrt)  
在Openwrt的官网上能找到详细的官方文档，对整个OpenWrt所实现、支持的功能以及各个版本都有很详细的介绍，并且可以选择语言（虽然大部分中文翻译都没有完工）
![org](/img/OpenWrt/org.png)

# **OpenWrt的编译**

## **固件编译**
此前提到OpenWrt是开源的，谁都可以下载源码直接进行编译。常见的Linux发行版如Ubuntu、CentOS都可以搭建其编译环境。  
OpenWrt的编译过程会将系统打包压缩成一个可刷写的固件。如果要修改OpenWrt的固件内容，例如向其中添加自己的程序，或者修改默认的配置、内核实现等，便可以将自己修改的内容拷贝到源码中，从源码再编译。  
在OpenWrt系统编译配置方面，采用的是make menuconfig命令，该命令常用于针对于嵌入式Linux的内核编译选项修改。在OpenWrt上，除了内核外，还可以修改很多其他选项。  
![menuconfig](/img/OpenWrt/menuconfig.png)  

![makeluci](/img/OpenWrt/makeluci.png)  
在完成编译选项的配置之后，可以直接进入源码的source/目录 执行 make V=sc -j4命令进行固件的编译  

## **第三方程序编译**
如果想要自己编写程序放入OpenWrt上使用，首先要确定使用的硬件的CPU架构，毕竟大部分嵌入式设备都不是x86的，无法跟我们常用的主机设备共用编译工具，因此这些程序都需要在对应的编译环境上实用工具链交叉编译。  

# **OpenWrt的原生工具**

## **UCI:Unified Configuration Interface**
意为统一配置接口，用来操作OpenWrt上所有配置项的集中接口。  
这个工具可以直接执行命令传输参数就能达到修改系统参数的目的，同时也会有相关的lua、c等语言可调用的库函数，实现通过一个工具一种规则来配置整个OpenWrt  

OpenWrt的配置文件细化到每一个模块，都存放在/etc/config路径下，均是按照一定格式来保存的纯文本文件。除了通过uci指定的命令来修改，也可以通过vi手动修改配置文件。  
**要注意uci工具自身与提供给lua、c的uci库在实现的功能方面是由少许区别的**  

- 对于uci工具，可以直接在终端查看其操作指示  
  ![uci](/img/OpenWrt/uci.jpeg)  

- 在lua内，可以调用uci.so，其本质是uci工具提供给Lua的库，其具体实现的函数如下：  
![uciso](/img/OpenWrt/uciso.png)  
 具体可以参考：https://github.com/wangshawn/uci/blob/master/lua/uci.c  

- 在Lua内也可以直接调用uci.lua库，这是luci自身的库，在uci的基础上做了一些改进，其实现的方法可以参考[uci.lua](https://htmlpreview.github.io/? https://raw.githubusercontent.com/openwrt/luci/master/documentation/api/modules/luci.model.uci.html)  

三种方式各有利弊，实际应用时根据需要选择引用即可，**使用uci时，还有一些需要注意的点**：  

1. 在修改完配置后需要commit该配置文件，意为提交更改的内容，这样真正的文本文件才会被更改，否在改动内容仍缓存在uci的进程里。
2. 不同的进程去调用uci时都是独立的，甚至是lua文件去调用某个lua接口，这两个lua调用到的uci也会是相互独立的，因此在跨文件处理配置或者需要反复读取配置的操作的时候需要考虑这些问题。
3. uci类似数据库支持很多保存、提交、回滚操作，用以处理复杂的事务逻辑。
4. 除了更新改动到配置文件外，还需要让相关进程重新加载配置。这时需要去执行程序对应的/etc/init.d目录下的脚本，进行reload或者restart操作。读取配置的动作都是写在脚本里的，所以不执行脚本单单重启进程是没法完成配置的重新读取的。

## **ubus**
ubus是OpenWrt进行守护进程和应用程序间的通讯的工具，其客户端可以请求服务器上注册的服务。并且提供HTTP和Lua的操作扩展  
    ![ubus](/img/OpenWrt/ubus.jpeg)  
ubus list可以查看当前已经注册的服务，主要使用的是network。  
![ubusli](/img/OpenWrt/ubusli.png)  
使用-v参数可以查看某一个服务已经注册的方法，然后使用ubus call来调用  
 ![ubusliv](/img/OpenWrt/ubusliv.png)  
![ubuslivn](/img/OpenWrt/ubuslivn.png)  
在OpenWrt上，像类似于network reload 、ifup wan、ifstatus lan、获取网卡IP，最终都是用ubus去执行的  
## **luci库**
之前提到编译选项中可以选择luci的操作库，这一块是luci自身实现的一些便捷操作的lua接口，类似于构建http请求操作、ip地址解析、json操作、sys系统操作、util基本工具、i18n等等，非常方便  

像类似于ip操作库，可以直接解析普通的ip/掩码格式，也能直接解析CIDR格式的IP地址，util工具中自带lua的数组打印，sys中可以直接获取系统的当前环境变量  
整个库还是比较庞大，详细的可以参考：[接口文档](https://htmlpreview.github.io/?https://raw.githubusercontent.com/openwrt/luci/master/documentation/api/index.html)  
## **luci_debug** 
这个操作界面是另一个精髓工具，它可以用来真正的配置OpenWrt，其包含的功能如下：  
![lucidebug](/img/OpenWrt/lucidebug.png)  
其中状态页主要是用于查看系统的信息，系统页用于配置系统各项参数，网路页专门用于设置网络，我们着重挑几项作介绍。  

- 系统->系统
在系统页面中，可以设置语言，如果语言没有中文，则需要要在编译时配置选项中加入中文。  
![lang](/img/OpenWrt/lang.png)  

- 系统->软件包
之前提到过OpenWrt自身有3000+软件包可用。设备上所有安装的软件包都会显示在这里，如果设备接入了互联网的话，可以在线更新可用软件列表，需要使用什么软件的话直接可以下载安装。  
![software](/img/OpenWrt/software.png)  
实际上这个页面对应的功能是依赖opkg工具来实现的，这是OpenWrt上的一个软件包管理工具。在后台直接使用opkg命令可以看到很多可操作内容。  

- 系统->备份/升级
![upgrade](/img/OpenWrt/upgrade.png)  
该功能是依赖工具sysupgrade来实现的，后台可以直接调用并查看相关命令。 
![sysup](/img/OpenWrt/sysup.png)  
第一个是可以选择备份和恢复配置，这一项可以将选择保留的配置列表中内容做备份或者用相关备份来恢复。也就是对应的backup-command中的内容。在配置选项卡中可以配置和查看当前选择保留的配置文件列表，手动追加的列表保存在/etc/config/sysupgrade.conf中。对于具体保存哪些文件，可以直接点打开列表来查看。即对应sysupgrade -l。  
第二个便是刷写固件，实际上我们的前端升级功能也是依赖与此，其本质上是保留配置文件的刷机。在该页面可以选择不保留配置，这样刷写固件后设备就是全新的，对应sysupgrade -n参数。  

- 网络->接口
    - 物理设置方面主要是选择该网络配置要绑定哪一个实际物理网卡。如果选择了创建桥接，那么选项就会变为多选，被选择的多个网卡会建立桥接，成为一个桥口。  
    ![](/img/OpenWrt/eth.png)  
    
    - 防火墙设置是为了分配该网卡属于哪一个区域，一般只有lan和wan两个防火墙区域，即代表lan、wan口，功能性网卡一定要划分区域，防火墙会对进出及转发的数据流量做一定的限制。  
![](/img/OpenWrt/firew.png)  

- 网络->防火墙
    - 基本设置，这一项设定了整个防火墙的基础功能项。入站数据代表从网口进来的数据包，出站数据代表从网口出去的数据包，转发代表在防火墙各个区域之间转发的数据包。  
    ![](/img/OpenWrt/dffw.png)  
    
    - 针对区域的设置，首先每个区域内也会有入站数据、出站数据和转发的设定，前两项代表内容和防火墙基本设置相同，而此处的转发表示的是区域内不同网络接口之间的数据包转发，同时在下面还可以配置不同防火墙区域之间的转发规则。这些转发规则与防火墙的基本规则在生效上有优先级，具体要看修改后的iptables规则。  
    除此之外，还可以针对每个区域配置IP动态伪装（实际为SNAT），MSS钳制，以及该区域覆盖哪些网络接口  
    ![](/img/OpenWrt/zfw.png)  

# **OpenWrt的动态配置生成**

OpenWrt对于默认配置的处理方式稍有特别，它有两个配置文件是动态生成的，一个是网络配置network，另一个是系统配置system。主要说一下network：  
首先启动时board_detect脚本会去判断/etc/board.json文件是否存在，该文件是生成网络配置的模板文件。如果不存在则会去遍历执行/etc/board.d/下面的所有脚本，从而生成一个board.json，在修改默认配置的基础大项时（比如要添加几个默认网卡），要修改的是board.json的生成脚本，即修改其模板。  
之后config_generate会去判断network配置文件是否存在，如果不存在，那么这个脚本就会去根据board.json文件去生成network网络配置，针对默认配置的细节处理，可以在这个脚本中修改。  

# **OpenWrt的配置文件解析**
## **network**

```bash
config interface 'lan'                   #网络接口
        option type 'bridge'             # 接口模式，这里是桥模式
        option ifname 'eth0.1'           # 接口名称
        option proto 'static'            # 协议类型，可选项有‘dhcp’、‘pppoe’、‘3g’、‘ppp’等
        option netmask '255.255.255.0'   # 掩码
        option ipaddr '192.168.1.1'      # ip

config switch_vlan                       #虚拟vlan，用以划分网卡为多个虚拟网卡
        option device 'switch0'          # 定义设备
        option vlan '1'                  # vlan号
        option ports '1 2 3 4 6t'        # 设备网口
```

## **dhcp**

```bash
config dhcp 'lan'  
        option interface 'lan'        # 指定生成DHCP服务的网络接口
        option start '20'             # 地址池开始的IP为改网卡网段的第几个IP
        option limit '99'             # 限制客户数，即地址池大小
        option leasetime '1h'         # 租期时长，支持h和m为单位
        list 'dhcp_option' '6,114.114.114.114,8.8.8.8'   # 设定DNS
        list 'dhcp_option' '3,192.168.1.2'               # 设定默认网关
```