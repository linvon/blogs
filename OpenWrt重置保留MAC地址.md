---
title: "OpenWrt重置保留MAC地址"
date: "2019-05-15"
tags: ["openwrt","mtd"]
categories: ["学习笔记", "openwrt"]
---
OpenWrt重置时会丢失网络配置，而系统的MAC地址是随机生成的，这样可能会造成MAC地址变化。但重置时mtd字符设备不会变。且mtd本身留有一段空间用来存储lan_mac，遂在启动生成配置时，检测到mtd中的mac如果为ff:ff:ff:ff:ff:ff，就把设备生成的mac写进去，再次生成时先从mtd中读取。

> MTD，Memory Technology Device即内存技术设备，在Linux内核中，引入MTD层为NOR FLASH和NAND FLASH设备提供统一接口。MTD将文件系统与底层FLASH存储器进行了隔离。  
> ![mtd](../../img/OpenWrt/mtd.png)   
> 
> 如上图所示，MTD设备通常可分为四层，从上到下依次是：设备节点、MTD设备层、MTD原始设备层、硬件驱动层。  
> 
> Flash硬件驱动层：Flash硬件驱动层负责对Flash硬件的读、写和擦除操作。MTD设备的Nand Flash芯片的驱动则drivers/mtd/nand/子目录下,Nor Flash芯片驱动位于drivers/mtd/chips/子目录下。  
> 
> MTD原始设备层：用于描述MTD原始设备的数据结构是mtd_info，它定义了大量的关于MTD的数据和操作函数。其中mtdcore.c:  MTD原始设备接口相关实现，mtdpart.c :  MTD分区接口相关实现。  
> 
> MTD设备层：基于MTD原始设备，linux系统可以定义出MTD的块设备（主设备号31）和字符设备（设备号90）。其中mtdchar.c :  MTD字符设备接口相关实现，mtdblock.c : MTD块设备接口相关实现。块设备模拟：MTD提供一个称谓mtdblock的块驱动程序，它在闪存上模拟一块硬盘，你可以将任何文件系统(如：ext2)放在模拟的闪存磁盘上，mtdblock隐藏了复杂的闪存访问过程(比如写之前先删除相关扇区的内容)，被mtdblock创建的设备节点命名为/dev/mtdblock/X，其中X是分区号。字符设备模拟：mtdchar是底层闪存设备呈现出线性特点，与文件系统的块设备特性不同，mtdchar建立的设备节点命名为/dev/mtd/X，其中X为分区号，例如，写入引导程序: dd if=bootloader.bin of=/dev/mtd/0 ；一个原始mtdchar分区的使用示例是POST错误日志，另外一个嵌入式系统使用字符闪存分区的例子是保存类似于PC的CMOS、EEPROM信息。 
>  
> 设备节点：通过mknod在/dev子目录下建立MTD块设备节点（主设备号为31）和MTD字符设备节点（主设备号为90）。通过访问此设备节点即可访问MTD字符设备和块设备   

因此，可以修改02_network文件，在对应型号的mac生成操作中，对mtd中存储的数据进行检测。  
读内容（mtd中mac）：`hexdump -v -n 6 -s 57344 -e '5/1 "%02x:" 1/1 "%02x"' /dev/mtd2`  

注意：经过测试发现直接dd命令写mtd2，会失败，所以要先将mtd2拷贝成mtd2.bin，写完后用mtd write回写到factory：  
拷贝：`dd if=/dev/mtd2 of=/tmp/mtd2.bin`  
写内容（16进制）：`printf '\x72\xdd\xe1\xcb\xc0\xac' | dd of=/dev/mtd2.bin bs=1 seek=57344 count=6 conv=notrunc `  
回写：`mtd write /dev/mtd2.bin factory`  