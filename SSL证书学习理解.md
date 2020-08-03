---
title: "SSL证书学习理解"
date: "2017-08-22"
tags: ["SSL证书"]
categories: ["学习笔记", "零散知识"]
---
在服务器搭建配置过程中，我们会使用到SSL证书来让我们的域名通过信任，而SSL证书在不同服务器上有不同的部署形式，他们之间可以互相转化，在这里做一个简单的总结。

# 证书的类型

证书在申请后，下载得到的 www.domain.com.zip 文件，解压获得3个文件夹，分别是Apache、IIS、Nginx 服务器的证书文件。  

- Apache 2.x 证书  
  Apache文件夹内获得证书文件   
  1\_root\_bundle.crt，2\_www.domain.com\_cert.crt 和私钥文件 3_www.domain.com.key,  
  1\_root\_bundle.crt 文件包括一段证书代码 “-----BEGIN CERTIFICATE-----”和“-----END CERTIFICATE-----”,
  2\_www.domain.com\_cert.crt 文件包括一段证书代码 “-----BEGIN CERTIFICATE-----”和“-----END CERTIFICATE-----”,
  3\_www.domain.com.key 文件包括一段私钥代码“-----BEGIN RSA PRIVATE KEY-----”和“-----END RSA PRIVATE KEY-----”

- Nginx 证书  
  Nginx文件夹内获得SSL证书文件  
  1\_www.domain.com\_bundle.crt 和私钥文件 2\_www.domain.com.key,  
  1\_www.domain.com\_bundle.crt 文件包括两段证书代码 “-----BEGIN CERTIFICATE-----”和“-----END CERTIFICATE-----”,  
  2\_www.domain.com.key 文件包括一段私钥代码“-----BEGIN RSA PRIVATE KEY-----”和“-----END RSA PRIVATE KEY-----”。

- IIS 证书  
  IIS文件夹内获得SSL证书文件 www.domain.com.pfx。  
  这是主要是服务器证书类型，如果你想要部署这些证书到你的服务器，可以[查看这里](https://www.qcloud.com/document/product/400/4143)。

# 证书的转换
证书有很多种格式，他们之间可以互相转换

>X.509是常见通用的证书格式。所有的证书都符合为Public Key Infrastructure (PKI) 制定的 ITU-T X509 国际标准。  
>PKCS#7 常用的后缀是： .P7B .P7C .SPC  
>PKCS#12 常用的后缀有： .P12 .PFX  
>X.509 DER 编码(ASCII)的后缀是： .DER .CER .CRT  
>X.509 PAM 编码(Base64)的后缀是： .PEM .CER .CRT  
>.cer/.crt是用于存放证书，它是2进制形式存放的，不含私钥。  
>.pem跟crt/cer的区别是它以Ascii来表示。  
>pfx/p12用于存放个人证书/私钥，他通常包含保护密码，2进制形式  



—————以Apache证书来讲——————————  
获得三个文件 1 domain.com.crt （证书文件） 2 domain.com\_ca.crt  （证书链文件）     3 ssl.key   （KEY）  
将其合并为pfx证书文件：将文件拷贝到装有openssl的Linux机器，在文件夹下执行以下命令   
`openssl pkcs12 -export -out certificate.pfx -inkey ssl.key -in safeapp.com.cn.crt -certfile safeapp.com.cn_ca.crt
`升级过程中会提示输入密码，即以后使用证书时需要输入的密码。    
执行完毕后便会在该目录下生成目标证书。  
——————————————————————————————  
其他可能用到的合并：  
x509（crt&key）到pfx  
`pkcs12 -export -in client1.crt -inkey client1.key -out client1.pfx`  
PKCS#12 到 PEM 的转换   
`openssl pkcs12 -nocerts -nodes -in cert.p12 -out private.pem`  
验证  
` openssl pkcs12 -clcerts -nokeys -in cert.p12 -out cert.pem`  
PEM 到 PKCS#12 的转换   
`openssl pkcs12 -export -in Cert.pem -out Cert.p12 -inkey key.pem`


证书拆解：
pfx 提取出私钥key （pfx的私钥，PRIVATE KEY格式）  
`openssl pkcs12 -in mycert.pfx -nocerts -nodes -out mycert.key`  

-----------------------本例-----------------------------  
从pfx转换为pem，再将pem拆解出crt和key  
`openssl pkcs12 -in mycert.pfx -nodes -out server.pem`  
`openssl rsa -in server.pem -out server.key`  
`openssl x509 -in server.pem -out server.crt`  
(server.pfx是IIS导出的文件，利用openssl导出server.key和server.crt)  
导出后的文件适用于Apache