---
layout: post
title: Docker系列之MTU debug
date: Saturday, 04. May 2019 02:19PM 
---

昨天花了一整天时间解决了一个由MTU引起的问题，事情的起因是前天收到一个request， 开发人员要求部署个自定制的jenkins，依赖docker和anisble, 二话不说，架起sublimetext就写了个salt脚本,一键部署，各种顺畅，没想到事情来了。
Docker 起来很正常，没有任何错误征兆，但是 jenkins 却一直停留在“jenkins is starting” 的界面，这就很怪异了。
起初以为是读写什么东西没权限，因为在 jenkins community里有人说要把 JENKINS_HOME的owner都改成jenkins， 但是也不奏效，最要命的就是我把 log.properties改成了 info， 还是什么日志都没有。
跟开发人员一再确认需要连接什么服务，因为我在部署之前都已经找了network team开了对应的端口，需要的 mysql , sournar, ldap 之类的端口都是可以正常访问的，对比了一下本地测试的环境，运行的jenkins实例里也确实没有其他的配置需要访问其他的服务。
想着卡在 "jenkins is starting" 那应该是初始化的阶段了，就google了一下 jenkins initilization， 结果很意外的这个 [官页](https://jenkins.io/doc/developer/initialization/) 还是WIP阶段。
想起自己搭建jenkins的时候，初始化阶段就是验证密码，安装插件等等,安装插件…………墙内朋友可能都有过这个经历， `https://updates.jenkins.io/update-center.json`这个访问很慢，所以很多人都会改成清华的那个镜像，但是没道理啊，我们这个机器不受这个影响的啊，其他的jenkins实例也没有改过这个配置。但是也不能错过什么蛛丝马迹，就登上docker试试`curl -I https://updates.jenkins.io/update-center.json`， 结果一直没反应，不会是网络问题吧，跳出来在host机器上试了一下，咦， 马上就303,这里303是因为这个link会被重定向至`https://updates.jenkins.io/current/update-center.json`。 难道会是 docker0 网桥相关的iptables把这个域名给禁了，马上查看iptables， 却发现并没有。但是即使知道了docker里面访问不了这个地址，但是也不确定是不是就是这个原因导致的jenkins起不来。就在这个页面 [jenkins 镜像](http://mirrors.jenkins-ci.org/status.html) 里随便找了一个换上去，结果jenkins还真就起来了，好吧现在确认是update center的问题，其实到这里看是解决了问题，但是其实并没有，因为在这个镜像列表里面提供的都是http或者ftp的镜像，而我们的问题却是https， 这时候其实我也没多想，就告诉开发人员这个链接要改，然后把我刚刚用的http的镜像链接发给了他们，没想到他们却告诉我说用不了，因为他们是用的 [jenkinsci](https://github.com/jenkinsci/configuration-as-code-plugin) 这个工具去配置jenkins ,新的链接一直报错无法加进去，我只能又回到docker里面研究这两个链接怎么回事了。当我使用`curl -vvv -I https://update.jenkins.io/update-center.json`的时候，发现了卡在 tls握手阶段：
```
root@43b9ee5bbb5e:/# curl -I -vvv https://www.baidu.com
* Rebuilt URL to: https://www.baidu.com/
*   Trying 183.232.231.172...
* TCP_NODELAY set
* Connected to www.baidu.com (183.232.231.172) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* Cipher selection: ALL:!EXPORT:!EXPORT40:!EXPORT56:!aNULL:!LOW:!RC4:@STRENGTH
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.2 (OUT), TLS header, Certificate Status (22):
* TLSv1.2 (OUT), TLS handshake, Client hello (1):

```
我当时试的并不是 baidu， 不过是一样的结果，反正就是只看到发出去两个之后就没下文了，导致我一度怀疑是证书问题，但是如果证书不行应该服务端也有回应啊。而且很奇怪的是同样的 docker image 在其他环境都正常，就只有这个新的环境不行。
实在没头绪，我就跑到宿主机里 tcpdump 了，结果注意到一个地方， mss 1440, 这个数值我现在不是很确定了，但是肯定不是1460,回头再看看 docker0 和 eth0 的mtu , 一个是1500, 一个是1400, 心想问题肯定是在这里了，就把 docker0 的 mtu 改成了1400, 修改方法如下：
```
root@node2:~# cat /etc/docker/daemon.json 
{
  "mtu": 1400
}
```
把 mtu 对应的数字改成了1400,然后重启 docker 服务，正常了，jenkins可以正常起来了，这个问题至此就解决了。
但是因为我的网络基础不好，我以为 docker0 是经由 eth0 出去的，那 eth0 的 mtu 是1400， docker0 就不能超过1400了，然后我就试了1420, 1440几个数值，结果却都可以， 好像是到了1450左右就不行了，这一点我到现在还是没搞明白，为什么可以超过 eth0 呢， 如果说 docker0 出去的数据包可以分片，那为什么1500就不行了呢， 等我再研究研究，改天再仔细说说这个问题。
还有为什么http可以https就不行呢，难道刚刚好 tls 的握手就超出了 mtu 么？