---
date: 2018-12-08T09:00:00+08:00
title: Service DNS
menu:
  main:
    parent: "concept-service"
weight: 334
description : "Kubernetes service DNS相关只是"
---

集群中定义的每个服务（包括DNS服务器本身）都会分配一个DNS名称。默认情况下，客户端Pod的DNS搜索列表将包含Pod自己的命名空间和集群的默认域名。

示例：假设在Kubernetes命名空间 bar 中有名为 foo 的服务。在命名空间 bar 中运行的Pod可以通过简单地执行DNS查询 `foo` 来查找此服务。在命名空间 quux 中运行的Pod可以通过执行DNS查询  foo.bar 来查找此服务。

Service的DNS

### A记录

普通服务（非headless）分配有 `my-svc.my-namespace.svc.cluster.local` 形式的域名，这个域名会有一条DNS A记录，解析为服务的cluster IP。

Headless 服务（没有cluster IP）也分配有 `my-svc.my-namespace.svc.cluster.local` 形式的域名，这个域名会有一条DNS A记录。与普通服务不同，将解析为服务所在的pod的IP集合。

```bash
# 列出pod
k get pods --namespace default
k get pod  details-v1-68c7c8666d-q79lz  --namespace default
# 登录进去
k exec -it  details-v1-68c7c8666d-q79lz  --namespace default -- /bin/bash
# 遗憾是这些pod都没有安装 host/nslookup/dig 命令，后面再试
```



### SRV记录

为普通服务或headless服务的命名端口创建SRV记录。

对于每个命名端口，SRV记录的样式为 `_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster.local`。

- 对于常规服务，这将解析为端口号和域名：`my-svc.my-namespace.svc.cluster.local`。
- 对于headless服务，这将解析为多个答案，服务对应的每个pod一个答案，并包含端口号和pod的域名，形式为 `auto-generated-name.my-svc.my-namespace.svc.cluster.local`。


### DNS Policy

### DNS Config

TBD：稍后细看

### 


