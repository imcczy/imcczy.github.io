---
layout: post
title:  "VMware下扩展ubuntu虚拟机根目录分区"
date:   2014-12-07 00:13:40
categories: Linux
---
原文链接：[http://t.cn/RzCP9E2](http://t.cn/RzCP9E2)

>虚拟机用着用着空间就不够了，google了一篇文章，试了一下，有用，征得作者同意，翻译了下。删了一些无关的东西。

注：

>作者输入命令时都用“sudo bash”，用sudo或者su就可以了。

再注：

>**涉及到磁盘分区表删除，最好先备份整个虚拟机！**

#### 检查文件系统： 

    cruz@ubuntu:~$ sudo bash
    [sudo] password for cruz: 
    root@ubuntu:~# df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda1       9.0G  2.7G  5.9G  32% /
    udev            488M  4.0K  488M   1% /dev
    tmpfs           199M  800K  198M   1% /run
    none            5.0M     0  5.0M   0% /run/lock
    none            497M   76K  496M   1% /run/shm
    root@ubuntu:~# 

#### 检查磁盘分区表： 

    root@ubuntu:~# fdisk -l /dev/sda
    Disk /dev/sda: 10.7 GB, 10737418240 bytes
    255 heads, 63 sectors/track, 1305 cylinders, total 20971520 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x00001dec
 
    Device Boot      Start         End      Blocks   Id  System
    /dev/sda1   *        2048    18874367     9436160   83  Linux
    /dev/sda2        18876414    20969471     1046529    5  Extended
    /dev/sda5        18876416    20969471     1046528   82  Linux swap / Solaris
    root@ubuntu:~# 

记住上面显示的交换分区大小（Blocks的数目），这里就是1046528。如果交换分区和根目录不在一个磁盘(比如/dev/sdb），就不要记了。在本文，交换分区在/dev/sda，需要重新设置。

####先关闭linux：

    root@ubuntu:~# shutdown -h now

在虚拟机设置，硬盘，实用工具下选择扩展。重新设置虚拟机的最大磁盘大小。重启。

重新设置分区表要删除所有的旧分区，需要关闭系统的swap:

    cruz@ubuntu:~$ sudo bash
    [sudo] password for cruz: 
    root@ubuntu:~# free -m
                 total       used       free     shared    buffers     cached
    Mem:           992        924         67          0         43        426
     -/+ buffers/cache:        454        537
    Swap:         1021          0       1021
    root@ubuntu:~# swapoff -a
    root@ubuntu:~# free -m
                 total       used       free     shared    buffers     cached
    Mem:           992        924         67          0         43        426
     -/+ buffers/cache:        454        537
    Swap:            0          0          0
    root@ubuntu:~# 

接下来的步骤会删除/dev/sda1和/dev/sda2，**一定要记住**分区表的起始位置，这里是2048！

    root@ubuntu:~# fdisk /dev/sda
 
    Command (m for help): p
 
    Disk /dev/sda: 16.1 GB, 16106127360 bytes
    255 heads, 63 sectors/track, 1958 cylinders, total 31457280 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x00001dec
 
       Device Boot      Start         End      Blocks   Id  System
    /dev/sda1   *        2048    18874367     9436160   83  Linux
    /dev/sda2        18876414    20969471     1046529    5  Extended
    /dev/sda5        18876416    20969471     1046528   82  Linux swap / Solaris
 
    Command (m for help): d
    Partition number (1-5): 1
 
    Command (m for help): d
    Partition number (1-5): 2
 
    Command (m for help): p
 
    Disk /dev/sda: 16.1 GB, 16106127360 bytes
    255 heads, 63 sectors/track, 1958 cylinders, total 31457280 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x00001dec
 
       Device Boot      Start         End      Blocks   Id  System
 
    Command (m for help): 

**不要退出fdisk，创建新分区**

    Command (m for help): n
    Partition type:
       p   primary (0 primary, 0 extended, 4 free)
       e   extended
    Select (default p): p
    Partition number (1-4, default 1): 1
    First sector (2048-31457279, default 2048): 
    Using default value 2048
    Last sector, +sectors or +size{K,M,G} (2048-31457279, default 31457279): 30410751
 
    Command (m for help): n
    Partition type:
       p   primary (1 primary, 0 extended, 3 free)
       e   extended
    Select (default p): p
    Partition number (1-4, default 2): 
    Using default value 2
    First sector (30410752-31457279, default 30410752): 
    Using default value 30410752
    Last sector, +sectors or +size{K,M,G} (30410752-31457279, default 31457279): 
    Using default value 31457279

注意记得创建交换分区，大小别搞错，这里是1046528。修改sda2的分区类型为82，即交换分区。

    Command (m for help): p
 
    Disk /dev/sda: 16.1 GB, 16106127360 bytes
    255 heads, 63 sectors/track, 1958 cylinders, total 31457280 sectors
    Units = sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes
    Disk identifier: 0x00001dec
 
       Device Boot      Start         End      Blocks   Id  System
    /dev/sda1            2048    30410751    15204352   83  Linux
    /dev/sda2        30410752    31457279      523264   83  Linux
 
    Command (m for help): t
    Partition number (1-4): 2
    Hex code (type L to list codes): 82
    Changed system type of partition 2 to 82 (Linux swap / Solaris)
 
    Command (m for help): w
    The partition table has been altered!
 
    Calling ioctl() to re-read partition table.
 
    WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
    The kernel still uses the old table. The new table will be used at
    the next reboot or after you run partprobe(8) or kpartx(8)
    Syncing disks.
    root@ubuntu:~# 

####重启虚拟机：

    root@ubuntu:~# shutdown -r now

交换分区挂载需要UUID标识符。创建新的交换分区不会比配旧的UUID，重启的时候就没有swap可用了。这里主要有两种解决方法：在/etc/fstab里写入新的UUID，或者直接将旧的UUID用在新分区上，这里选择后者。
awk命令用来显示旧的UUID，dd命令确保分区没数据。
    cruz@ubuntu:~$ sudo bash
    [sudo] password for cruz: 
    root@ubuntu:~#  awk '/swap/ { print $1 }' /etc/fstab
    #
    UUID=8bb62351-4436-47df-92fe-af2865f03461
    root@ubuntu:~# swapoff -a
    root@ubuntu:~# free -m
                 total       used       free     shared    buffers     cached
    Mem:           992        695        296          0         23        325
    -/+ buffers/cache:        346        645
    Swap:            0          0          0
    root@ubuntu:~# dd if=/dev/zero of=/dev/sda2
    dd: writing to `/dev/sda2': No space left on device
    1046529+0 records in
    1046528+0 records out
    535822336 bytes (536 MB) copied, 11.9388 s, 44.9 MB/s
    root@ubuntu:~# mkswap -U 8bb62351-4436-47df-92fe-af2865f03461 /dev/sda2
    Setting up swapspace version 1, size = 523260 KiB
    no label, UUID=8bb62351-4436-47df-92fe-af2865f03461
    root@ubuntu:~# swapon -a
    root@ubuntu:~# free -m
                 total       used       free     shared    buffers     cached
    Mem:           992        693        298          0         23        325
    -/+ buffers/cache:        345        646
    Swap:          510          7        503
    root@ubuntu:~#

####最后，调整分区大小：

    root@ubuntu:~# df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda1       9.0G  2.8G  5.8G  33% /
    udev            488M  4.0K  488M   1% /dev
    tmpfs           199M  788K  198M   1% /run
    none            5.0M     0  5.0M   0% /run/lock
    none            497M  200K  496M   1% /run/shm
    root@ubuntu:~# resize2fs /dev/sda1
    resize2fs 1.42 (29-Nov-2011)
    Filesystem at /dev/sda1 is mounted on /; on-line resizing required
    old_desc_blocks = 1, new_desc_blocks = 1
    Performing an on-line resize of /dev/sda1 to 3801088 (4k) blocks.
    The filesystem on /dev/sda1 is now 3801088 blocks long.
 
    root@ubuntu:~# df -h
    Filesystem      Size  Used Avail Use% Mounted on
    /dev/sda1        15G  2.8G   11G  21% /
    udev            488M  4.0K  488M   1% /dev
    tmpfs           199M  788K  198M   1% /run
    none            5.0M     0  5.0M   0% /run/lock
    none            497M  200K  496M   1% /run/shm
    root@ubuntu:~# 