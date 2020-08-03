---
title: "CentOS7.4扩容物理硬盘根分区"
date: "2019-07-04"
tags: ["linux"]
categories: ["学习笔记","linux"]
---

CentOS现在大部分都会使用LVM，而对于使用了LVM的系统，进行根分区扩容比较简单，网络上能搜到很多相关的资料，就不在赘述。这里主要说一下如果没有启用LVM，而是使用了整块物理磁盘做了分区后的根分区如何扩容

首先这种情况下是不能对根分区进行热操作的，系统一般都运行在根分区上，无法对挂载的分区进行umount。所以首先要进入CentOS的救援模式（[CentOS7.4进入救援模式](https://blog.csdn.net/sunkuo1hao/article/details/79925177)），需要挂载光驱载入其ISO镜像。

在救援模式里使用fdisk 删除分区并重新分配分区，因为要调整根分区，所以需要将所有的分区都删除掉，并且重新分配。[linux 使用fdisk分区扩容](https://www.cnblogs.com/chenmh/p/5096592.html)

重启后使用df命令查看，此时应该已经完成了分区调整。如果没有变化，检查如果是xfs的文件系统，其实是super-block损坏，找不到文件系统，使用xfs_growfs /dev/vdax 生成文件系统即可