---
layout: post
title:  "第二届MSC安卓第二题"
date:   2016-10-21 20:42:40
categories: Android
tags: CrackMe
---

APK中的Main存在证书校验，修改校验返回恒为1，重打包即可。然后会在Ch中调用native
函数ch，参数即为输入字符串。

so文件有反调试函数，在`.init.array`段的`sub_1424`中：

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-112837.jpg)

反调试的主要逻辑在`sub_1284`，F5反编译如下：

```cpp
char *sub_1284()
{
  ......
  pid = getpid();
  sprintf(&v6, "/proc/%d/status");
  v0 = fopen(&v6, "r");
  _aeabi_memset(&s, 512, 0);
  pthread_mutex_lock((pthread_mutex_t *)&unk_5E40);
  if ( unk_5E48 )
    pthread_cond_signal((pthread_cond_t *)&unk_5E44);
  pthread_mutex_unlock((pthread_mutex_t *)&unk_5E40);
  result = fgets(&s, 512, v0);
  if ( result )
  {
    while ( 1 )
    {
      if ( strstr(&s, "TracerPid") )
      {
        _aeabi_memset(&v4, 128, 0);
        v3 = 0;
        sscanf(&s, "%s %d", &v4, &v3);
        if ( v3 >= 1 )
          break;
      }
      result = fgets(&s, 512, v0);
      if ( !result )
        return result;
    }
    result = (char *)kill(pid, 9);
  }
  return result;
```

由于调用了`pthread_cond_signal`，是不能简单将`sub_1284`函数调用nop掉的，需要将`pthread_cond_wait`的调用也同时nop掉，这个调用是后面动态生成的。这里可以将`kill`调用nop掉。另外`sub_3160`中也存在反调试，主逻辑在`sub_3400`:

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-113845.jpg)

修改直接让`sub_3400`返回0即可。这个`sub_3400`有人说是svc系统调用，不怎么明白 。另外也可以修改手机内核抹去调试信息，这样就不用修改so直接就可以调试了。

进入主函数ch：

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-114438.jpg)

这里`v1`为字符串指针，字符串长度必须小于等于16，首先生成一个16字节的数组，内容是0~15，然后将字符串覆盖到这个数组，例如输入`"abcd"`，则数组为`"a,b,c,d,4,5...,15"`。然后

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-114938.jpg)

这里`mprotect`修改内存状态，但是这里两次调用，一次的port是7，后面一次是5，没找到这俩定义，动态调试看，这里就是运行时修改函数，应该算函数加密。

```cpp
#define PROT_READ 0x1  
#define PROT_WRITE 0x2  
/* WARNING: DO NOT EDIT, AUTO-GENERATED CODE - SEE TOP FOR INSTRUCTIONS */
#define PROT_EXEC 0x4  
#define PROT_SEM 0x8  
#define PROT_NONE 0x0  
#define PROT_GROWSDOWN 0x01000000  
/* WARNING: DO NOT EDIT, AUTO-GENERATED CODE - SEE TOP FOR INSTRUCTIONS */
#define PROT_GROWSUP 0x02000000 
```

动态调试，调用`loc_24c8`处下断点，ida即可反编译函数内容。

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-120406.jpg)

首先，将之前16字节数组，每个元素加上自己的索引数。

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-120335.jpg)

然后动态修改`sub_b4c18de0`即静态分析中的`sub_1de0`的内容。

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-120903.jpg)

然后又是运行时修改`unk_b4c184a4`函数，跟之前一样在调用前下断，

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-121201.jpg)

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-121301.jpg)

传进来的数组，会和一个数组相对应的元素相加，同样，这个数组也是运行时生成的，我们只需要在相加前下断，将这个数组抠出来:

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-122441.jpg)

之后，在`v27`调用处下断

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-122935.jpg)

这个函数就是破解的主要逻辑了，相当复杂。

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-125307.jpg)

这里`unk_b5931750`函数返回一个数组的地址，数组如下：

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-125100.jpg)

熟悉AES的可以看见里面包含了一个s盒，其实密钥是硬编码的，在函数一开始就初始化了，`0x6bcdc67a,0x6b2b7c9d,0x8da459b1,0xab9d0680`，这一部分ida识别貌似有误。这时候我们可以跳出该函数，在校验前下断，验证下处理结果：

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-125857.jpg)

查看检验数组的内存：

![enter image description here](http://imcczy.b0.upaiyun.com/2016-10-31-125940.jpg)

然后用AES算法去解密就可以了。

[AES算法](https://github.com/dhuertas/AES/blob/master/aes.c)