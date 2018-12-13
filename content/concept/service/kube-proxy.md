---
date: 2018-12-08T09:00:00+08:00
title: Kube Proxy
menu:
  main:
    parent: "concept-service"
weight: 312
description : "Kubernetes pod的生命周期"
---

Kubernetes集群中的每个节点上都运行一个`kube-proxy`。`kube-proxy`负责实现为除 `ExternalName` 类型外的 `Services` 实现虚拟IP形式。

历史发展信息：

- 在Kubernetes v1.0中，`Services`是一个“4层”（TCP / UDP over IP）构造，代理纯粹工作在用户空间。
- 在Kubernetes v1.1中，`Ingress` API被添加（beta）以表示“7层”（HTTP）服务，也增加了iptables代理，并成为自Kubernetes v1.2以来的默认操作模式。
- 在Kubernetes v1.8.0-beta.0中，添加了ipvs代理。

### 代理模式：用户空间

在这种模式下，kube-proxy观察Kubernetes master 获悉Service和Endpoints对象的添加和删除。对于每个Service，它在本地节点上打开一个端口（随机选择）。与此“代理端口”的任何连接都将被代理到Service后端Pods其中的一个（称为 Endpoints）。使用哪个后端 Pod 是根据 Service 的 SessionAffinity来决定的。最后，它安装iptables规则，捕获发送到 Service 的 clusterIP（这是虚拟的）和端口的流量，并重定向流量到代理服务器端口，这个端口代理到后端Pod。默认情况下，轮询选择后端。
> For each `Service` it opens a port (randomly chosen) on the local node.
>
> 这句的理解，似乎是每个服务都要开一个随机端口，如果有很多服务岂不是要开很多端口？但看到也有说只看一个端口然后所有服务都转发到这个端口。待确认。

![](https://d33wubrfki0l68.cloudfront.net/e351b830334b8622a700a8da6568cb081c464a9b/13020/images/docs/services-userspace-overview.svg)

从图上看，流量是被iptables 转到 kube-proxy，然后 kube-proxy 真的是做了一次proxy。流量真的要转到userspace进kube-proxy，在kube-proxy中进行判断和转发。难怪说效率不高。

好在，userspace 只是旧版本支持的模式，以后可能会放弃维护和支持。

###代理模式：iptables

iptables 代理模式完全由 iptables 来实现 service，是目前默认的方式，也是推荐的方式，效率很高（只有内核中 netfilter 一些损耗）。

在 iptables 代理模式下，kube-proxy观察Kubernetes master 获悉Service和Endpoints对象的添加和删除。对于每一个`Service`，它安装 iptables 规则，捕获发送到 `Service` （虚拟的）的 `clusterIP` 和 `Port` 的流量，重定向流量到 `Service` 的后端集合中的一个。对于每个 `Endpoints` 对象，它会安装 iptables 规则来选择后端的`Pod`。默认情况下，后端的选择是随机的。

显然，iptables不需要在用户空间和内核空间之间切换，它应该比用户空间代理更快，更可靠。但是，与用户空间代理器不同，如果 iptables 代理器最初选择的那个 Pod 没有响应，则它不能自动重试其他的，因此它依赖于就绪探测器。

![](https://d33wubrfki0l68.cloudfront.net/27b2978647a8d7bdc2a96b213f0c0d3242ef9ce0/e8c9b/images/docs/services-iptables-overview.svg)

如图所示，在流量被转发的过程中，流量并没有经过kube-proxy，而这是被 iptables 规则操作，然后被转发，完全在内核空间完成。而kube proxy的工作仅仅是从 API server 获知信息然后设置对应的 iptables 规则。

#### kube-proxy 参数介绍

kube-proxy 的功能相对简单一些，也比较独立，需要的配置并不是很多，比较常用的启动参数包括：

| 参数                     | 含义                                                         | 默认值    |
| ------------------------ | ------------------------------------------------------------ | --------- |
| –alsologtostderr         | 打印日志到标准输出                                           | false     |
| –bind-address            | HTTP 监听地址                                                | 0.0.0.0   |
| –cleanup-iptables        | 如果设置为 true，会清理 proxy 设置的 iptables 选项并退出     | false     |
| –healthz-bind-address    | 健康检查 HTTP API 监听端口                                   | 127.0.0.1 |
| –healthz-port            | 健康检查端口                                                 | 10249     |
| –iptables-masquerade-bit | 使用 iptables 进行 SNAT 的掩码长度                           | 14        |
| –iptables-sync-period    | iptables 更新频率                                            | 30s       |
| –kubeconfig              | kubeconfig 配置文件地址                                      |           |
| –log-dir                 | 日志文件目录/路径                                            |           |
| –masquerade-all          | 如果使用 iptables 模式，对所有流量进行 SNAT 处理             | false     |
| –master                  | kubernetes master API Server 地址                            |           |
| –proxy-mode              | 代理模式，`userspace` 或者 `iptables`， 目前默认是 `iptables`，如果系统或者 iptables 版本不够新，会 fallback 到 userspace 模式 | iptables  |
| –proxy-port-range        | 代理使用的端口范围， 格式为 `beginPort-endPort`，如果没有指定，会随机选择 | `0-0`     |
| –udp-timeout             | UDP 空连接 timeout 时间，只对 `userspace` 模式有用           | 250ms     |
| –v                       | 日志级别                                                     | 0         |

`kube-proxy` 的工作模式可以通过 `--proxy-mode` 进行配置，可以选择 `userspace` 或者 `iptables`。

### 代理模式：ipvs



### 参考资料

- [Kubernetes如何利用iptables](http://www.dbsnake.net/how-kubernetes-use-iptables.html)
- [kubernetes 简介：service 和 kube-proxy 原理](https://cizixs.com/2017/03/30/kubernetes-introduction-service-and-kube-proxy/): 这个讲的比较容易懂