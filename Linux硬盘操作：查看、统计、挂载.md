---
title: "Linux磁盘操作：查看、统计、挂载"
date: "2019-04-15"
tags: ["linux","磁盘"]
categories: ["学习笔记", "linux"]
---
# **df:查看磁盘情况**

> 使用`df`命令可以查看当前磁盘使用情况，-h格式化大小单位，可以看到文件系统、大小、使用量以及挂载点  

![df](/img/LinuxDisk/df.png)  



# **du：查看空间占用**

> 使用`du`命令可以查看目录占用的空间大小，同样使用-h来格式化大小单位，可以指定目录，并且可以使用--max-depth num参数来指定查看的目录层级，比如查看/etc子级目录的占用：`du -h /etc/ --max-depth 1`  

![du](/img/LinuxDisk/du.png)  



# **查看磁盘挂载**

> Linux的磁盘都是挂载到某一个目录上的，当添加新磁盘时需要将该磁盘重新挂载，df命令看不到未挂载的磁盘  
>使用`lsblk`来查看所有的磁盘，以及他们当前的挂载点（可能需要root权限） 
![lsblk](/img/LinuxDisk/lsblk.png)  
使用`mount disk dir`来挂载磁盘如`mount /dev/sda2 /opt/data`  
挂载后要到`/etc/fstab`下配置挂载信息，添加一条记录，可以复制已有的记录，否则重启后挂载信息会丢失  
添加完毕以后可以执行`sudo mount -a`测试fstab文件是否能正常运行,查看是否挂载成功。  