---
layout:     post
title:      "Ubuntu16.04 LVM分区扩容"
subtitle:   "日常linux维护管理"
date:       2017-04-13
author:     "Garry fan"
header-img: "img/post-bg-2015.jpg"
tags:
    - LVM
    - Ubuntu
---

# Ubuntu16.04 LVM分区扩容

>最近ubuntu系统上的根分区磁盘空间不够用了，由于之前安装系统时挂载的是lvm分区，可以很方便的通过lvm在线扩容来解决分区大小不够的问题，当然前提就是要有足够的空闲磁盘或分区来做前提。

### 准备空闲磁盘或分区

>这里使用一块新的空闲硬盘，对应的设备文件是/dev/sdb

```
root@hostname # fdisk /dev/sdb
Welcome to fdisk (util-linux 2.27.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x7533a4ef.

Command (m for help):
```

* 输入n创建一个新的分区
* 输入p创建一个主分区
* 回车选择默认partition number 1
* 回车选择默认first sector
* 回车选择默认last sector
* 输入t修改分区类型
* 输入8e修改默认类型为Linux LVM
* 输入w保存修改

>用ext4文件系统格式化该分区：

```
root@hostname # mkfs.ext4 /dev/sdb1
mke2fs 1.42.13 (17-May-2015)
Creating filesystem with 2621184 4k blocks and 655360 inodes
Filesystem UUID: 9f757396-9534-4e45-9271-6065c600468e
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

```

>这样，我们就把整个新硬盘划分为了一个大分区/dev/sdb1


### root分区LVM扩容

>现在进入正题，开始给lvm分区扩容，首先查看一下当前的分区情况：

```
root@hostname # df -h
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-root   16G   12G  2.9G  80% /
/dev/sda1                    472M  114M  334M  26% /boot
```
>可以看到我们的root分区只剩下2.9G的磁盘空间了

#### 创建PV(Physical volume)
>把我们刚才创建好的空闲分区初始化成PV

```
root@hostname # pvcreate /dev/sdb1
  Physical volume "/dev/sdb1" successfully created
```
>查看一下当前的PG列表，系统里有2个PG，1个是之前安装系统时创建的/dev/sda5，另外一个是我们刚刚创建的/dev/sdb1，其中前者属于ubuntu-vg的VG，后者目前还不属于任何VG

```
root@hostname # pgdisplay
--- Physical volume ---
PV Name               /dev/sda5
VG Name               ubuntu-vg
PV Size               19.52 GiB / not usable 2.00 MiB
Allocatable           yes (but full)
PE Size               4.00 MiB
Total PE              4997
Free PE               0
Allocated PE          4997
PV UUID               P84XnS-RPPi-3onf-tT3B-FO20-ANHW-XlV5MV

"/dev/sdb1" is a new physical volume of "10.00 GiB"
--- NEW Physical volume ---
PV Name               /dev/sdb1
VG Name               
PV Size               10.00 GiB
Allocatable           NO
PE Size               0   
Total PE              0
Free PE               0
Allocated PE          0
PV UUID               Q07O31-68ni-cLT6-ww0D-sjws-pJzf-1R2hLa
```

#### 添加PV到VG(Volume Group)

>查看一下当前的VG列表

```
root@hostname # vgdisplay
--- Volume group ---
VG Name               ubuntu-vg
System ID             
Format                lvm2
Metadata Areas        1
Metadata Sequence No  3
VG Access             read/write
VG Status             resizable
MAX LV                0
Cur LV                2
Open LV               2
Max PV                0
Cur PV                1
Act PV                1
VG Size               19.52 GiB
PE Size               4.00 MiB
Total PE              4997
Alloc PE / Size       4997 / 19.52 GiB
Free  PE / Size       0 / 0   
VG UUID               agGXVo-EAbq-jdr3-S2RI-EmgO-waAn-LRvMKQ
```

>我们现在要做的就是把/dev/sdb1这个PV添加到ubuntu-vg这个VG里

```
root@hostname # vgextend ubuntu-vg /dev/sdb1
  Volume group "ubuntu-vg" successfully extended
```

#### root分区扩容

```
root@hostname # lvextend -L +9G /dev/ubuntu-vg/root
  Size of logical volume ubuntu-vg/root changed from 15.52 GiB (3973 extents) to 24.52 GiB (6277 extents).
  Logical volume root successfully resized.
```

>分区变化同步到文件系统

```
root@hostname # resize2fs /dev/ubuntu-vg/root
resize2fs 1.42.13 (17-May-2015)
Filesystem at /dev/ubuntu-vg/root is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 2
The filesystem on /dev/ubuntu-vg/root is now 6427648 (4k) blocks long.
```

>查看一下现在扩容后的磁盘状态：

```
root@hostname # df -h
Filesystem                   Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-root   25G   12G   12G  51% /
/dev/sda1                    472M  114M  334M  26% /boot
```

>可以看到root分区扩容成功！

