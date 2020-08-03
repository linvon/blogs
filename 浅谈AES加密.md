---
title: "\u6D45\u8C08AES\u52A0\u5BC6"
date: "2019-08-18"
toc: false
tags: ~
categories:
- "\u5B66\u4E60\u7B14\u8BB0"
...
--- 最近在项目上用到了AES加密，虽然是非常简单的加密算法了，但是密码学零基础的话还是要花一点时间理解用法的。主要说一下使用方法，具体的实现就不深究了。



> AES加密的实现是将数据看做一个块来实现的，每次对一个块（也就是矩阵）大小的数据进行加密操作，矩阵中的每一个字节都与该回合的加密块（round key）做XOR运算；每个子密钥由密钥生成方案产生。之后回合密钥将会与原矩阵合并。在每次的加密循环中，都会由主密钥产生一把回合密钥（通过Rijndael密钥生成方案产生），这把密钥大小会跟原矩阵一样，以与原矩阵中每个对应的字节作异或加法。



由上述可知，AES每次处理的数据是一块（16字节），当数据不是16字节的倍数时就需要填充，这就是所谓的分组密码（区别于基于比特位的流密码），16字节是分组长度。

AES有几种不同的加密模式，不同的模式需要的参数也是不同的：

- ECB(Electronic Code Book电子密码本)：是一种基础的加密方式，密文被分割成分组长度相等的块（不足补齐），然后单独一个个加密，一个个输出组成密文。

- CBC(Cipher Block Chaining，加密块链)：是一种循环模式，前一个分组的密文和当前分组的明文异或或操作后再加密，是SSL、IPSec的标准。但这样不利于并行计算，且需要初始化向量IV
- CFB(Cipher FeedBack Mode，加密反馈)/OFB(Output FeedBack，输出反馈)：隐藏了明文的模式，分组密码转化为流模式，可以及时加密传送小于分组的数据



几种模式中最常用的便是CBC，是比较可靠的一种方式，我们主要就介绍这一种方式。

代码我们以C语言的OpenSSL库和golang的标准库作为范例。

首先看一下参数

``` c
#include <openssl/aes.h>
int AES_set_encrypt_key(const unsigned char *userKey, const int bits, AES_KEY *key);
int AES_set_decrypt_key(const unsigned char *userKey, const int bits, AES_KEY *key);
//以上两个函数是用来根据用户指定的密码来生成加密所需要的key，也就是加密块
```

| 参数    | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| userKey | 用户指定的密码。必须是16、24、32字节，对应AES-128、AES-256等加密方式 |
| bits    | 密码位数。即userKey的长度 * 8，只能是128、192、256位。       |
| key     | 输出的AES_KEY，即加密块                                      |

``` c
void AES_cbc_encrypt(const unsigned char *in, unsigned char *out,
                     size_t length, const AES_KEY *key,
                     unsigned char *ivec, const int enc);
```

| 参数   | 说明                                                         |
| ------ | ------------------------------------------------------------ |
| in     | 输入数据。长度任意。                                         |
| out    | 输出数据。能够容纳下输入数据，且长度必须是16字节（AES加密块长度）的倍数。 |
| length | 输入数据的实际长度。                                         |
| key    | 使用AES_set_encrypt/decrypt_key生成的Key。                   |
| ivec   | 可读写的一块内存。长度必须是16字节。                         |
| enc    | 是否是加密操作。AES_ENCRYPT表示加密，AES_DECRYPT表示解密。   |

实际上输入数据在长度上AES是有要求的，只不过在长度不满足是加密块长度的倍数时，它会自动帮你做填充。输出的数据缓冲区长度就必须满足是加密块长度的倍数。

其中的ivec是AES加密过程中使用的向量，其主要意义是混淆加密，可以让同样的明文产生出不同的加密密文。该向量必须可写、独立，因为他在加密的过程中是会根据当前一轮的加密结果进行变化的。此外加解密需要使用相同的向量，因此将向量放在密文中也是一种加密选择的方式。下面的go代码中我们能看到。

``` go
// 这是golang官网文档提供的AES加密实例
key := []byte("example key 1234")
plaintext := []byte("exampleplaintext")
// 此前提到的，明文的长度必须满足是加密块长度的倍数
if len(plaintext)%aes.BlockSize != 0 {
    panic("plaintext is not a multiple of the block size")
}
// 此处根据用户指定的key来生成加密块
block, err := aes.NewCipher(key)
if err != nil {
    panic(err)
}
// IV向量是用来保证加密的随机性的，而不是保证加密的安全性的。
// 我们可以看到，对于密文其申请了明文长度+AES块（即IV长度）的大小，将IV放在了密文的最前一块
ciphertext := make([]byte, aes.BlockSize+len(plaintext))
iv := ciphertext[:aes.BlockSize]
if _, err := io.ReadFull(rand.Reader, iv); err != nil {
    panic(err)
}
mode := cipher.NewCBCEncrypter(block, iv)
mode.CryptBlocks(ciphertext[aes.BlockSize:], plaintext)

fmt.Printf("%x\n", ciphertext)
```



