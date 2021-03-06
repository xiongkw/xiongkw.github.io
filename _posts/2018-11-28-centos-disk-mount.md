---
layout: post
title: CentOS下磁盘分区和挂载
categories: [编程, linux]
tags: [centos, fdisk]
---

> 部署服务，运维直接扔了一台物理机，需要自己挂载硬盘

#### 1. 查看硬盘

使用`fdisk`命令查看硬盘及其分区列表

```
$ fdisk -l

Disk /dev/sdb: 239.9 GB, 239902654464 bytes, 468559872 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

```

> `Linux`下`SCSI`和`SATA`设备的命名格式为`/dev/sd[a-z]`，例如第一块硬盘为`/dev/sda`，第二块为`/dev/sdb`...

#### 2. 分区

使用`fdisk`命令为硬盘分区

```
$ fdisk /dev/sdb

Welcome to fdisk (util-linux 2.23.2).

Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table
Building a new DOS disklabel with disk identifier 0x7d807d4b.

Command (m for help): n                                      // 添加分区
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p):                                         // 主分区
Using default response p
Partition number (1-4, default 1):                          // 分区号
First sector (2048-468559871, default 2048):
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-468559871, default 468559871):
Using default value 468559871
Partition 1 of type Linux and of size 223.4 GiB is set

Command (m for help): w                                     // 保存分区信息
```

#### 3. 查看分区

使用`fdisk`命令查看指定硬盘和其分区信息

```
$ fdisk -l /dev/sdb

Disk /dev/sdb: 239.9 GB, 239902654464 bytes, 468559872 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x38ba6c38

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048   468559871   234278912   83  Linux
```

#### 4. 格式化

分区要格式化后才能使用

```
$ mkfs.xfs /dev/sdb1
```

> 这里使用`xfs`文件系统

#### 5. 挂载

```
$ mkdir /app

$ mount /dev/sdb1 /app

$ df -h /app
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1                              /app
```

> 把分区挂载到指定目录，这样才能访问

#### 6. 设置开机自动挂载

```
$ echo "/dev/sdb1    /app    xfs    defaults    0    0" >> /etc/fstab
```

#### 7. 附.也可不分区直接格式化

##### 7.1 lsblk查看可用块设备

```
$ lsblk
NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda       8:192  0 446.1G  0 disk 
├─sda1    8:193  0     1G  0 part /boot
└─sda2    8:194  0 445.1G  0 part /
sdb       8:80   0   5.5T  0 disk 
sdc       8:48   0   5.5T  0 disk
```

> 可以看到`sdb/sdc`都没有挂载

##### 7.2 不分区直接格式化整个硬盘

```
$ mkfs.xfs -f -i attr=2 -l lazy-count=1,sectsize=4096 -b size=4096 -d sectsize=4096 -n ftype=1 /dev/sdb
```

##### 7.3 挂载

```
$ mkdir /data
$ mount /dev/sdb /data
$ echo "/dev/sdb    /data    xfs    defaults    0    0" >> /etc/fstab
```