---
layout: post
title:  "ARM动态链接理解"
date:   2016-08-6 20:42:40
categories: Android
tags: Security
---

实验环境为`Linux raspberrypi 4.4.11-v7+ #888 SMP Mon May 23 20:10:33 BST 2016 armv7l GNU/Linux`

测试代码：

```cpp
#include<stdio.h>
void main(){
    printf("first printf");
    printf("second printf");
}
```

首先查看main函数的汇编代码：

```asm
0001041c <main>:
   1041c:	e92d4800 	push	{fp, lr}
   10420:	e28db004 	add	fp, sp, #4
   10424:	e59f000c 	ldr	r0, [pc, #12]	; 10438 <main+0x1c>
   10428:	ebffffa5 	bl	102c4 <puts@plt>
   1042c:	e59f0008 	ldr	r0, [pc, #8]	; 1043c <main+0x20>
   10430:	ebffffa3 	bl	102c4 <puts@plt>
   10434:	e8bd8800 	pop	{fp, pc}
```

可见printf的调用地址是`0x102c4`，查看该处内容：

```armasm
//objdump
000102c4 <puts@plt>:
   102c4:	e28fc600 	add	ip, pc, #0, 12
   102c8:	e28cca10 	add	ip, ip, #16, 20	; 0x10000
   102cc:	e5bcf314 	ldr	pc, [ip, #788]!	; 0x314

//gdb disas
gef> disass 0x102c4,0x102d0
Dump of assembler code from 0x102c4 to 0x102d0:
   0x000102c4:	add	r12, pc, #0, 12
   0x000102c8:	add	r12, r12, #16, 20	; 0x10000
   0x000102cc:	ldr	pc, [r12, #788]!	; 0x314

```

即`plt`中的`puts`条目，计算可得，`0x102cc`处指令的pc值为`0x205e0=0x102cc+0x10000+0x314`。查看其内容：

```armasm
gef> xinfo $r12+0x314
Found 0x000205e0
Page: 0x00020000 -> 0x00021000 (size=0x1000)
Permissions: rw-
Pathname: /home/pi/test
Offset (from page): +0x5e0
Inode: 99388
Segment: .got (0x000205d4-0x000205f4)

gef> p/x *0x205e0
$1 = 0x102b0

//objdump
Disassembly of section .got:
000205d4 <_GLOBAL_OFFSET_TABLE_>:
   205d4:	000204ec 	andeq	r0, r2, ip, ror #9
	...
   205e0:	000102b0 			; <UNDEFINED> instruction: 0x000102b0
   205e4:	000102b0 			; <UNDEFINED> instruction: 0x000102b0
   205e8:	000102b0 			; <UNDEFINED> instruction: 0x000102b0
   205ec:	000102b0 			; <UNDEFINED> instruction: 0x000102b0
   205f0:	00000000 	andeq	r0, r0, r0
```
即跳转到了GOT中的puts条目，其内容是`0x102b0`，所以程序会接着跳转到这：
```armasm
000102b0 <puts@plt-0x14>:
   102b0:	e52de004 	push	{lr}		; (str lr, [sp, #-4]!)
   102b4:	e59fe004 	ldr	lr, [pc, #4]	; 102c0 <_init+0x1c>
   102b8:	e08fe00e 	add	lr, pc, lr
   102bc:	e5bef008 	ldr	pc, [lr, #8]!
   102c0:	00010314 	andeq	r0, r1, r4, lsl r3
```

这里会继续跳向`0x205dc`，也是一个GOT条目，储存着，动态链接器的入口地址：

```armasm
gef> p/x *0x205dc
$2 = 0x76fe4f38
gef> disass 0x76fe4f38
Dump of assembler code for function _dl_runtime_resolve:
   0x76fe4f38 <+0>:	push	{r0, r1, r2, r3, r4}
   0x76fe4f3c <+4>:	ldr	r0, [lr, #-4]
   0x76fe4f40 <+8>:	sub	r1, r12, lr
   0x76fe4f44 <+12>:	sub	r1, r1, #4
   0x76fe4f48 <+16>:	add	r1, r1, r1
   0x76fe4f4c <+20>:	bl	0x76fde2e8 <_dl_fixup>
   0x76fe4f50 <+24>:	mov	r12, r0
   0x76fe4f54 <+28>:	pop	{r0, r1, r2, r3, r4, lr}
   0x76fe4f58 <+32>:	bx	r12
End of assembler dump.
```

解析完后，会将符号的地址写入GOT，当我们第二次调用该函数时，GOT条目的值是：

```armasm
gef> p/x *0x205e0
$3 = 0x76ec4478
gef> disas 0x76ec4478
Dump of assembler code for function _IO_puts:
   0x76ec4478 <+0>:	push	{r4, r5, r6, r7, lr}
   0x76ec447c <+4>:	sub	sp, sp, #12
   0x76ec4480 <+8>:	mov	r6, r0
   0x76ec4484 <+12>:	bl	0x76edb630 <strlen>
   ...
```