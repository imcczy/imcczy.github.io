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
编译

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

第二种方法

```bash
unmkbootimg -i boot_img/boot.img

kernel written to 'kernel' (8331496 bytes)
ramdisk written to 'ramdisk.cpio.gz' (498796 bytes)

To rebuild this boot image, you can use the command:
mkbootimg --base 0 --pagesize 2048 --kernel_offset 0x00008000 --ramdisk_offset 0x02900000 --second_offset 0x00f00000 --tags_offset 0x02700000 --cmdline 'console=ttyHSL0,115200,n8 androidboot.hardware=hammerhead  user_debug=31 maxcpus=2 msm_watchdog_v2.enable=1' --kernel kernel --ramdisk ramdisk.cpio.gz -o boot_img/boot.img
重新打包boot.img
```