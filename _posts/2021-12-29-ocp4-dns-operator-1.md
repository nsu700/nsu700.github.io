---
layout: post
title: OCP4 是如何炼成的之 DNS 篇一
date: 2021-12-29 18:32:24.000000000 +08:00
tags: [ocp, operator]
---

### OCP4 上的 DNS operator 是什么样子的？

因为OCP4上的全新设计，在 DNS 处理方面全部是透由 cluster-dns operator 去处理，因此要了解 OCP4 上是如何管理 DNS 的，我们先从这个 operator 入手。

```bash
❯ oc get co dns -o yaml
apiVersion: config.openshift.io/v1
kind: ClusterOperator
metadata:
...
spec: {}
status:
  conditions:
...
  extension: null
  relatedObjects:
  - group: ""
    name: openshift-dns-operator
    resource: namespaces
  - group: operator.openshift.io
    name: default
    resource: dnses
  - group: ""
    name: openshift-dns
    resource: namespaces
  versions:
  - name: kube-rbac-proxy
    version: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:4f0fb136c17a69dfa3e09b9a519de88ba0acdb088c941a747a3a2771b7452193
  - name: operator
    version: 4.8.10
  - name: coredns
    version: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:1af1fe8e19c135b73be209a30c8b0f51b3e88bd470e25930d0bda637da2bf2a2
  - name: openshift-cli
    version: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:d008ba712c838086e60365fee598e33e54e0d30b1bdf5880f05ebc1a48cdbc2f
```

从上面的命令结果来看，这个 operator 会创建以下资源：

1. namespace — openshift-dns-operator
2. crd — dnses
3. namespace — openshift-dns

接下来让我们拆解这三个资源：

### namespace ⇒ openshift-dns-operator

```bash
❯ oc get all -n openshift-dns-operator
NAME                                READY   STATUS    RESTARTS   AGE
pod/dns-operator-7465949cd4-2qslq   2/2     Running   0          42h

NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/metrics   ClusterIP   172.30.204.142   <none>        9393/TCP   42h

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/dns-operator   1/1     1            1           42h

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/dns-operator-7465949cd4   1         1         1       42h
```

主要是两个东西，一个是 deployment ,一个是  service， 后面的 replicaset 跟 pod 是这个 deployment/dns-operator 生出来的。 如果有兴趣，研究这个 dns-operator 的[源码](https://github.com/openshift/cluster-dns-operator)能更加清晰到底 OCP4 是如何管理 DNS 的

### CRD ⇒ dnses

第二个资源是一个名字为 dnses 的 CRD， 从上面的命令结果来看，它是帮我们创建了一个叫 default  的 dnses

```bash

❯ oc get dnses.operator.openshift.io default -o yaml
apiVersion: operator.openshift.io/v1
kind: DNS
metadata:
  finalizers:
  - dns.operator.openshift.io/dns-controller
  name: default
...
spec:
  nodePlacement: {}
status:
  clusterDomain: cluster.local
  clusterIP: 172.30.0.10
...
❯ oc explain dnses.operator.openshift.io.spec
KIND:     DNS
VERSION:  operator.openshift.io/v1

RESOURCE: spec <Object>

DESCRIPTION:
     spec is the specification of the desired behavior of the DNS.

FIELDS:
   nodePlacement	<Object>
     nodePlacement provides explicit control over the scheduling of DNS pods.
     Generally, it is useful to run a DNS pod on every node so that DNS queries
     are always handled by a local DNS pod instead of going over the network to
     a DNS pod on another node. However, security policies may require
     restricting the placement of DNS pods to specific nodes. For example, if a
     security policy prohibits pods on arbitrary nodes from communicating with
     the API, a node selector can be specified to restrict DNS pods to nodes
     that are permitted to communicate with the API. Conversely, if running DNS
     pods on nodes with a particular taint is desired, a toleration can be
     specified for that taint. If unset, defaults are used. See nodePlacement
     for more details.

   servers	<[]Object>
     servers is a list of DNS resolvers that provide name query delegation for
     one or more subdomains outside the scope of the cluster domain. If servers
     consists of more than one Server, longest suffix match will be used to
     determine the Server. For example, if there are two Servers, one for
     "foo.com" and another for "a.foo.com", and the name query is for
     "www.a.foo.com", it will be routed to the Server with Zone "a.foo.com". If
     this field is nil, no servers are created.
```

从第一个命令的输出结果可以知道 nodePlacement 默认情况下是没有设置的，也就是说会把 DNS  pod 跑在所有节点上， 那为什么要这样子做呢，第二个命令的解析里有这么一句把 DNS 跑在每个节点上，可以让 DNS 查询全都在本地进行，避免跨网络，这样可以让 DNS 解析更加高效。 如果说因为某些安全或者其他原因不想每个节点都跑 DNS ,也可以在这里设置

然后第一个命令的输出结果里面还有两个地方，clusterDomain 和 clusterIP

clusterIP 是 DNS 的 service IP, 我们在接下来的章节会介绍到

clusterDomain 这个是 cluster.local，这是 k8s 里的本地域名，有点类似 localhost

### namespace ⇒ openshift-dns

接下来看第三个资源，一个名字是 openshift-dns 的 namespace

```bash
❯ oc get all -n openshift-dns
NAME                      READY   STATUS    RESTARTS   AGE
pod/dns-default-62ppd     2/2     Running   0          43h
pod/dns-default-dcftg     2/2     Running   2          43h
pod/dns-default-flfs5     2/2     Running   2          43h
pod/dns-default-m7bcg     2/2     Running   0          43h
pod/dns-default-qzdqr     2/2     Running   2          43h
pod/dns-default-tjrb5     2/2     Running   0          43h
pod/node-resolver-2b9l6   1/1     Running   0          43h
pod/node-resolver-42fpq   1/1     Running   0          43h
pod/node-resolver-fn25w   1/1     Running   0          43h
pod/node-resolver-gwrw4   1/1     Running   0          43h
pod/node-resolver-h7l57   1/1     Running   0          43h
pod/node-resolver-ktrb5   1/1     Running   0          43h
pod/node-resolver-vj6rd   1/1     Running   1          43h
pod/node-resolver-xgqhd   1/1     Running   1          43h
pod/node-resolver-ztc88   1/1     Running   1          43h

NAME                  TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
service/dns-default   ClusterIP   172.30.0.10   <none>        53/UDP,53/TCP,9154/TCP   43h

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/dns-default     6         6         6       6            6           kubernetes.io/os=linux   43h
daemonset.apps/node-resolver   9         9         9       9            9           kubernetes.io/os=linux   43h
```

首先看  service，注意到它的 CLUSTER-IP 就是我们上面所提到的 172.30.0.10，所以集群里的 DNS 解析都可以指向这个 IP，但是有个问题，就是如果我把 DNS 指向这个 IP，但是这个 IP 其实是一个 roundrobin 的 lb, 也就是说它并不会根据我的来源地址就近选择 DNS 服务，那这样的话我在每个节点上都跑 DNS 就是浪费了。 另外可以看到它暴露了三个端口，两个53是给 DNS 解析用的，另外一个是 9154是给 prometheus 抓 metrics 用的。

接下来看两个 daemonset 

- dns-default， 它只跑在六个节点上， 为什么只需要六个呢？
- node-resolver，它跑在了所有节点上，这个跟上面那个有什么区别呢？

### daemonset ⇒ dns-default

先看 dns-default

```bash
❯ oc get daemonset.apps/dns-default -o yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    dns.operator.openshift.io/owning-dns: default
  name: dns-default
  namespace: openshift-dns
spec:
  selector:
    matchLabels:
      dns.operator.openshift.io/daemonset-dns: default
  template:
    metadata:
      annotations:
        target.workload.openshift.io/management: '{"effect": "PreferredDuringScheduling"}'
      labels:
        dns.operator.openshift.io/daemonset-dns: default
    spec:
      containers:
      - args:
        - -conf
        - /etc/coredns/Corefile
        command:
        - coredns
        image: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:1af1fe8e19c135b73be209a30c8b0f51b3e88bd470e25930d0bda637da2bf2a2
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: dns
        ports:
        - containerPort: 5353
          name: dns
          protocol: UDP
        - containerPort: 5353
          name: dns-tcp
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 3
          successThreshold: 1
          timeoutSeconds: 3
        resources:
          requests:
            cpu: 50m
            memory: 70Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: FallbackToLogsOnError
        volumeMounts:
        - mountPath: /etc/coredns
          name: config-volume
          readOnly: true
      - args:
        - --logtostderr
        - --secure-listen-address=:9154
        - --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_RSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
        - --upstream=http://127.0.0.1:9153/
        - --tls-cert-file=/etc/tls/private/tls.crt
        - --tls-private-key-file=/etc/tls/private/tls.key
        image: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:4f0fb136c17a69dfa3e09b9a519de88ba0acdb088c941a747a3a2771b7452193
        imagePullPolicy: IfNotPresent
        name: kube-rbac-proxy
        ports:
        - containerPort: 9154
          name: metrics
          protocol: TCP
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/tls/private
          name: metrics-tls
          readOnly: true
      dnsPolicy: Default
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-node-critical
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: dns
      serviceAccountName: dns
      terminationGracePeriodSeconds: 30
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: Corefile
            path: Corefile
          name: dns-default
        name: config-volume
      - name: metrics-tls
        secret:
          defaultMode: 420
          secretName: dns-default-metrics-tls
  updateStrategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 10%
    type: RollingUpdate
```

这个 ds 会生出来两个容器，一个是 kube-rbac-proxy 另外一个是 dns, 我们先看 dns, 首先它使用的镜像是 [`quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:1af1fe8e19c135b73be209a30c8b0f51b3e88bd470e25930d0bda637da2bf2a2`](http://quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:1af1fe8e19c135b73be209a30c8b0f51b3e88bd470e25930d0bda637da2bf2a2，) 然后它启动的命令是 `coredns -conf /etc/coredns/Corefile`, 这里其实挂了一个名字叫 dns-default 的 configmap 进去了，我们看看它的内容

```bash
❯ oc get cm dns-default -n openshift-dns -o yaml | cat -n
     1	apiVersion: v1
     2	data:
     3	  Corefile: |
     4	    .:5353 {
     5	        bufsize 512
     6	        errors
     7	        health {
     8	            lameduck 20s
     9	        }
    10	        ready
    11	        kubernetes cluster.local in-addr.arpa ip6.arpa {
    12	            pods insecure
    13	            fallthrough in-addr.arpa ip6.arpa
    14	        }
    15	        prometheus 127.0.0.1:9153
    16	        forward . /etc/resolv.conf {
    17	            policy sequential
    18	        }
    19	        cache 900 {
    20	            denial 9984 30
    21	        }
    22	        reload
    23	    }
    24	kind: ConfigMap
    25	metadata:
    27	  labels:
    28	    dns.operator.openshift.io/owning-dns: default
    29	  name: dns-default
    30	  namespace: openshift-dns
```

我们仔细来看看这个配置，从第三行开始就是我们 CoreDNS 的配置

| 行数 | 目的 |
| --- | --- |
| 4 | 将服务监听在 5353 端口，对应的看回上面 DS 的 yaml 可以发现，暴露了两个 5353 端口 |
| 5 | 配置了 512 byte 的 buffer |
| 6 | 启用错误日志输出，如果查询过程中发生什么错误，会打印在 stdout |
| 7-9 | 启用服务检查，如果服务正常会在 :8080/health 返回 200， 看回 DS 的 yaml 通用可以发现 livenessProbe 就是去检查这个路径 |
| 10 | 启用服务可用检查，如果服务正常并可使用，会在 :8181/ready 返回200，如果不正常会返回503并在回复消息体里写明那些插件不可用。 同样的看回 DS 也可以看到 readinessProbe 就是利用这个路径做的检查 |
| 11-14 | 使用 kuberntes 插件，提供 pod， svc 查询机制 |
| 15 | 启用 prometheus metrics |
| 16-18 | 如果在 OCP 的 domain 里面查询不到的话，会转发到 /etc/resolv.conf |
| 19-21 | 启用 cache 机制，保留 cache 900 秒钟 |
| 22 | 运行重新载入配置，修改对应的 configmap 后，大概需要两分钟来读取并应用新的配置 |

如果跟 [k8s 的配置](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/) 对比的话，可能有两个问题：

1. 为什么 k8s 的 cache 是 30秒，而 OCP 是 900， 差了30倍？

这个问题我找到了这个 [https://github.com/openshift/cluster-dns-operator/pull/240/](https://github.com/openshift/cluster-dns-operator/pull/240/)

1. 为什么 k8s 会需要 `loadbalance`, 但是 OCP 不要？

关于这个我的理解是如果跑 `nslookup` 会先返回 A 记录，然后返回 AAAA 记录，如果启用了 `loadbalance` 那这两个 ipv4 ipv6 的记录就会轮流返回，这次是 ipv4 先，下一次是 ipv6 先。 OCP 没有启用这个，可能是性能考虑吧，比较真的在 OCP 里用 ipv6 还是很少见的。

以上是这个 DNS 容器主要运行的信息，有个地方还没看到就是 `/etc/resolv.conf`, 这个文件里面是怎样的，它又是怎么产生的呢？

```bash

search ap-south-1.compute.internal
nameserver 10.0.0.2
```

因为我这个是跑在 aws 上面的集群，所以它的 search 是 aws 的域名。但是 nameserver 这个我却查不到，猜想是 rosa 为了机器节点的域名解析而创建的一个东西。

### daemonset ⇒ dns-resolver

我们还有另外一个 DS 没有看，就是 dns-resolver

```bash
❯ oc get ds node-resolver -o yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-resolver
  namespace: openshift-dns
spec:
  selector:
    matchLabels:
      dns.operator.openshift.io/daemonset-node-resolver: ""
  template:
    metadata:
      annotations:
        target.workload.openshift.io/management: '{"effect": "PreferredDuringScheduling"}'
      labels:
        dns.operator.openshift.io/daemonset-node-resolver: ""
    spec:
      containers:
      - command:
        - /bin/bash
        - -c
        - |
          #!/bin/bash
          set -uo pipefail

          trap 'jobs -p | xargs kill || true; wait; exit 0' TERM

          OPENSHIFT_MARKER="openshift-generated-node-resolver"
          HOSTS_FILE="/etc/hosts"
          TEMP_FILE="/etc/hosts.tmp"

          IFS=', ' read -r -a services <<< "${SERVICES}"

          # Make a temporary file with the old hosts file's attributes.
          cp -f --attributes-only "${HOSTS_FILE}" "${TEMP_FILE}"

          while true; do
            declare -A svc_ips
            for svc in "${services[@]}"; do
              # Fetch service IP from cluster dns if present. We make several tries
              # to do it: IPv4, IPv6, IPv4 over TCP and IPv6 over TCP. The two last ones
              # are for deployments with Kuryr on older OpenStack (OSP13) - those do not
              # support UDP loadbalancers and require reaching DNS through TCP.
              cmds=('dig -t A @"${NAMESERVER}" +short "${svc}.${CLUSTER_DOMAIN}"|grep -v "^;"'
                    'dig -t AAAA @"${NAMESERVER}" +short "${svc}.${CLUSTER_DOMAIN}"|grep -v "^;"'
                    'dig -t A +tcp +retry=0 @"${NAMESERVER}" +short "${svc}.${CLUSTER_DOMAIN}"|grep -v "^;"'
                    'dig -t AAAA +tcp +retry=0 @"${NAMESERVER}" +short "${svc}.${CLUSTER_DOMAIN}"|grep -v "^;"')
              for i in ${!cmds[*]}
              do
                ips=($(eval "${cmds[i]}"))
                if [[ "$?" -eq 0 && "${#ips[@]}" -ne 0 ]]; then
                  svc_ips["${svc}"]="${ips[@]}"
                  break
                fi
              done
            done

            # Update /etc/hosts only if we get valid service IPs
            # We will not update /etc/hosts when there is coredns service outage or api unavailability
            # Stale entries could exist in /etc/hosts if the service is deleted
            if [[ -n "${svc_ips[*]-}" ]]; then
              # Build a new hosts file from /etc/hosts with our custom entries filtered out
              grep -v "# ${OPENSHIFT_MARKER}" "${HOSTS_FILE}" > "${TEMP_FILE}"

              # Append resolver entries for services
              for svc in "${!svc_ips[@]}"; do
                for ip in ${svc_ips[${svc}]}; do
                  echo "${ip} ${svc} ${svc}.${CLUSTER_DOMAIN} # ${OPENSHIFT_MARKER}" >> "${TEMP_FILE}"
                done
              done

              # TODO: Update /etc/hosts atomically to avoid any inconsistent behavior
              # Replace /etc/hosts with our modified version if needed
              cmp "${TEMP_FILE}" "${HOSTS_FILE}" || cp -f "${TEMP_FILE}" "${HOSTS_FILE}"
              # TEMP_FILE is not removed to avoid file create/delete and attributes copy churn
            fi
            sleep 60 & wait
            unset svc_ips
          done
        env:
        - name: SERVICES
          value: image-registry.openshift-image-registry.svc
        - name: NAMESERVER
          value: 172.30.0.10
        - name: CLUSTER_DOMAIN
          value: cluster.local
        image: quay.io/openshift-release-dev/ocp-v4.0-art-dev@sha256:d008ba712c838086e60365fee598e33e54e0d30b1bdf5880f05ebc1a48cdbc2f
        imagePullPolicy: IfNotPresent
        name: dns-node-resolver
        resources:
          requests:
            cpu: 5m
            memory: 21Mi
        securityContext:
          privileged: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: FallbackToLogsOnError
        volumeMounts:
        - mountPath: /etc/hosts
          name: hosts-file
      dnsPolicy: ClusterFirst
      hostNetwork: true
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-node-critical
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: node-resolver
      serviceAccountName: node-resolver
      terminationGracePeriodSeconds: 30
      tolerations:
      - operator: Exists
      volumes:
      - hostPath:
          path: /etc/hosts
          type: File
        name: hosts-file
  updateStrategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 33%
    type: RollingUpdate
```

这个主要是使用了 `openshift-cli` 的镜像，然后直接把自己节点上的 /etc/hosts 挂到容器里面的 /etc/hosts, 并且使用了 hostnetwork, 然后跑了一段 bash 脚本，主要是更新**节点上的**/etc/hosts 的内容，那它会做什么样的变更，为什么又需要这样的变更呢？我们先看看随便几个节点上的 /etc/hosts 内容

```bash
# Kubernetes-managed hosts file (host network).
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.30.155.179 image-registry.openshift-image-registry.svc image-registry.openshift-image-registry.svc.cluster.local # openshift-generated-node-resolver
```

再结合 yaml 里面中间那段 bash ，可以发现这段 bash 是个死循环，主要是查询检测 `SERVICES` 有没有变更，如果有的话就更新到 /etc/hosts 里面去。那这里就有个问题了，为什么要这样设计？为什么不能通过 DNS 去解决这个服务的解析呢？

让我们先看回这个 DS 的 yaml，它插入了三个环境变量，然后把它替换到脚本里面去，现在脚本是这样运行的， 不断的去 `172.30.0.10` 查询 `image-registry.openshift-image-registry.svc` 服务的 v4 和 v6 地址，如果发现变动就会更新 /etc/hosts. 这样有什么意义呢？难道 CoreDNS 自己没办法解析这样的地址吗，让我们试试看

```bash
sh-4.2$ cat /etc/resolv.conf
search test.svc.cluster.local svc.cluster.local cluster.local ap-south-1.compute.internal
nameserver 172.30.0.10
options ndots:5

sh-4.2$ nslookup -debug image-registry.openshift-image-registry.svc
Server:		172.30.0.10
Address:	172.30.0.10#53

------------
    QUESTIONS:
	image-registry.openshift-image-registry.svc.test.svc.cluster.local, type = A, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ->  cluster.local
	origin = ns.dns.cluster.local
	mail addr = hostmaster.cluster.local
	serial = 1639031709
	refresh = 7200
	retry = 1800
	expire = 86400
	minimum = 5
	ttl = 5
    ADDITIONAL RECORDS:
------------
** server can't find image-registry.openshift-image-registry.svc: NXDOMAIN
Server:		172.30.0.10
Address:	172.30.0.10#53

------------
    QUESTIONS:
	image-registry.openshift-image-registry.svc.svc.cluster.local, type = A, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ->  cluster.local
	origin = ns.dns.cluster.local
	mail addr = hostmaster.cluster.local
	serial = 1639031709
	refresh = 7200
	retry = 1800
	expire = 86400
	minimum = 5
	ttl = 5
    ADDITIONAL RECORDS:
------------
** server can't find image-registry.openshift-image-registry.svc: NXDOMAIN
Server:		172.30.0.10
Address:	172.30.0.10#53

------------
    QUESTIONS:
	image-registry.openshift-image-registry.svc.cluster.local, type = A, class = IN
    ANSWERS:
    ->  image-registry.openshift-image-registry.svc.cluster.local
	internet address = 172.30.155.179
	ttl = 5
    AUTHORITY RECORDS:
    ADDITIONAL RECORDS:
------------
Name:	image-registry.openshift-image-registry.svc.cluster.local
Address: 172.30.155.179
------------
    QUESTIONS:
	image-registry.openshift-image-registry.svc.cluster.local, type = AAAA, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ->  cluster.local
	origin = ns.dns.cluster.local
	mail addr = hostmaster.cluster.local
	serial = 1639031709
	refresh = 7200
	retry = 1800
	expire = 86400
	minimum = 5
	ttl = 5
    ADDITIONAL RECORDS:
------------

sh-4.2$
```

上面这个结果是跑在一个 pod 里面的，如果看回 /etc/resolv.conf 的内容，就可以看出来它是去 `172.30.0.10` 去查询 DNS，然后添加不同的后缀去查询，比如接着的 nslookup 命令结果，它其实是先查询 `image-registry.openshift-image-registry.svc.test.svc.cluster.local` 然后再 `image-registry.openshift-image-registry.svc.svc.cluster.local`, 最后是 `image-registry.openshift-image-registry.svc.cluster.local`

那如果节点上又是怎样的呢，因为上面的测试是 pod 里面，然后 pod 会根据 dnspolicy 修改自己的 /etc/resolv.conf，所以才能解析到上面的查询。那节点又怎样呢

```bash
sh-4.4# cat /etc/resolv.conf
search ap-south-1.compute.internal
nameserver 10.0.0.2

sh-4.4# nslookup -debug image-registry.openshift-image-registry.svc
Server:		10.0.0.2
Address:	10.0.0.2#53

------------
    QUESTIONS:
	image-registry.openshift-image-registry.svc, type = A, class = IN
    ANSWERS:
    AUTHORITY RECORDS:
    ->  .
	origin = a.root-servers.net
	mail addr = nstld.verisign-grs.com
	serial = 2021120801
	refresh = 1800
	retry = 900
	expire = 604800
	minimum = 86400
	ttl = 613
    ADDITIONAL RECORDS:
------------
** server can't find image-registry.openshift-image-registry.svc: NXDOMAIN

sh-4.4#
```

答案是节点上解析不了，因为节点上的 `NAMESERVER` 并非集群内部的服务器，并不会记录这些 SVC 的变动，所以需要靠 `/etc/hosts` 来帮忙解析。那为什么只有这么一个 SVC 需要解析呢，应该是因为这是跑在 OCP 里面的镜像服务，如果解析不了那就无法从 OCP 里面拉取镜像了。所以如果有其他的服务也需要在节点上面去连接的话，也需要加入到这个 `SERVICES` 列表里面去，让它一起更新到 `/etc/hosts` 里面.

好了，关于 cluster-dns operator 就拆解到这里，改天再研究 OCP 是怎么样使用这些机制来做到服务发现的呢。

# 参考链接

- [https://github.com/openshift/cluster-dns-operator](https://github.com/openshift/cluster-dns-operator)
- [https://coredns.io/manual/configuration/](https://coredns.io/manual/configuration/)
- [https://access.redhat.com/solutions/3804501](https://access.redhat.com/solutions/3804501)