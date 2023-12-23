---
title: "lvm逻辑卷"
date: 2021-11-05T16:24:18+08:00
tags: ["存储"]
description: lvm逻辑卷
---

## why lvm

lvm全称为logical volume manager,即逻辑卷管理。

因为硬件层实现的RAID阵列扩展起来不够灵活，所以故引入了一个中间层，在软件层面对磁盘资源进行再一次的组织管理。

纸上得来终觉浅，马上来看看怎么玩。

对了，在此之前，再贴一下之前写过的和lvm相关的一些概念

```
- PV（physical volume）:OS识别到的物理磁盘
- VG（volume group）:多个PV的集合，卷组
- PP（physical partition）:将VG分割成逻辑上连续的小块，但物理上不一定连续
- LP：逻辑区块。多个PP的集合
- LV：多个LP组成的逻辑卷
```

开始吧

第一件事情，先创建几个磁盘分区，用于制作PV。重复三次

```shell
[root@jayice ~]# fdisk /dev/sdb
Command (m for help): n
Command action
   e   extended
   p   primary partition (1-4)
p
Partition number (1-4): 1
First cylinder (1-1044, default 1):1
Last cylinder, +cylinders or +size{K,M,G} (1-1044, default 1044): +1G

Command (m for help): t
Selected partition 1
Hex code (type L to list codes): 8e
Changed system type of partition 1 to 8e (Linux LVM)
```

看看成功没：

```shell
[root@jayice ~]# fdisk -l
Device Boot      Start         End      Blocks   Id  System
/dev/sdb1               1         132     1060258+  8e  Linux LVM
/dev/sdb2             133         264     1060290   8e  Linux LVM
/dev/sdb3             265         396     1060290   8e  Linux LVM
```

nice



接下来就可以基于这三个分区制作PV了，非常简单

```shell
[root@jayice ~]# pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created.
[root@jayice ~]# pvcreate /dev/sdb2
  Physical volume "/dev/sdb2" successfully created.
[root@jayice ~]# pvcreate /dev/sdb3
  Physical volume "/dev/sdb3" successfully created.
```



然后用这三个PV制作成一个VG

```shell
[root@jayice ~]# vgcreate volume-group /dev/sdb1 /dev/sdb2 /dev/sdb3
--- Volume group ---
VG Name               volume-group
System ID
Format                lvm2
Metadata Areas        3
Metadata Sequence No  1
VG Access             read/write
VG Status             resizable
MAX LV                0
Cur LV                0
Open LV               0
Max PV                0
Cur PV                3
Act PV                3
VG Size               3.02 GiB
PE Size               4.00 MiB
Total PE              774
Alloc PE / Size       0 / 0
Free  PE / Size       774 / 3.02 GiB
VG UUID               wqd7pE-ktAz-lEJD-qu7c-TK8s-dJ7s-hU8Jk0
```

还不能在VG上直接初始化文件系统，还得在VG之上创建逻辑卷LV

```shell
[root@jayice ~]# lvcreate -L 100M -n lv-1 volume-group ## 创建一个100m大小，名为lv-1的逻辑卷
```

然后在这个逻辑卷上格式化一个ext4文件系统，然后挂载到/lvm-test中，之后就可以在上面读写数据了

```shell
[root@jayice ~]# mkfs.ext4 /dev/volume-group/lv-1
[root@jayice ~]# mount /dev/volume-group/lv-1 /lvm-test/ 
```

前面说了，lvm的出现很大部分原因就是扩展性，然后来给这个lv扩下容

```shell
[root@jayice ~]# umount /lvm-test   ## 卸载
[root@jayice ~]# lvresize -L 500M /dev/volume-group/lv-1  ## 修改设置,扩容到500M
[root@jayice ~]# resize2fs /dev/volume-group/lv-1  ## 执行修改
```

然后看下

```shell
[root@jayice ~]# lvdisplay
---Logical volume ---
LV Name/dev/volume-group/lv-1
VG Name volume-group
LV UUID 5eEdFY-8Eei-DH6s-n&j8-Y8jY-s7ao-7giGiw
LV WriteAccess read/write
LV Status available
# open 0
LV Size 500.00MiB
Current LE 50
Segments1
Allocation inherit
Read ahead sectors auto
- currently set to 256
Block device 253:2
```

nice！



简单玩了下而已......