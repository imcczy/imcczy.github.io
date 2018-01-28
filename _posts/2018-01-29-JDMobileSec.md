---
layout: post
title:  "某东微联安全防护分析"
date:   2018-01-29 01:26:30
categories: Android
tags: App
---


 前一阵实验室师妹做IoT相关研究的时候发现某东微联的App无法使用Xposed，具体表现为Xposed没有任何报错但是hook不成功，今天抽空分析了下其安全防护模块的实现。

官网上的最新App版本为v4.5.1，拖进jeb发现存在Application类，在其静态代码块中加载了Native库`JDMobileSec`：
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-161035.png)
同时App代码中的字符串被加密：
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-161104.png)
且解密算法在Native层中实现：
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-161142.png)
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-161229.png)
IDA打开看看，init段存在一堆函数：
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-161539.png)
`Jni_OnLoad`很不友好的样子：
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-161839.png)
跟一般的ollvm控制流混淆还不太一样，这个函数是递归调用，看着脑阔疼。退而求其次去看下历史版本，发现`JDMobileSec`最早出现在4.4.0版本，这个版本的`Jni_OnLoad`就好看多了：
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-163626.png)
动态调试一哈
`init_arrary`段的函数，都是初始化一些字符串和线程相关的变量，字符串貌似跟Native函数动态注册相关：
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-163823.png)
`init_arrary`段没有做反调试，直接进入`Jni_OnLoad`，发现`Jni_OnLoad`本身没有涉及核心逻辑，相关代码都是以独立函数实现的。通过动态调试，观察到函数的执行顺序是：
> 1. sub_465C
> 2. sub_DDA0
> 3. sub_C4D4
> 4. sub_BFF0
> 5. sub_EA18
> 6. sub_440C
> 7. sub_F718
> 8. sub_440C

### sub_465C
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-165139.png)
简单的检测运行时间反调试，但`gettimeofday`的第一个参数单位是秒，这粒度未免也太大了，目测`F8`按快一点就测不出来了。

### sub_DDA0
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-165729.png)

`sub_DDA0`主要实现反调试，反模拟器等功能。`sub_CBD0`是模拟器检测：
![enter image description here](http://imcczy.b0.upaiyun.com/2018-01-28-170007.png)
`sub_B908`是字符串解密，目测每一个字符对应一个4字节的int，且一一对应，这一部分应该是可以通过脚本恢复出来的。`sub_CABE`用于判断文件是否存在，一旦存在就退出程序。`sub_D1A4`和`sub_D01C`是常见的调试端口检测以及`Tracepid`字段的检测。`sub_D300`功能未知，反正是开一个线程用于反调试，直接nop掉了。

### sub_C4D4
`sub_C4D4`函数是反Xposed的核心逻辑，本来以为会是什么黑科技，看下来发现是ç存在一个名为`disablehooks`的静态字段，功能跟其字面意思一样。通过反射把值设为`false`，就可以禁用对本App的Xposed Hook功能了。需要注意的是`XposedBridge`类的`classloader`是要通过`getSystemClassLoader`获取的，很神奇，这块需要再深入看看。Native调用Java的代码真的是又臭又长，就不贴了。

### sub_440C
`sub_440C`的主要功能是签名校验，不知道为啥在`Jni_OnLoad`里调用了两次。没仔细看。

###其他
剩余3个函数是Jni函数的动态注册和一些变量的初始化，无关紧要了。

### End
`sub_C4D4`中查找`XposedBridge`类的方法是`loadClass`，所以应对方法也很简单，用Xposed把`loadClass`hook了就行，加载`XposedBridge`类的时候直接返回`null`。