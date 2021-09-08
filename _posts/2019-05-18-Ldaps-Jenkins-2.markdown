---
layout: post
title: 记录一次 jenkins 事故之初探 tls 双向认证
date: Monday, 13. May 2019 11:00PM 
---

先简单介绍一下问题背景，现公司的ldap不止是ldaps还要验证客户端，也就是说在客户端验证服务端的tls证书的同时，服务端也会要求验证客户端的证书，其实并不复杂，平时单向认证见得多了，只要客户端有服务端的ca证书，当拿到服务端自己的证书的时候，就会用ca证书去验证服务端是不是想连接的服务端，所以只要告诉客户端ca证书在哪里就可以。双向认证其实也不难，就加多了让客户端知道自己的证书和私钥在哪里就可以了。这样当服务端请求客户端的证书的时候，客户端知道自己的证书在哪里并且发给服务端之后能拿自己的私钥来解密服务端发过来的随机码。
先介绍一下我是如何配置 jenkins 的，其实在这个案例中就是 ldaps 的客户端，前面解释过，因为 ldaps 要求 verify client ，  因此我们这个 jenkins 不只要有信任的 ca 证书，还需要有自己的私钥和证书。

但是在解决问题之前，有必要了解 TLS 是如何工作的，如何建立连接的，要不根本就不知道中间经历了什么，就无所谓怎么解决问题了。如果启用 debug 模式去看整个 ssl 的过程，是可以大概了解到一些的：

```
root@saltjenkins:~# tail -f /var/log/jenkins/jenkins.log | grep "^\*\*\*"
*** ClientHello, TLSv1.2
***
*** ServerHello, TLSv1.2
***
*** Certificate chain
***
*** ECDH ServerKeyExchange
*** CertificateRequest
*** ServerHelloDone
*** Certificate chain
***
*** ECDHClientKeyExchange
*** CertificateVerify
*** Finished
***
*** Finished

```
其实这里可以看出整个 TLS 的双向验证的过程，我画了个图大概如下：
![ ](http://www.plantuml.com/plantuml/png/ZO-n3e8m54LtlgA9A_o00mkI4DG54XVZKCk5DZIFsXOZVo-Y0r58NDyzvxxtSb2ho4NdpLNtkCJiK773jane1V9CGkikUCsYcELlTanBs3iiovRJ1DQhMWdkiQRkCQGF8Jar2ETyWLiFvyCFDYtOTOGWNxbpvev5qz6pxd-q4wogXr6UZ4GP2LiQY92b9EnWFAgCK-KaRt64SxnP-EfR_YLMovu0  "TLS")

按照顺序 tls 的双向验证步骤大概如下：
1.客户端发送建立连接请求， 包括一个随机数，tls 版本，自己支持的 cipher suit，抓包内容如下：
```
Transport Layer Security
    TLSv1.2 Record Layer: Handshake Protocol: Client Hello
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 220
        Handshake Protocol: Client Hello
            Handshake Type: Client Hello (1)
            Length: 216
            Version: TLS 1.2 (0x0303)
            Random: 5cdfb1301211cf30ca14e9bf939e1382200b634361d4d55e…
            Session ID Length: 0
            Cipher Suites Length: 86
            Cipher Suites (43 suites)
                Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384 (0xc024)
       .....省略一部分.....
                Cipher Suite: TLS_EMPTY_RENEGOTIATION_INFO_SCSV (0x00ff)
            Compression Methods Length: 1
            Compression Methods (1 method)
            Extensions Length: 89
            Extension: supported_groups (len=22)
            Extension: ec_point_formats (len=2)
            Extension: signature_algorithms (len=28)
            Extension: extended_master_secret (len=0)
            Extension: server_name (len=17)```
```
2.服务端收到请求后也回复一个 Hello， 包括自己的 tls 版本，一个随机数，和选定的 cipher suite等等：
```
Transport Layer Security
    TLSv1.2 Record Layer: Handshake Protocol: Server Hello
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 91
        Handshake Protocol: Server Hello
            Handshake Type: Server Hello (2)
            Length: 87
            Version: TLS 1.2 (0x0303)
            Random: 5cdfb10df63dd653624975ce5d6259b7df423fefc2ec4b7f…
            Session ID Length: 32
            Session ID: c7fc57f7dad81235e5b8a4d6464e95ffe2ff1718dcb75461…
            Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384 (0xc024)
            Compression Method: null (0)
            Extensions Length: 15
            Extension: extended_master_secret (len=0)
            Extension: renegotiation_info (len=1)
            Extension: ec_point_formats (len=2)
```
3.接着服务端发送自己的证书给客户端：
```
Transport Layer Security
    TLSv1.2 Record Layer: Handshake Protocol: Certificate
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 1061
        Handshake Protocol: Certificate
            Handshake Type: Certificate (11)
            Length: 1057
            Certificates Length: 1054
            Certificates (1054 bytes)
```
4.然后服务端请求客户端发送证书，因为我们这里是双向验证，所以其实多了两步， ServerKeyExchange 和 CertificateRequest. CertificateRequest 这个就是请求客户端发送证书， 因为 ldap 要求了 verifyClient. 但是这个 ServerKeyExchange 就有点不同了，这个应该是只有 DH 算法才有的这一步，因为 Diffie-Hellman 算法， 客户端无法自己计算出来 Pre-master secret, 客户端需要从服务端获得 DH 算法的公钥， 而这个公钥并不包含在服务端的证书里面，所以它才需要发多一个信息来传递给客户端,这样客户端才能计算出来 Pre-master secret
```
Transport Layer Security
    TLSv1.2 Record Layer: Handshake Protocol: Server Key Exchange
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 147
        Handshake Protocol: Server Key Exchange
            Handshake Type: Server Key Exchange (12)
            Length: 143
            EC Diffie-Hellman Server Params
    TLSv1.2 Record Layer: Handshake Protocol: Certificate Request
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 103
        Handshake Protocol: Certificate Request
            Handshake Type: Certificate Request (13)
            Length: 99
            Certificate types count: 3
            Certificate types (3 types)
            Signature Hash Algorithms Length: 20
            Signature Hash Algorithms (10 algorithms)
            Distinguished Names Length: 71
            Distinguished Names (71 bytes)
    TLSv1.2 Record Layer: Handshake Protocol: Server Hello Done
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 4
        Handshake Protocol: Server Hello Done
            Handshake Type: Server Hello Done (14)
            Length: 0

```
5.接着客户端回复自己的证书：
```
Transport Layer Security
    TLSv1.2 Record Layer: Handshake Protocol: Multiple Handshake Messages
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 632
        Handshake Protocol: Certificate
            Handshake Type: Certificate (11)
            Length: 558
            Certificates Length: 555
            Certificates (555 bytes)
        Handshake Protocol: Client Key Exchange
            Handshake Type: Client Key Exchange (16)
            Length: 66
            EC Diffie-Hellman Client Params
                Pubkey Length: 65
                Pubkey: 04ad0050ae430146da8149107acf0a2ee3f8e9fb4e4900d6…
```
6.客户端告诉服务端已验明正身, 并通过 changeCipherSpec 告诉服务端它已经完成了自己的握手阶段， 在 encryptedHandshakeMessage 里总结了自己经历的所有握手通信的过程， 因为 TLS 要求有客户端和服务端双方的 encryptedHandshakeMessage 来确保没有中间人发起降级攻击
```
Transport Layer Security
    TLSv1.2 Record Layer: Handshake Protocol: Certificate Verify
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 79
        Handshake Protocol: Certificate Verify
            Handshake Type: Certificate Verify (15)
            Length: 75
            Signature Algorithm: ecdsa_secp256r1_sha256 (0x0403)
            Signature length: 71
            Signature: 3045022071983de55816f586cbaa5c6eade62f4a5c2adc3e…
    TLSv1.2 Record Layer: Change Cipher Spec Protocol: Change Cipher Spec
        Content Type: Change Cipher Spec (20)
        Version: TLS 1.2 (0x0303)
        Length: 1
        Change Cipher Spec Message
    TLSv1.2 Record Layer: Handshake Protocol: Encrypted Handshake Message
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 96
        Handshake Protocol: Encrypted Handshake Message
```
7.服务端告诉客户端它完成了他自己的握手阶段，接下来就发送 encryptedHandshakeMessage
```
Transport Layer Security
    TLSv1.2 Record Layer: Change Cipher Spec Protocol: Change Cipher Spec
        Content Type: Change Cipher Spec (20)
        Version: TLS 1.2 (0x0303)
        Length: 1
        Change Cipher Spec Message
```
8.服务端发送encryptedHandshakeMessage：
```
Transport Layer Security
    TLSv1.2 Record Layer: Handshake Protocol: Encrypted Handshake Message
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 96
        Handshake Protocol: Encrypted Handshake Message
```
至此握手结束，双方都拥有了加密的钥匙，以后的信息都可以通过双方认知的加密方式加密了。

参考：
[TLS_Handshake](https://wiki.osdev.org/TLS_Handshake#Change_Cipher_Spec_Message) 


