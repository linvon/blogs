---
title: "Windows下Clion的配置与使用"
date: "2018-12-04"
tags: ["Cygwin", "CLion"]
categories: ["学习笔记", "零散知识"]
---
# Clion的下载与安装

该步骤省略，比较简单

# Cygwin下载与安装

选择Cygwin来配置编译环境，首先[下载](http://cygwin.com/install.html)对应安装包。
打开后选择安装方式为 `Install from Internet`

![1](/img/clion/1.png)

调整好安装目录后选择网络连接方式为 `Direct Connection`

![2](/img/clion/2.png)

之后可以选择下载镜像，选择163镜像，若无对应选项可以自行搜索镜像地址添加。

![3](/img/clion/3.png)

接下来选择要安装的模块，直接搜索gcc-core、gcc-g++、make、gdb、binutils，cmake来安装部署C++的环境，需要安装的模块都在devel目录下面，点击该目录下面搜索到的对应模块的Skip选项，将其调整为版本号即为选择安装，之后便可以下载安装。

![4](/img/clion/4.png)

如果需要离线安装，则需要先进行一遍在线安装获取离线安装包

> - 获得离线安装包
> 在线安装时，指定过下载文件存放的位置。在线安装完成后，到安装目录下面有个文件夹为“http%3a%2f%2fmirrors.163.com%2fcygwin%2f”。
> 这个文件夹和setup.exe共同组成离线安装包。
> - 安装
> 把上述提到的两个文件夹拷贝到需要离线安装的机器上后，启动setup
> 与在线安装有两点不同：
> 1、要选择从Local安装
> 2、选择拷贝过来的文件夹为安装包，同时记得也需要选择安装模块。  

# Clion的配置

安装完成后再Clion中打开Settings，进入Toolchains面板，设置Cygwin目录后其他可以自动配置，至此配置即可完成。

![5](/img/clion/5.png)

Tips:Clion编译项目时，记得将所有有执行的文件加入CMakeList的add_executable中，否则会出现链接错误