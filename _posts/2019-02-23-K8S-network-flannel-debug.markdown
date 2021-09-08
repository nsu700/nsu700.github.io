---
layout: post
title: 记一次flannel vxlan的debug过程 
date: 2019-02-24 01:14:16.000000000 +08:00
---

自从部署好k8之后，用了一段时间都是自己配置路由，简单的使用 cni 的网络，简直是累得不行，每次加减新节点都得配置路由，因此就部署了 flannel , 结果今天晚上发现在 node 上居然访问不到服务，仔细看才知道节点之间的 pod 竟无法相互 ping 通，因此才有了今晚的这个过程。

首先介绍一下我的这个环境是由一个master和四个node组成的一个集群，工作节点分别是node1,node2,node3和node4,对应的ip地址分别是192.168.1.12{1,2,3,4} ， 之间的网络方案是使用的 flannel 的 vxlan 模式。简单说一下，flannel有三种工作模式，分别是 udp, vxlan 和 gw-host,官网有大致的介绍，请看 [这里](https://coreos.com/flannel/docs/latest/backends.html)

简单说一下 vxlan 是怎么实现的，也就清楚了我的解题思路了。首先可能会发现部署了 flannel 的 vxlan 模式后，只要正常运行的话，都会看到一个 flannel.1 的网络设备，比如说我这个：

```shell
root@node1:~# ip a s flannel.1
5: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether 3e:7f:7a:12:51:fd brd ff:ff:ff:ff:ff:ff
    inet 9.0.2.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::3c7f:7aff:fe12:51fd/64 scope link 
       valid_lft forever preferred_lft forever
```

这是一个叫 vtep 的设备，在这个模式中主要是为了封装和解封二层数据帧。也就是说如果从一个节点上的 pod1 发送流量到另外一个节点上的 pod2 的话，网络流量首先从 pod1 出来，经过 veth pair 的链接出现在了 cni0 的网桥上，这个 cni0 跟 docker0 是一样的，这是 k8s 为了隐藏容器细节而抽象出来的一个设备，然后这时根据路由表，比如我的路由表这样：

```shell
root@node1:~# ip route 
default via 192.168.1.1 dev eth1 
9.0.0.0/24 via 9.0.0.0 dev flannel.1 onlink 
9.0.1.0/24 via 9.0.1.0 dev flannel.1 onlink 
9.0.2.0/24 dev cni0 proto kernel scope link src 9.0.2.1 
9.0.3.0/24 via 9.0.3.0 dev flannel.1 onlink 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 
192.168.1.0/24 dev eth1 proto kernel scope link src 192.168.1.121 
root@node1:~# 
```

为了方便解释，这里假设 pod1 的 ip 是 9.0.2.18 ，pod2 的 ip 是 9.0.3.3， 根据路由规则要去 9.0.3.3 的流量下一跳应该是 经过 flannel.1 到 9.0.3.0 这里，（onlink的意思是下一跳是直接连接到当前设备的），因此这个流量从 cni0 出来之后就到了 flannel.1 ，当 flannel.1 收到这个数据包之后呢，就需要把这个包封装成二层数据帧，也就是添加对应的 MAC 地址，比如说：

```shell
root@node2:~# ip neigh show dev flannel.1 | grep 9.0.3.0
9.0.3.0 lladdr c2:8a:fe:64:a4:f0 PERMANENT
```

可能有人会问，为什么是 9.0.3.0 而不是 9.0.3.3 呢？因为这个 vxlan 的设计是通过每个 flannel.1 设备虚拟连接成一个二层网络，而根据上面的路由表我们知道要去往 9.0.3.3 的路必须先经过 9.0.3.0 ，也就是另外一个节点上的 flannel.1 设备。等 flannel.1 封装完这个包之后呢， linux 内核还需要给这个包加一个 VXLAN 的头以表示这是一个 VXLAN 的包，然后再送到真正的网卡通过 udp 发送出去，可是发送到哪里呢？怎么知道发送到哪里呢？ 这就是 FDB 的作用了。

```shell
root@node2:~# bridge fdb show | grep c2:8a:fe:64:a4:f0
c2:8a:fe:64:a4:f0 dev flannel.1 dst 10.0.2.15 self permanent
root@node2:~# 
```

这就是我的 FDB 的数据了，用过 vagrant 的同学可能会知道问题出在哪里了，解释完这个流程先。这个 fdb 规则告诉 linux 内核，发往 c2:8a:fe:64:a4:f0 也就是 9.0.3.0 的数据包应该通过 flannel.1 发向 10.0.2.15 这个地址去。正常来说这个 ip 地址就应该是 pod2 运行的节点地址了，然后当接收节点 node2 接收到这个数据包之后呢，就会先拆下 10.0.2.15 这个三层地址头之后发现是一个 VXLAN 的包，然后拆下对应的包头后发现是给 node2  上的 flannel.1 设备，就把包交给 flannel.1 然后 flannel.1 解封这个包之后发现是给 pod2 的就会送到 cni0 然后给到 pod2.

我这个问题就出在 10.0.2.15 上，这是 vagrant 设置的一个默认网卡，使用的是 nat 模式的网络，因此这个 ip 只能用于虚拟机和 host 之间通讯，导致网络包无法发送到另外一个虚拟机节点上。那怎么解决这个问题呢？

其实首先我查看了官网，发现有这么一段话 ： 

```english
Running on Vagrant
Vagrant has a tendency to give the default interface (one with the default route) a non-unique IP (often 10.0.2.15).
This causes flannel to register multiple nodes with the same IP.
To work around this issue, use --iface option to specify the interface that has a unique IP.
If you're running on CoreOS, use cloud-config to set coreos.flannel.interface to $public_ipv4.
```

这个问题呢，其实有两种解决办法，第一个是禁用掉这个网卡，不过如果禁用了这个虚拟设备的话，以后就无法使用 vagrant 的 provision 功能了，因为这个 10.0.2.15 对我来说并没啥作用，因此我直接在系统里面禁用掉了这个虚拟网卡，这回这个 daemonset 就能正常运行了，问题到这里也就解决了。但是我这个并不是最好的解决办法，因为有些机器避免不了会使用到多网卡，这时候就应该是自己指定 iface 了。

由于我是使用 vagrant 加 ansible 来部署这个集群，因此禁用网卡这个事情就交回给了 vagrant 了，也算是有始有终了，就是在 Vagrantfile 里面加这么一段：

```ruby
config.vm.provision "shell",
      run: "once",
      inline: "sudo sed '/eth0/d' /etc/network/interfaces && sudo systemctl restart networking"
```

第二个解决办法呢，就是使用 flanneld 的 --iface， 因为我是部署的 flanneld 的 daemonset , 因此在环境里执行这个命令， `kubectl -n kube-system edit daemonset kube-flannel` ,然后在对应的 `container`里的`args`添加上`--iface=eth`,如下：

```
containers:
      - name: kube-flannel
        image: quay.io/coreos/flannel:v0.11.0-amd64
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=eth1     #添加了这一行
```

