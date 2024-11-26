---
layout: post
title: 在 Proxmox 上安装 ocp461
date: 2022-10-19 19:52:24.000000000 +08:00
tags: [ocp]
---

# 在 Proxmox 上安装 ocp461

#### 前言

本文介绍如何在 Proxmox 上使用 upi(User Provision Infrastructure) 的方式安装一个简单的 OpenShift4.6.1 的集群，其他版本只要都是 OpenShift4.x, 其实安装方法没什么差异。如果安装过 3.11 的朋友应该知道整个安装过程有多复杂，但是从 4 之后推出了 ipi(Installer Provision Infrastructure),真的是只要一键就能搭建一个全新集群，最最方便的做法比如说 aws, 只要把 `openshift-install` 下载下来，然后跑个命令 `openshift-install create cluster`, 接着按照提示输入 sshkey, aws验证， pullSecret 之后就等大概半个小时，安装完毕后会输出 OpenShift 的 url 和登陆验证信息，就这么简单就搭建好一个集群了。

但是考虑到为了更清楚的了解 OpenShift 跟自己家实验使用，因此选择了 UPI 的方式来安装。我自己有一台 Xeon 2680v2 + 96G 的机器，试过 RHV， 试过 Exsi，都不太理想，机器配置跟不上，更别提 RHV 经常 oom, 因此就选择了 Proxmox，不得不说还是 debian 大法好阿，非常顺滑。

如何安装 proxmox ,请参考 [此文](https://www.proxmox.com/en/proxmox-ve/get-started)

准备好虚拟环境之后呢，还需要几张 iso 以便安装：
1. debian 或者 rhel 或者任何其他系统只要能跑容器就可以，用于 quay 镜像跟 dns, haproxy
2. rhcos 用于 openshift 安装，如果是PXE安装的话，镜像有点不一样，请参考 [此文](https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-bare-metal.html#installation-user-infra-machines-pxe_installing-bare-metal). 因为我这里机器不多，就全手动形式安装，主要需要下载两个文件，请参考 [此文](https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-bare-metal.html#installation-user-infra-machines-iso_installing-bare-metal)
在 [RHCOS image mirror](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.6/) 下载 [`rhcos-4.6.1-live.x86_64.iso`](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.6/4.6.1/rhcos-4.6.1-x86_64-live.x86_64.iso) 和 [`rhcos-metal.x86_64.raw.gz`](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.6/4.6.1/rhcos-metal.x86_64.raw.gz)

除此之外还需要下载的是 OpenShift 的 CLI, 从 [这里](https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/) 下载 `oc` 命令和 `openshift-install`

另外要注意的是 OCP4.6 之后才支援使用 network-kargs 这样去配置网络，因此在4.6之前必须手动进去配置，详情查阅 [这里](https://access.redhat.com/solutions/5499911)

准备妥当后就要开始我们这次安装的奇妙旅程了。

#### 安装过程解释

![png](../pics/bootstrap-process.png)

1. 在 bootstrap 机器上建立一套 etcd 及临时集群，主要是提供 api 跟 machineConfig
2. 

#### 需要解决的问题
1. IP规划，master, worker node, lb
2. 搭建镜像服务器，自签证书
3. 制作离线镜像服务器
4. 拉取 openshift-install 命令，千万注意这里的镜像路径，因为后面的 /var/usrlocal/bin/release-image-download.sh 是写死了地址去拉镜像
4. 创建 DNS 服务器
5. 创建 LB
6. 创建 install-config.yaml 并生成 ignition-configs
7. 信任自签证书
8. 建立 bootstrap 机器
9. 建立其他机器
10. 拉取 registry.redhat.io/rhel8/support-tools
10. 定期同步镜像

#### IP

因为这次的安装只有 3 个 master 节点，熟悉 Kubernetes 的朋友都知道主要有两种节点角色 master 和 worker,主要是 master 上面会运行控制平面的东西，比如 `apiserver`, `controller-manager`, `scheduler` 等等还有 `etcd`, 而 worker 上面主要是客户的工作负载了，也就是客户自己的应用还有一些监控日志收集类的应用比如 `exporter`, `fluentd` 之类的。

| Hostname | IP |
| --- | --- |
| bootstrap.example.io | 192.168.38.100 |
| master1.example.io | 192.168.38.11 |
| master2.example.io | 192.168.38.12 |
| master3.example.io | 192.168.38.13 |
| etc-0.example.io | 192.168.38.11 |
| etc-1.example.io | 192.168.38.12 |
| etc-2.example.io | 192.168.38.13 |
| api.sb95q.dynamic.redhatworkshops.io | 192.168.38.10 |
| api-int.sb95q.dynamic.redhatworkshops.io | 192.168.38.10 |
| apps.sb95q.dynamic.redhatworkshops.io | 192.168.38.10 |

#### 配置 dns 和 loadbalance

官方文档请参考[网络需求](https://docs.openshift.com/container-platform/4.6/installing/installing_bare_metal/installing-bare-metal.html#installation-network-user-infra_installing-bare-metal)

###### 安装所需软件

```
yum install nginx haproxy dnsmasq -y
```

###### DNS

这里是采用了 dnsmasq，比较方便，如果是使用 coredns 还要搭配上 etcd, 因为 etcd 是利用 DNS 的 [srv](https://etcd.io/docs/v2/clustering/) 记录来发现彼此从而形成一个集群的。因此 DNS 需要解析的主要有两组记录：

1. 地址

主要四台主机的地址加上几个域名

| Record | 描叙 | 后端 |
| --- | --- | --- |
| api.<cluster_name>.<base_domain>. | 要求集群内外都可以解析到这个地址 | master |
| api-int.<cluster_name>.<base_domain>. | 要求集群内部可以解析到这个地址 | master |
| *.apps.<cluster_name>.<base_domain>. | 应用 expose 出来的 route | router |

2. SRV

三台 etcd 主机的 SRV 记录，dnsmasq 的写法如下：
```bash
srv-host=_etcd-server._tcp.example.io,master1.example.io,2380,0,100
srv-host=_etcd-server._tcp.example.io,master2.example.io,2380,0,100
srv-host=_etcd-server._tcp.example.io,master3.example.io,2380,0,100
```

总结上面两个部分，一个完整的 dnsmasq 需要解析的地址列表如下，如果有 worker 就只需要增加 worker 节点的地址记录就可以了。

```dns
address=/api.sb95q.dynamic.redhatworkshops.io/192.168.38.10
address=/api-int.sb95q.dynamic.redhatworkshops.io/192.168.38.10
address=/apps.sb95q.dynamic.redhatworkshops.io/192.168.38.10

address=/bootstrap.example.io/192.168.38.100
address=/master1.example.io/192.168.38.11
address=/master2.example.io/192.168.38.12
address=/master3.example.io/192.168.38.13

srv-host=_etcd-server._tcp.example.io,master1.example.io,2380,0,100
srv-host=_etcd-server._tcp.example.io,master2.example.io,2380,0,100
srv-host=_etcd-server._tcp.example.io,master3.example.io,2380,0,100
```

###### 负载均衡

这里使用 haproxy 来提供负载均衡功能，也可以使用 envoy. 而需要提供负载均衡的主要就是上面 DNS 部分提及到的几组域名，包括：

1. API
主要是两个端口需要配置：
| 端口 | 后端 | 内部访问 | 外部访问 | 描叙 |
| --- | --- | --- | --- | ---- |
| 6443 | bootstrap, master | Y | Y | Kuberentes API server |
| 22623 | boostrap, master | Y | N | Machine Config server |

第一个6443这个端口主要是用于集群跟 api-server 的通信，因此集群内外都需要可以访问到；第二个22623端口则只是 `MachineConfig` 需要而以，因此只需要内部访问即可。需要注意的是因为 OpenShift4 的安装过程需求，详情请参考[安装过程解释](#安装过程解释), 因此这两组记录里都需要添加上 bootstrap 机器，但是在机器安装完成后就需要把 bootstrap 机器从这两组记录里拿掉。

2. 应用入口 ingress
用于应用外部访问，因此只要能负载到集群上的 router 的地址就可以，不需要 bootstrap 机器。

总结上面两个部分，一个完整的 haproxy 配置可以如下：

```haproxy
global
  maxconn  2000
  ulimit-n  16384
  log  127.0.0.1 local0 err

defaults
  log global
  mode  http
  option  httplog
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  timeout http-request 15s
  timeout http-keep-alive 15s

listen stats
    bind         :9000
    mode         http
    stats        enable
    stats        uri /
    stats        refresh   30s
    stats        auth      admin:openshift #web页面登录
    monitor-uri  /healthz

frontend openshift-api-server
    bind :6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

backend openshift-api-server
    balance roundrobin
    mode tcp
    option httpchk GET /readyz
    http-check expect string ok
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server bootstrap 192.168.38.100:6443 check check-ssl verify none #安装结束后删掉此行
    server master1 192.168.38.11:6443 check check-ssl verify none
    server master2 192.168.38.12:6443 check check-ssl verify none
    server master3 192.168.38.13:6443 check check-ssl verify none

frontend machine-config-server
    bind :22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance roundrobin
    mode tcp
    server bootstrap 192.168.38.100:22623 check #安装结束后删掉此行
    server master1 192.168.38.11:22623 check
    server master2 192.168.38.12:22623 check
    server master3 192.168.38.13:22623 check

frontend ingress-http
    bind :80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance roundrobin
    mode tcp
    server master1 192.168.38.11:80 check
    server master2 192.168.38.12:80 check
    server master3 192.168.38.13:80 check

frontend ingress-https
    bind :443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance roundrobin
    mode tcp
    server master1 192.168.38.11:443 check
    server master2 192.168.38.12:443 check
    server master3 192.168.38.13:443 check
```

###### HAproxy 报错

配置完上述文件之后，就可以尝试启动 haproxy 了，启动也是简单的 `systemctl start haproxy`, 但是如果你是使用 RHEL 系列的话，有可能你的 haproxy 会起不来，比如说报下面的错误：

```
[root@bastion-sb95q ~]# journalctl -u haproxy.service --no-pager
Sep 04 09:13:18 bastion-sb95q systemd[1]: Starting HAProxy Load Balancer...
Sep 04 09:13:18 bastion-sb95q haproxy[44205]: [NOTICE]   (44205) : haproxy version is 2.4.22-f8e3218
Sep 04 09:13:18 bastion-sb95q haproxy[44205]: [NOTICE]   (44205) : path to executable is /usr/sbin/haproxy
Sep 04 09:13:18 bastion-sb95q haproxy[44205]: [ALERT]    (44205) : Starting frontend openshift-api-server: cannot bind socket (Permission denied) [0.0.0.0:6443]
Sep 04 09:13:18 bastion-sb95q haproxy[44205]: [ALERT]    (44205) : Starting frontend machine-config-server: cannot bind socket (Permission denied) [0.0.0.0:22623]
Sep 04 09:13:18 bastion-sb95q haproxy[44205]: [ALERT]    (44205) : [/usr/sbin/haproxy.main()] Some protocols failed to start their listeners! Exiting.
```

这时候只要一行命令就可以搞定了，主要是 selinux 的问题 `setsebool -P haproxy_connect_any=1`, 执行这个命令就可以起来了。

#### 搭建镜像服务器

参考自官方文档 [quay3.4](https://access.redhat.com/documentation/en-us/red_hat_quay/3.4/html/deploy_red_hat_quay_for_proof-of-concept_non-production_purposes/index), 需要放开的一些 tcp 端口如下, 因为是使用容器来运行这些服务，因此实际端口还要看容器运行时如何做的端口映射：

| Port | Protocol | App |
| --- | --- | --- |
| 80 | tcp | Quay |
| 443 | tcp | Quay |
| 5432 | tcp | Postgresql |
| 6379 | tcp | Redis | 

Redis 跟 Postgresql 这两个可以不开放，因为主要是用于 Quay 使用，因此如果没有外部访问需求也可以不开放这两个端口。

1. 创建 quay 用户，因为 quay 镜像里的 uid 是1001,因此创建一个 uid 为 1001 的用户以便管理文件权限

```bash
useradd -u 1001 quay -s /bin/nologin
```

2. 文件准备

为了让容器可以有持久化存储，这里使用 hostpath 的方式来将本地目录挂载到容器里面， 以便容器可以在重启之后还能保有原有数据。

```bash
mkdir -p /svc/quay/{redis,postgresql,config,storage}
chown -R quay:quay /svc/quay
```


1. Redis

2. Postgresql

3. 自签证书

4. Quay

#### OCP 安装文件准备

1. 先配置 install-config.yaml
```

```
2. 生成 OCP ignition 文件
```
[root@bastion-sb95q ocp]# ls
install-config.yaml
[root@bastion-sb95q ocp]# openshift-install create manifests
INFO Consuming Install Config from target directory
[root@bastion-sb95q ocp]# ls
manifests  openshift
[root@bastion-sb95q ocp]# openshift-install create ignition-configs
INFO Consuming Openshift Manifests from target directory
INFO Consuming Common Manifests from target directory
INFO Consuming Worker Machines from target directory
INFO Consuming Master Machines from target directory
INFO Consuming OpenShift Install (Manifests) from target directory
[root@bastion-sb95q ocp]# ls
auth  bootstrap.ign  master.ign  metadata.json  worker.ign
[root@bastion-sb95q ocp]#
```
3. 配置 bootstrap ignition
```
[root@bastion-sb95q ocp]# vi append-bootstrap.ign
[root@bastion-sb95q ocp]# cat append-bootstrap.ign
{
  "ignition": {
    "config": {
      "append": [
        {
          "source": "http://192.168.38.10:81/bootstrap.ign",
          "verification": {}
        }
      ]
    },
    "timeouts": {},
    "version": "2.2.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {}
}
```
4. 把上述的 bootstrap.ign, master.ign, worker.ign, append-bootstrap.ign 放到 nginx 的目录下，确保能通过 nginx 访问到这些文件，因为 coreos 启动的时候需要去拉取这些文件 `cp *ign /usr/share/nginx/html/`

#### 启动配置解释
1. ip这一行主要是配置 IP地址，格式是。。。。。，其中要注意的是网卡这里，如果只有一张网卡不需要特别指明的话，建议用默认做法，也就是说不要指定，因为我碰到这样的坑，可能某个版本的 RHEL 识别出来的名字不同，有的是 ens18, 有的是 ens192, 如果指错了的话，机器会起不来，所以建议不要自己去设置
2. nameserver
3. 
#### Bootstrap

如果是安装 4.4 的版本，切记要选择 Compatible with ESXi6.5 and later, 然后选择 RHEL7

在安装界面摁 tab, 然后下面会出来这样一行, 就在这一行后面添加上后面的配置
```
>/images/vmlinux initrd=/images/initramfs.img nomodeset rd.neednet=1 coreos.inst=yes
```

ip=192.168.38.40::192.168.38.1:255.255.255.0:bootstrap.example.io:ens18:none nameserver=192.168.38.136 coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.38.10/rhcos-443.raw.gz coreos.inst.insecure coreos.inst.ignition_url=http://192.168.38.10/bootstrap.ign 

需要加上 `coreos.inst.insecure`, 要不机器启动的时候除了去拉 rhcos-443.raw.gz 之外，还会去拉 rhcos.sig 去作签名认证，因此如果没有这个签名文件的话，就需要加上这个 inseucre.

#### Master

ip=192.168.38.50::192.168.38.1:255.255.255.0:master1.example.io:ens18:none nameserver=192.168.38.136 coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.38.10/rhcos-443.raw.gz coreos.inst.insecure coreos.inst.ignition_url=http://192.168.38.10/master.ign

#### Worker

ip=192.168.38.61::192.168.38.1:255.255.255.0:worker1.example.io:ens18:none nameserver=192.168.38.136 coreos.inst.install_dev=sda coreos.inst.image_url=http://192.168.38.10/rhcos-443.raw.gz coreos.inst.insecure coreos.inst.ignition_url=http://192.168.38.10/worker.ign


#### Reference
https://zhangguanzhang.github.io/2020/09/18/ocp-4.5-install/#%E5%AE%89%E8%A3%85%E9%9B%86%E7%BE%A4
