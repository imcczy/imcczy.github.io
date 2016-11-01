---
layout: post
title:  "腾讯游戏安全竞赛Android第二题"
date:   2016-09-05 17:33:40
categories: Android
tags: CrackMe
---

将APK丢进JEB，可以看到扫描的得到的数据首先会base64解码一遍，得到的字符串长度必须为32，即注册码的长度为44。
解码得到的32位byte[]会利用公钥解密一遍，解出来的数据前两个字节是长度，`int v1 = v0_1[0] * 256 + v0_1[1];`，必须是20个字节。即解密出来的数据总长度至少是22字节。然后就是将这20个字节送入native函数了。第一个参数是device ID，第二个参数即解密出来的20字节byte[]的code。
对deviceid和code转换成c数据后，直接跳sub_1660。R0是deviceid，R1是code,计算了deviceid的长度，且长度范围是6~100进入计算deviceid的sha1。**判断点是sha1的5个常量**

{% include image.html src="http://imcczy.b0.upaiyun.com/2016-09-05-11%3A38%3A18.jpg" %}

**注意，这里实际计算sha1的多了一个字节**

{% include image.html src="http://imcczy.b0.upaiyun.com/2016-09-05-15%3A27%3A01.jpg" %}

```c 
void SHA1Init(SHA1_CTX *context);
void SHA1Update(SHA1_CTX *context, unsigned char *input, unsigned int inlen);
void SHA1Final(unsigned char *output, SHA1_CTX *context);
```

The SHA1Init() function initializes the SHA1 context structure pointed to by context.
SHA1Init的五个常量是：

```
0x67452301
0xEFCDAB89
0x98BADCFE
0x10325476
0xC3D2E1F0
```

>The SHA1Update() function computes a partial SHA1 digest on the inlen-byte message block pointed to by input, and updates the SHA1 context structure pointed to by context accordingly.

>The SHA1Final() function generates the final SHA1 digest, using the SHA1 context structure pointed to by context. The 16-bit SHA1 digest is written to output. After a call to SHA1Final(), the state of the context structure is undefined. It must be reinitialized with SHA1Init() before it can be used again.

计算完后得到5个字160bit的数据，然后就是将这个5个字的前四个字和后4个字作为参数，进入sub_1330，查看后，发现该函数调用了DES的S盒，即该函数是DES函数。

**AES加解密详解**

{% include image.html src="http://imcczy.b0.upaiyun.com/2016-09-05-15%3A27%3A33.jpg" %}

不管是加密还是解密都需要密钥扩展，即都要用到s盒。密钥bit的总数＝分组长度×（轮数Round＋1）例如当分组长度为128bits和轮数Round为10时，轮密钥长度为128×(10＋1)＝1408bits。

根据逆s盒可以判定这里是aes解密，sha1的前四个字是密闻，后四个字是key。同时产生4个字的新数据。`AES/ECB/NoPadding`这里还要看看。
接下来的函数还是以deviceid为参数，判定点是crc32的table：

{% include image.html src="http://imcczy.b0.upaiyun.com/2016-09-05-15%3A27%3A55.jpg" %}

CRC32产生四个字节的数据，复制到sha1长生的16字节数据后面。

{% include image.html src="http://imcczy.b0.upaiyun.com/2016-09-05-15%3A28%3A10.jpg" %}

这里会将deskey的后四个字节用RSA解密出来的最后四个字节覆盖。DesKey 是8个字节，前四个是 固定的3F 5D 23 13 后四个字节是 password_copy的最后四个字节（上下文判断就是CRC32）

后面就是两轮des解密，每一轮解密覆盖8个字节，最后四个字节不变。然后和之前由deviceID生成的20字节比较。
总结下：
deviceID经sha1生成20个字节
前16个字节为密文，后16个字节为key，AES解密生成16个字节
对deviceID作CRC32校验生成4字节，跟在AES解密生成的16字节后合并成20字节。

扫描二维码得到44字节
base64解码，得到32字节
然后用RSA公钥解密这32字节，解出来的数据前两个字节是长度，`int v1 = v0_1[0] * 256 + v0_1[1];`，必须是20个字节。即`0x00`和`0x14`，解出来长度至少是22字节。
20个字节作为参数，两轮des解密，每一轮解密覆盖8个字节，最后四个字节不变。
这里des解密具体用的什么方式需要再看，简单一点可以debug抠出解密的数据来验真。

公钥是:
`MDwwDQYJKoZIhvcNAQEBBQADKwAwKAIhAMw8CJ6Azv7ak+y+AEJmen4UMMPkGQ5D2QBrG7vKcX6XAgMBAAE=`