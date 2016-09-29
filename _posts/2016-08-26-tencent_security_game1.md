---
layout: post
title:  "腾讯游戏安全竞赛Android第一题"
date:   2016-08-26 17:33:40
categories: Android
tags: Security
---


APK放进JEB即可看到软件调用了`libCheckRegister.so`的`NativeCheckRegister`函数，参数是name和code。丢进IDA

```bash
.text:00001758                 EXPORT Java_com_tencent_tencent2016a_MainActivity_NativeCheckRegister
.text:00001758 Java_com_tencent_tencent2016a_MainActivity_NativeCheckRegister
.text:00001758
.text:00001758 var_20          = -0x20
.text:00001758 var_1C          = -0x1C
.text:00001758
.text:00001758                 PUSH    {R0-R2,R4-R7,LR}
.text:0000175A                 LDR     R1, [R0]
.text:0000175C                 MOVS    R7, R2
.text:0000175E                 MOVS    R2, #0x2A4
.text:00001762                 MOVS    R5, R3
.text:00001764                 LDR     R3, [R1,R2]；R2偏移R3，是NewStringUTF，可以查看JNI API(Android软件安全与逆向分析7.6节也有介绍),如下图所示，所有的函数在附件中。

.text:00001766                 MOVS    R1, R7
.text:00001768                 MOVS    R2, #0
.text:0000176A                 MOVS    R4, R0
.text:0000176C                 BLX     R3；调用NewStringUTF函数，第一个参数R0，是JNIEnv,子程序返回,第二个参数是R1，这里R1<-R7<-R2，即name
.text:0000176E                 LDR     R1, [R4]
.text:00001770                 MOVS    R2, #0x2A4
.text:00001774                 MOVS    R6, R0
.text:00001776                 LDR     R3, [R1,R2]
.text:00001778                 MOVS    R0, R4
.text:0000177A                 MOVS    R1, R5
.text:0000177C                 MOVS    R2, #0
.text:0000177E                 BLX     R3
.text:00001780                 STR     R0, [SP,#0x20+var_20]
.text:00001782                 LDR     R1, [SP,#0x20+var_20]
.text:00001784                 MOVS    R0, R6
.text:00001786                 BL      sub_1634
... ...
.text:000017AE ; End of function Java_com_tencent_tencent2016a_MainActivity_NativeCheckRegister
```

jni函数默认带有两个参数`JNIEnv*`和`jobject`，对应R0和R1。`.text:00001764  `中偏移对应的函数，有人整理了一个[表](https://github.com/zhengmin1989/TheSevenWeapons/blob/master/KongQueLing/JNI_ENV_FUNCTIONS.xlsx)，也可以F5在IDA的反汇编页面中，选中相应的指针按Y键，将指针类型改为`JNIEnv* ` ，这样IDA即可显示jni函数，这里对应的是`GetStringUTFChars`，将java中的string对象转换成`char*`。主函数中两次转换后，即跳入`sub_1634`，是验证的主要逻辑。

![Alt text](http://imcczy.b0.upaiyun.com/2016-09-29-14%3A07%3A49.jpg)

这里R0是name，R1是code，这里主要计算了name的长度，且`len(name)-6<=14`，即name的长度在6～20之间。

![Alt text](http://imcczy.b0.upaiyun.com/2016-09-29-14%3A09%3A00.jpg)

`loc_165E`首先分配冷20个字节的内存空间，且全初始化为0。`loc_1672`主要用于循环处理name，生成一个新的20字节字符串。`__aeabi_idivmod`是Run－time ABI函数，即求余，返回R0为商，R1为余。主要逻辑是：

```
char *s[20],*name[x];
for(i=0 ; i++ ; i<16){
    (word)s[i] = (Byte)name[i%len(name)] * (i + 0x1339E7E) * len(name) ＋ s[i]的高三个字节
｝
```

即每次取name一字节数据，作相应变换后，写入s[i]一个字，也就是4字节，这样上次写入的高三个字节都会被新的数据覆盖，16次循环即会产生19字节数据＋1字节的0。

![Alt text](http://imcczy.b0.upaiyun.com/2016-09-29-14%3A09%3A18.jpg)

这里有两个处理函数`sub_146C`和`sub_1498`，这里应该是逻辑与，两个函数返回结果都满足才会进入下一步。第二个函数包含冷第一个函数，这里只分析第二个。

![Alt text](http://imcczy.b0.upaiyun.com/2016-09-29-14%3A09%3A40.jpg)

由上一步可以R0是分配的内存空间，R1指向code。首先会以code的每个字节为键值查表，表值不能大于0x3f，否则code会被截断。`.text:000014AE  `处，R2为计算得到符合的code长度。后面的逻辑是：

```
((len＋2)/4)*3；结果保存到了R5
```

然后就是if判断分支了，这里判断的是R2，即符合的code长度。

![Alt text](http://imcczy.b0.upaiyun.com/2016-09-29-14%3A15%3A00.jpg)

主体是一个循环，用于处理code长度大于4的情况。就是读取code四个字节，进过转换变成3个字节存入新的字符串数组。当code的长度大于4:

```
len-4
*(Byte)c = T[*(Byte)code] * 4 | (T[*(Byte)(code+1)] >> 4)
*(Byte)(c+1) = T[*(Byte)(code+1)] * 16 | (T[*(Byte)(code+2)] >> 2)
*(Byte)(c+2) = T[*(Byte)(code+2)] * 64 | T[*(Byte)(code+3)]
code = code + 4
c = c + 3
```

当code长度等于2时：

![Alt text](http://imcczy.b0.upaiyun.com/2016-09-29-14%3A11%3A22.jpg)

这里

```cpp
*(Byte)c = T[*(Byte)code] * 4 | (T[*(Byte)(code+1)] >> 4)
```

当code长度等于3时：

```cpp
*(Byte)c = T[*(Byte)code] * 4 | (T[*(Byte)(code+1)] >> 4)
*(Byte)(c+1) = T[*(Byte)(code+1)] * 16 | (T[*(Byte)(code+2)] >> 2)
```
最后，

```bash
.text:00001548 loc_1548                                ; CODE XREF: sub_1498+68j
.text:00001548                                         ; sub_1498+84j ...
.text:00001548                 MOVS    R1, #0
.text:0000154A                 STRB    R1, [R3]
.text:0000154C                 NEGS    R2, R2
.text:0000154E                 MOVS    R3, #3
.text:00001550                 ANDS    R2, R3
.text:00001552                 SUBS    R0, R5, R2
.text:00001554                 POP     {R4-R7,PC}
```

R5是`((len＋2)/4)*3`，R2是剩余的长度即`len％4`，`R2=-R2 & 3｀即，R0:

```
((len＋2)/4)*3－(-R2 & 3)
```

这里的R0必须等于20，由上可知，4字节数据生成3字节，3字节生成2字节，2字节生成1字节数据。
20 = 3x6+2
所以code的长度`6x4+3＝27`。

到这为止就是利用name和code生成了两个长度20的数组s和c。

![Alt text](http://imcczy.b0.upaiyun.com/2016-09-29-14%3A11%3A42.jpg)

`ADD R3, SP, #0x470+s`，`R3`重新指向根据name生成的字符串，`R4=0`。这里的循环每次取s一个字的数据，然后除以10，新的数据存入新的数组。同时将c的20字节数据复制到一个新的数组。至此所有数据处理完毕。
 然后是几个方程组：

![Alt text](http://imcczy.b0.upaiyun.com/2016-09-29-14%3A12%3A01.jpg)

由于这里的操作都以字为单位，可以当成int数组来处理。我们假设name产生的数组为n，code产生的数组为c。R5对应c，R6对应n。
所以

```cpp
c[4]+n[0]=c[2]
c[4]+n[0]+n[1] = 2*c[4]
c[3]+n[2] = c[0]
c[3]+n[2]+n[3] = 2*c[3]
n[4]+c[1]=3*n[2]
```

code的处理类似base64解码。因为每个键值都是6位，编码的过程类似：

```cpp
c[0] = 6+2
c[1] = 4+4
c[2] = 2+6
```

每3个字节对应4个6比特的键值，每个键值对应一个键。程序里用的就是base64的表。keygen就是将3个字节分解成4个6bit的键值，根据键值去查找对应的键，然后组成code