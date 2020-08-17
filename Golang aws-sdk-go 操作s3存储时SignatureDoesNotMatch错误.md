---
title: "Golang aws-sdk-go 操作s3存储时 SignatureDoesNotMatch 错误"
date: "2020-05-27"
categories: ["学习笔记"]
---

相同的问题我们可以在 [Github Issue](https://github.com/aws/aws-sdk-go/issues/562) 上找到：

bucket 本身没有 /slash，只是 object 的 prefix 模拟实现了文件夹

因此加入 / 会导致请求编码问题，S3 侧无法对请求体正确解析导致错误。

同时要保证如果要进行 copyObject 类似的含有 copySource 字段的操作，该进行 URLEncode 的字段必须要 Encode。