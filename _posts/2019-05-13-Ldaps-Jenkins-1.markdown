---
layout: post
title: ldap之tls 双向认证要我命
date: Monday, 13. May 2019 11:00PM 
---

先简单介绍一下问题背景，现公司的ldap不止是ldaps还要验证客户端，也就是说在客户端验证服务端的tls证书的同时，服务端也会要求验证客户端的证书，其实并不复杂，平时单向认证见得多了，只要客户端有服务端的ca证书，当拿到服务端自己的证书的时候，就会用ca证书去验证服务端是不是想连接的服务端，所以只要告诉客户端ca证书在哪里就可以。双向认证其实也不难，就加多了让客户端知道自己的证书和私钥在哪里就可以了。这样当服务端请求客户端的证书的时候，客户端知道自己的证书在哪里并且发给服务端之后能拿自己的私钥来解密服务端发过来的随机码。
1. ldap 启用 tls:

- 生成证书和私钥
- 修改 ldap 指定证书和私钥地址
```
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/sasl2/ca.pem
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/sasl2/ldap.pem
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/sasl2/ldap-key.pem
```
- 要求ldap验证客户端证书
```
dn: cn=config
changetype: modify
replace: olcTLSVerifyClient
olcTLSVerifyClient: demand
```

2. 客户端配置ca和自己的证书私钥
java配置truststore和密码，并修改java启动参数
```
root@saltjenkins:~# grep -i keystore /etc/default/jenkins 
JAVA_ARGS="-Djava.awt.headless=true -Djavax.net.ssl.trustStorePassword=changeit -Djavax.net.ssl.trustStore=/var/lib/jenkins/cacerts -Djavax.net.ssl.keyStorePassword=123123 -Djavax.net.ssl.keyStore=/var/lib/jenkins/jenkins.p12"
```
trsutstore和keystore分别是存放ca证书和自己的证书和私钥，因为keystore不支持分别导入证书和私钥，因此需要把pem格式转换成pk12然后再导入
```
cat jenkins.pem jenkins-key.pem > jenkins-p12.txt
openssl pkcs12 -export -in jenkins-p12.txt -out jenkins.pkcs12 -name jenkins.home.kd -noiter -nomaciter
keytool -importkeystore -srckeystore /var/lib/jenkins/jenkins.p12 -destkeystore /var/lib/jenkins/cacerts -srcstoretype pkcs12 -deststoretype jks
```
上面三个命令，前两个命令是把pem格式的证书和私钥转换成pkcs12格式，第三个命令是把pkcs12导入keystore里，然后在java启动的时候指定这个keystore就可以了。

3.制作 truststore, 其实 best practise是把ca 证书放在truststore ,自己的证书和私钥放在 keystore, 分开管理，但是放在一起都叫keystore也没问题。下面的命令拿出ldap的证书
```
root@saltjenkins:~# openssl s_client -connect 192.168.1.111:636 2>/dev/null | sed -n '/-BEGIN CERTIFICATE/,/-END CERTIFICATE-/p'
-----BEGIN CERTIFICATE-----
MIICIjCCAcigAwIBAgIUa93Qjet4RxSMu2YJvuMoaNx4WHYwCgYIKoZIzj0EAwIw
QzELMAkGA1UEBhMCVVMxFjAUBgNVBAgTDVNhbiBGcmFuY2lzY28xCzAJBgNVBAcT
AkNBMQ8wDQYDVQQDEwZjbGllbnQwHhcNMTkwNTEzMTI0MjAwWhcNMjAwNTEyMTI0
MjAwWjBDMQswCQYDVQQGEwJVUzEWMBQGA1UECBMNU2FuIEZyYW5jaXNjbzELMAkG
A1UEBxMCQ0ExDzANBgNVBAMTBmNsaWVudDBZMBMGByqGSM49AgEGCCqGSM49AwEH
A0IABCO5+M2R7y4QW4N0MzDxaFzbZ8w0wKfrVpRyAk+njr+Otm6wzHSdW7K9SMWP
5+iyr/xMGEJATVoT5SPVpYFJgrKjgZkwgZYwDgYDVR0PAQH/BAQDAgWgMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAMBgNVHRMBAf8EAjAAMB0GA1UdDgQW
BBTCYuzgyBEkGcOJ8pnBkazja8Uw2jAfBgNVHSMEGDAWgBRY9e8z7ILjqekNYvL7
+5FuwqdGVjAXBgNVHREEEDAOggxsZGFwLmhvbWUua2QwCgYIKoZIzj0EAwIDSAAw
RQIhAJvu4H/eF7BFRUKPpL0WHHEIWixdj9/4i208rKvXtYfiAiBpO/Hk8LzJtGPa
D8mn6f2axz6VA5aJ47Ut/O8NAJApaw==
-----END CERTIFICATE-----

```
把拿到的证书导入truststore
```
root@saltjenkins:~# keytool -import -trustcacerts -alias ldap -file ldap.crt -keystore cacerts.ts
Enter keystore password:  
Re-enter new password: 
Owner: CN=client, L=CA, ST=San Francisco, C=US
Issuer: CN=client, L=CA, ST=San Francisco, C=US
Serial number: 6bddd08deb7847148cbb6609bee32868dc785876
Valid from: Mon May 13 20:42:00 HKT 2019 until: Tue May 12 20:42:00 HKT 2020
Certificate fingerprints:
	 MD5:  CD:E8:92:A2:27:51:9D:FD:72:37:7D:B4:0F:28:56:86
	 SHA1: A0:6D:32:5C:49:BC:B1:83:11:B5:75:49:EE:7E:02:7E:EF:3F:66:C9
	 SHA256: 81:E0:44:3E:F5:33:BD:DD:AE:53:1F:66:A3:50:DC:D5:62:A8:A5:FD:BE:C8:A0:B3:2B:CA:7A:BF:9F:60:34:47
Signature algorithm name: SHA256withECDSA
Subject Public Key Algorithm: 256-bit EC key
Version: 3

Extensions: 

#1: ObjectId: 2.5.29.35 Criticality=false
AuthorityKeyIdentifier [
KeyIdentifier [
0000: 58 F5 EF 33 EC 82 E3 A9   E9 0D 62 F2 FB FB 91 6E  X..3......b....n
0010: C2 A7 46 56                                        ..FV
]
]

#2: ObjectId: 2.5.29.19 Criticality=true
BasicConstraints:[
  CA:false
  PathLen: undefined
]

#3: ObjectId: 2.5.29.37 Criticality=false
ExtendedKeyUsages [
  serverAuth
  clientAuth
]

#4: ObjectId: 2.5.29.15 Criticality=true
KeyUsage [
  DigitalSignature
  Key_Encipherment
]

#5: ObjectId: 2.5.29.17 Criticality=false
SubjectAlternativeName [
  DNSName: ldap.home.kd
]

#6: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: C2 62 EC E0 C8 11 24 19   C3 89 F2 99 C1 91 AC E3  .b....$.........
0010: 6B C5 30 DA                                        k.0.
]
]

Trust this certificate? [no]:  yes
Certificate was added to keystore
```
几个有用的命令：
`openssl verify -CAfile $cacert $cliencert`
使用ca证书验证
