---
title: kube-proxy 与 service 之间的那些事
date: 2019-11-29 17:58:54
tags:
---

在 k8s 中，service 是一个和重要的概念。service 为一组相同的 pod 提供了统一的入口，并且能达到负载均衡的效果。

service 具有这样的功能，正是 kube-proxy 的功劳。当我们暴露一个 service 的时候，kube-proxy 会在 iptables 中追加一些规则，为我们实现了路由与负载均衡的功能。

下面用一个具体的小例子来看看 service 是怎么路由的。

### KUBE-SERVICES 链

在我搭建的一个 minikube 环境中，运行 `iptables -t nat -nvL` 命令查看 iptables 中的内容。

首先，看下 PREROUTING 的规则

`
Chain PREROUTING (policy ACCEPT 5 packets, 271 bytes)
 pkts bytes target     prot opt in     out     source               destination
 130K 6355K KUBE-SERVICES  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service portals */
 121K 5633K DOCKER     all  --  *      *       0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL
`

可以看出，所有请求都会先通过 k8s 追加的 KUBE-SERVICES 链，我们再看一下 KUBE-SERVICES 链中有什么规则

```
Chain KUBE-SERVICES (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SVC-NPX46M4PTMTKRN6Y  tcp  --  *      *       0.0.0.0/0            10.96.0.1            /* default/kubernetes:https cluster IP */ tcp dpt:443
    4   422 KUBE-SVC-TCOU7JCQXEZGVUNU  udp  --  *      *       0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns cluster IP */ udp dpt:53
    0     0 KUBE-SVC-ERIFXISQEP7F7OF4  tcp  --  *      *       0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:dns-tcp cluster IP */ tcp dpt:53
    0     0 KUBE-SVC-JD5MR3NA4I4DYORP  tcp  --  *      *       0.0.0.0/0            10.96.0.10           /* kube-system/kube-dns:metrics cluster IP */ tcp dpt:9153
    0     0 KUBE-SVC-MW5CXKKSAWLC5IQG  tcp  --  *      *       0.0.0.0/0            10.102.64.24         /* default/device: cluster IP */ tcp dpt:8080
    0     0 KUBE-SVC-LKM3CLHDXK6GWQVY  tcp  --  *      *       0.0.0.0/0            10.100.159.228       /* default/traffic: cluster IP */ tcp dpt:8080
    0     0 KUBE-SVC-2Q6H6PHDBHDVLI7U  tcp  --  *      *       0.0.0.0/0            10.99.126.231        /* default/gateway: cluster IP */ tcp dpt:8080
   53  2720 KUBE-NODEPORTS  all  --  *      *       0.0.0.0/0            0.0.0.0/0            /* kubernetes service nodeports; NOTE: this must be the last rule in this chain */ ADDRTYPE match dst-type LOCAL

```

可以看到，有一系列目标为 KUBE-SVC-xxx 链的规则，每条规则都会匹配某个目标 ip 与端口。也就是说访问某个 ip:port 的请求会由 KUBE-SVC-xxx 链来处理。

这个目标 IP ，其实就是我 minikube 里面发布的 service ip，用 kubectl get svc -o wide 查看一下 demo 中的服务。

```
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE     SELECTOR
device    NodePort    10.102.64.24     <none>        8080:31748/TCP   5d19h   app=device
gateway   ClusterIP   10.99.126.231    <none>        8080/TCP         2d1h    app=gateway
traffic   ClusterIP   10.100.159.228   <none>        8080/TCP         2d19h   app=traffic
kubernetes       ClusterIP   10.96.0.1        <none>        443/TCP          7d7h    <none>
```

比如 gateway 服务的 CLUSTER-IP 是 10.99.126.231，端口 8080。那么按照上面的 KUBE-SERVICES 链的规则，target 是 KUBE-SVC-2Q6H6PHDBHDVLI7U。我们再看一下 KUBE-SVC-2Q6H6PHDBHDVLI7U 链的内容

```
Chain KUBE-SVC-2Q6H6PHDBHDVLI7U (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SEP-2S4ZDBRDPHRMTYD5  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

继续看下 KUBE-SEP-2S4ZDBRDPHRMTYD5 链的内容

```
Chain KUBE-SEP-2S4ZDBRDPHRMTYD5 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.17.0.8           0.0.0.0/0
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:172.17.0.8:8080
```

会发现这条链做了一次目标网络地址转换 DNAT，将原来的的目标网络地址 service 的 CLUSTER-IP 10.99.126.231 端口 8080 转换成了 172.17.0.8:8080。此时，我们用 `kubectl get pod -o wide` 命令看一下 demo 中 pod 的 IP 地址

```
NAME                               READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
device-5b6d58dc76-ddtsm     1/1     Running   0          8h      172.17.0.10   minikube   <none>           <none>
device-5b6d58dc76-dk4kq     1/1     Running   0          2d23h   172.17.0.5    minikube   <none>           <none>
gateway-594c8d6cb5-25mc6    1/1     Running   7          2d5h    172.17.0.8    minikube   <none>           <none>
traffic-68cbcb88b9-qqrtk    1/1     Running   0          3d      172.17.0.9    minikube   <none>           <none>
```

172.17.0.8 就是我们 gateway service 的 endpoint，也就是我们想要调用的 pod。这就是为什么我们调用 service 的 CLUSTER-IP 和端口能访问到 pod 的原因了。

### KUBE-SEP-xxx 链

上面的 KUBE-SEP-xxx 链其实就是 service 的 endpoint，那么 service 怎么实现负载均衡的呢。道理其实很简单，一个 service 的链对应多个 endpoint 即可，然后按照一定的比例随机调动。我在 demo 中部署了两个 device 服务的 endpoint，我们我看 kube-proxy 为我们生成的规则


```
Chain KUBE-SVC-MW5CXKKSAWLC5IQG (2 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-SEP-B7QXYLUMAA6RGM2S  all  --  *      *       0.0.0.0/0            0.0.0.0/0            statistic mode random probability 0.50000000000
    0     0 KUBE-SEP-EZG7Q3GTCBMWXIRX  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```

50% 的概率会到 KUBE-SEP-B7QXYLUMAA6RGM2S，50% 概率到 KUBE-SEP-EZG7Q3GTCBMWXIRX。这两个目标正是 device 的两个 pod，如下所示

```
Chain KUBE-SEP-B7QXYLUMAA6RGM2S (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.17.0.10          0.0.0.0/0
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:172.17.0.10:8080
```

```
Chain KUBE-SEP-EZG7Q3GTCBMWXIRX (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  all  --  *      *       172.17.0.5           0.0.0.0/0
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp to:172.17.0.5:8080
```

### KUBE-NODEPORTS 链

回头再看看我们最初入口处的 KUBE-SERVICES，首先会判断是否匹配到了某个服务 KUBE-SVC-xxx。如果没有匹配到，最后还有 KUBE-NODEPORTS 链，匹配目标地址类型属于主机系统的本地网络地址的数据包。我们看下 demo 中 KUBE-NODEPORTS 链的内容

```
Chain KUBE-NODEPORTS (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 KUBE-MARK-MASQ  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/device: */ tcp dpt:31748
    0     0 KUBE-SVC-MW5CXKKSAWLC5IQG  tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            /* default/device: */ tcp dpt:31748
```

会发现只要是目标端口为 31748 的请求都会交给  KUBE-SVC-MW5CXKKSAWLC5IQG 链来处理，这条链就是我们 device service 。其实 31748 这个端口，就是我将 device 服务以 NodePort 类型发布随机分配的端口。看到这里，大家应该明白了为什么通过 NODEPORT 暴露的服务能在集群外部访问了。


集群外部：通过节点 IP 加 nodePort 可以访问到 device service
集群内部：通过 device service 的 CLUSTER-IP 和 port 能访问到 device service

### 总结

1. kube-proxy 会为我们暴露的 service 追加 iptables 规则
2. 所有进出请求都会经过 KUBE-SERVICES 链
3. KUBE-SERVICES 链有我们发布的每一个服务，每个服务的链会对应到 DNAT 到 service 的 endpoint
4. KUBE-SERVICES 链最后的是 KUBE-NODEPORTS 链，会匹配请求本主机的一些端口，这就是我们通过 NODEPORT 类型发布服务能在集群外部访问的原因

### 延伸资料

[iptables 入门](https://www.zsythink.net/archives/1199)
[iptables 命令，规则介绍](https://www.cnblogs.com/zclzhao/p/5081590.html)
[kube-proxy 转发规则](https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/03/27/Kubernetes-kube-proxy.html)
