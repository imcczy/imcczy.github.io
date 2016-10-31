---
layout: post
title:  "编译Nexus5内核"
date:   2016-09-21 19:33:40
categories: Android
tags: Security
---

一些准备工作

```bash
$ sudo apt-get install android-tools-adb android-tools-fastboot
$ sudo apt-get install build-essential abootimg git
```

git仓库在 
`https://android.googlesource.com/kernel/msm`
工具链
`git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/arm/arm-eabi-4.6`

```bash
export PATH=$(pwd)/arm-eabi-4.6/bin:$PATH
export ARCH=arm
export SUBARCH=arm
export CROSS_COMPILE=arm-eabi-
```

官网
`http://source.android.com/source/building-kernels.html`

切换到相对应的分支后编译

```bash
make hammerhead_defconfig
make -j8
```

问题
`Can't use 'defined(@array)' (Maybe you should just omit the defined()?) at kernel/timeconst.pl line 373.`
把373行的defined()去掉，只留@val

打包内核

```bash
abootimg -x boot.img
cp arch/arm/boot/zImage-dtb .
删除bootimg.cfg第一行
abootimg --create myboot.img -f bootimg.cfg -k zImage-dtb -r initrd.img
fastboot boot myboot.img
```

第二种方法使用[bootimg-tools](https://github.com/pbatard/bootimg-tools)

```bash
unmkbootimg -i boot_img/boot.img

kernel written to 'kernel' (8331496 bytes)
ramdisk written to 'ramdisk.cpio.gz' (498796 bytes)

To rebuild this boot image, you can use the command:

重新打包boot.img
cp arch/arm/boot/zImage-dtb kernel

mkbootimg --base 0 --pagesize 2048 --kernel_offset 0x00008000 --ramdisk_offset 0x02900000 --second_offset 0x00f00000 --tags_offset 0x02700000 --cmdline 'console=ttyHSL0,115200,n8 androidboot.hardware=hammerhead  user_debug=31 maxcpus=2 msm_watchdog_v2.enable=1' --kernel kernel --ramdisk ramdisk.cpio.gz -o boot_img/boot.img
重新打包boot.img
```

**修改内核绕过反调试**

```cpp
//base.c 对应改成如下：
else {
           if (strstr(symname, "trace")) {
                return sprintf(buffer, "%s", "sys_epoll_wait");
           }//添加部分
           return sprintf(buffer, "%s", symname);
      }

//array.c 修改如下：
      static const char * const task_state_array[] = {
           "R (running)",        /*    0 */
           "S (sleeping)",       /*    1 */
           "D (disk sleep)",     /*    2 */
           "S (sleeping)",       /*"T (stopped)"   4 */
           "S (sleeping)",       /*"t (tracing stop)"     8 */
           "Z (zombie)",         /*  16 */
           "X (dead)",           /*  32 */
           "x (dead)",           /*  64 */
           "K (wakekill)",       /* 128 */
           "W (waking)",         /* 256 */
      };
      
      "Gid:\t%d\t%d\t%d\t%d\n",
                get_task_state(p),
                task_tgid_nr_ns(p, ns),
                pid_nr_ns(pid, ns),
                ppid, /*tpid*/0,//修改
                cred->uid, cred->euid, cred->suid, cred->fsuid,
                cred->gid, cred->egid, cred->sgid, cred->fsgid);
     //也可以在获取tpid之后跟一句
     //tpid = 0；
```