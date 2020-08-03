---
title: "Linux文件大小浅谈"
date: "2019-06-30"
tags: ["linux","文件系统"]
toc: "true"
categories: ["学习笔记","linux"]
---

在使用Linux过程中，du与df是经常会用到的命令，但有时候我们会发现，du与df的结果会出现不一致的情况。这里简单分析一下可能的原因与其背后的原理。

## du与df命令都代表什么？

#### du 的工作原理

du 命令会对待统计文件逐个调用fstat这个系统调用，获取文件大小。它的数据是基于文件获取的，所以可能跨越多个分区操作。

#### df 的工作原理

df 命令使用的是statfs这个系统调用，直接读取分区的超级块信息获取分区使用情况。它的数据是基于分区元数据的，所以只能针对整个分区。

## df的结果比du的大？

#### 文件inode使用较多

> 常用的ext3/4系统都是采用inode+block来存储文件的
>
> **inode 称为索引节点**，用来存储文件的元信息，同时指向**存储文件实际内容的block块**。具体包含以下内容：
>
> - 文件的字节数
> - 文件拥有者的User ID
> - 文件的Group ID
> - 文件的读、写、执行权限
> - 文件的时间戳，共有三个：change time, modify time, access time
> - 链接数，即有多少文件名指向这个inode
> - 文件数据block的位置
>
>  **inode 是磁盘上的一块存储空间，Centos6 非启动分区inode默认大小256 字节，Centos5 是128 字节。**
>
> **inode 的表现形式是一串数字，不同的文件对应的inode(一串数字) 在文件系统里是唯一的。**
>
> **inode 节点号相同的文件，互为硬连接文件，可以认为是一个文件的不同入口。**
>
> **一个文件在创建后至少要占用一个indoe和一个block**
>
>  **block是用来存储实际数据的，它的大小一般有1k，2k，4k几种。如果一个文件很大（4G），可能需要占用多个block；如果文件很小（0.01k），至少占一个block，并且这个block的剩余空间浪费了，即无法再存储其它数据**。这也会引入一个问题：如果文件很大时，会占用很多block，而索引到这些block需要很多的inode，因此inode有种特殊的记录方式：将inode记录block号码的区域定义为12个直接、一个间接、一个双间接与一个三间接记录区。间接代表使用记录的block再去记录inode，那么在block size为1K、inode占用为4bytes的情况下，一个间接能记录1K/4b = 256个inode。那么一个inode能记录的block总大小，也就是文件系统允许的单个文件最大容量就是`12+blocksize/inodesize+(blocksize/inodesize)^2+(blocksize/inodesize)^3`,1K/4b的情况下，就是大概16GB左右
>
> 

如果因为blocksize等设定不合适或其他原因，导致inode消耗数量特别多，实际也会占用了大量的磁盘空间，导致df的结果会大于du统计的文件大小	

#### 小文件太多，文件小于一个block的大小

上面提到文件大小如果小于一个blocksize的大小，也是要占用一个inode和block的，因此如果存在大量的小文件，会消耗大量的inode和block，导致磁盘空间被大量占用，而实际的文件总大小却很小。

#### 已经被删除的文件仍在被程序使用

> linux的文件名是存在父目录的bolck里面，并指向这个文件的inode节点，这个文件的inode节点再标记指向存放这个文件的bolck的数据块。我们删除一个文件，实际上并不清除inode节点和block的数据。只是在这个文件的父目录里面的bolck中，删除这个文件的名字，从而使这个文件名消失，并且无法指向这个文件的inode节点。当没有文件名指向这个inode 节点的时候，会同时释放inode节点和存放这个文件的数据块，并更新inode MAP和block MAP，今后让这些位置可以用于放置其他文件数据。

因此当文件被删除而进程句柄还在占用inode时，该block仍不会清空，还会继续占用磁盘空间，分区superblock中的信息也就不会更改，df仍旧会统计这个被删除了的文件占用的block，但du已经不会再统计它的大小了，就会造成df比du的结果要大的情况

>  **Superblock (超级区块)**
>
> 记录的信息主要有：
>
> - block 与inode 的总量；
> - 未使用与已使用的inode / block 数量；
> - block 与inode 的大小(block 为1, 2, 4K，inode 为128bytes 或256bytes)；
> - filesystem 的挂载时间、最近一次写入资料的时间、最近一次检验磁盘(fsck) 的时间等档案系统的相关信息；
> - 一个valid bit 数值，若此档案系统已被挂载，则valid bit 为0 ，若未被挂载，则valid bit 为1 。
>
> 一般来说， superblock的大小为1024bytes。

## du的结果比df的大？

#### 统计目录下挂载了其他文件系统

如果统计的目录下挂载了其他文件系统或分区，那么du也会对这些部分进行统计，而df仅会对当前分区的superblock进行统计

#### 文件链接到其他分区或跨分区进行了统计

du可以跨分区统计，因此不同分区符合条件的文件都会被统计到，而df只统计当前分区

## ls -h的结果比du、df的大？

#### 稀疏文件系统

> 稀疏文件，是UNIX类和NTFS等文件系统的一个特性。稀疏文件就是在文件中留有很多空余空间，留备将来插入数据使用。稀疏文件以64KB（不同文件系统不同）为单位增量增长，因此磁盘上稀疏文件的大小总是64KB的倍数。
>
> 在UNIX文件操作中，文件位移量可以大于文件的当前长度，在这种情况下，对该文件的下一次写将延长该文件，并在文件中构成一个空洞。位于文件中但没有写过的字节都被设为0。文件系统存储稀疏文件时，inode索引节点中，只给出实际占用磁盘空间的block号。数据为0的部分并不会分配block，因此文件空洞部分不占用磁盘空间，而不为0部分文件所占用的磁盘空间仍然是连续的

当文件系统创建了稀疏文件时，会看到这个文件的大小很大，但其实并没有占用实际的磁盘空间

## 拓展内容

#### 文件系统最大空间容量

> 块地址索引空间：
>
> 32bit的块索引空间：就是最大只能划出2^32个blocks，多一个block，都没有序号分给它了
>
> EXT2、3：
> 在ext3文件系统中，采用32bit的块索引空间，并且采用的无符号int整型，因此一个分区的最大block数量为：2^32个
> 则最大利用空间为4KB * 2^32 = 2^44 = 16TB
> 由此，得知在ext3文件系统中，当block为4K时，一个分区的空间将最大，且最大空间为16TB。
>
> EXT4：
> ext4使用48bit的块地址索引空间，因此在64位系统下，block为4k的情况下，大小为：
> 2^48 * 4KB = 2^60KB =1024*1024TB =1024PB = 1EB

#### 扇区大小与blocksize

磁盘里面的和文件系统里面的两个基本单位都叫block size，但是实际上是两个东西，一个是硬盘扇区的大小，应该叫做sector size；一个是文件系统的block的大小。磁盘扇区一般都是512bytes。

## 参考链接

[Linux inode、block、文件类型、软硬链接等相关文件的知识](https://blog.csdn.net/vic_qxz/article/details/80536673)

[linux下du和df结果不一致的原因及处理](http://blog.itpub.net/26964624/viewspace-2564207/)

[Linux 深入理解inode/block/superblock](https://blog.csdn.net/Ohmyberry/article/details/80427492)

[Ext-文件系统支持多大空间怎么算](https://blog.csdn.net/Angel_94/article/details/87454980 )

[详细分析du和df的统计结果为什么不一样](https://www.cnblogs.com/f-ck-need-u/p/8659301.html)

[稀疏文件（Sparse File）](https://blog.csdn.net/tiandyoin/article/details/43405369)