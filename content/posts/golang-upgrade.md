---
title: Golang 升级导致SSL连接失败
categories:
    - 技术
date: 2024-06-05
tags:
    - golang
    - go1.22
---

## 背景

最近，有个线上功能客户反馈上传图片失败。这个图片是保存在内部的一个对象存储服务中。

## 第一次尝试

使用之前编译好的[S3命令行工具](https://github.com/lpflpf/s3-command-line)做了验证，发现时好时坏。大概率是S3服务有某个接入节点有问题，这个S3服务经常出问题，应该是有个节点故障了。  
快速联系相关运维同学，lvs摘除了节点。用S3命令行工具测试，没有问题了。但再次测试还是有问题，这次的问题变成了 ` remote error: tls: handshake failure`

## tls 失败

s3 服务是基于Http协议来交互的。公司的http 使用了https 支持，毕竟是安全公司。为了快速回复，上传服务改成了 http, 问题得到了暂时解决。

## 那为什么tls会失败呢

难道go的tls依赖了系统的openssl库？用CGO_ENABLED重新验证，问题还是稳定复现。看来使用的是 ‘crypto/tls’这个包，而非系统库。

看最近的代码改动，有个go版本的升级，从 1.21 -> 1.22.2； 搜Issue，果然不出所料 [crypto/tls: https request, tls handshake failure in go1.22]( https://github.com/golang/go/issues/66512)

## 如何复现

服务端配置：
```
openssl ciphers -v | column -t | grep RSA
找到Kx=RSA 相关的RSA 交换密钥算法；配置在Nginx的ssl 交换算法上。如下：
 
ssl_ciphers RC4-SHA:ECDH-RSA-DES-CBC3-SHA;
```

客户端(需要用go1.22+ 的版本)：

```
resp, err := http.Get("https://xxx")
    fmt.Println(resp, err) // err 会报错： tls: handshake failure
    if resp != nil {
        data, err := io.ReadAll(resp.Body)
        fmt.Println(data, err)
    }
```

GODEBUG 环境变量中加上 tlsrsakex=1 编译；又可以正常请求。 是这个问题没跑了

## 为什么要删除呢

那golang1.22+版本为啥要删除 rsa key 交换算法的套件呢？

RSA Key 交换算法是一种用于在公共密钥加密和非对称加密之间进行转换的协议。它允许使用 RSA 公钥加密来传输对称密钥，从而避免了中间人攻击。

RSA 密钥交换比较简单；由于加密预主密钥的服务器公钥，一般会保持多年不变。任何能接触到对应死要的人都可以恢复主密钥，并构建相同的主密钥，危害会话安全。有一个专有名词 **Forward secrecy** 前向保证，如过密钥泄漏，之前的请求都可能被恢复。

所以，golang1.22+版本删除了 RSA Key 交换算法的套件。