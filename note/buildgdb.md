---
layout: note
title: MAC下编译gdb&gdbserver for android
permalink: /note/buildgdb
---


一，
这里切换到对应NDK的分支
```
git clone https://android.googlesource.com/toolchain/gdb
git branch -a 
git checkout -b r12 origin/ndk-r12-release
```

二，
修改/gdb-7.7/gdb/remote.c
```cpp
//if(buf_len > 2 * rsa->sizeof_g_packet)
//    error(_("Remote 'g' packet reply is too long: %s"),rs->buf);

if(buf_len > 2 * rsa->sizeof_g_packet)
    rsa->sizeof_g_packet = buf_len;
```

三，
target即为对应于目标系统 ，这里是android，--prefix是安装目录
```bash
./configure --target=arm-linux-androideabi --prefix=<result_absolute_path>
make
make install
```

四，
这里貌似直接把`{ndk}/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/`加到PATH也行。
```
{NDK}/build/tools/make-standalone-toolchain.sh --toolchain=arm-linux-androideabi-4.9 --install-dir=<target_dir> --ndk-dir=<extracted_file_dir>
```
五，
生成makefile，--target是目标程序，--host是gdbserver运行的平台
```
CC=arm-linux-androideabi-gcc ./configure --target=arm-linux-androideabi --host=arm-linux-androideabi --prefix=<result_absolute_path>
```
完了直接make会报错：
类似这样
```
linux-low.c:5227:15: error: 'Elf64_auxv_t' undeclared (first use in this function)
     ? sizeof (Elf64_auxv_t) : sizeof (Elf32_auxv_t);
```
这里还没找到好的解决方法，直接暴力修改gsbserver目录下的config.in文件，注释掉下面两行：
```
/* Define to 1 if the system has the type `Elf32_auxv_t'. */
/*#undef HAVE_ELF32_AUXV_T*/

/* Define to 1 if the system has the type `Elf64_auxv_t'. */
/*#undef HAVE_ELF64_AUXV_T*/
```

在安卓上运行还要修改生成的makefile，cflags加上-fPIE和ldflags加上-fPIE -pie