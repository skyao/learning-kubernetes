---
title: "安装 metrics server"
linkTitle: "metrics server"
weight: 50
date: 2025-03-04
description: >
  安装 kubernetes 的 metrics server
---



## 安装 metrics server

下载：

```bash
cd ~/work/soft/k8s
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

修改下载下来的 components.yaml， 增加 `--kubelet-insecure-tls` 并修改 `--kubelet-preferred-address-types`：

```yaml
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP   # 修改这行，默认是InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls  # 增加这行
```

然后安装：

```bash
k apply -f components.yaml
```

稍等片刻看是否启动:

```bash
$ kubectl get pod -n kube-system | grep metrics-server
```

验证一下，查看 service 信息

```bash
kubectl describe svc metrics-server -n kube-system
```

简单验证一下基本使用:

```bash
kubectl top nodes
kubectl top pods -n kube-system 
```





## 参考资料



- [Debian 12 安装 Kubernetes(k8s) + docker ，配置主从节点](https://acytoo.com/ladder/debian12-kubernetes-installation-and-config-master-worker/)

- [Install Kubernetes Cluster on Debian 12 (Bookworm) servers](https://computingforgeeks.com/install-kubernetes-cluster-on-debian-12-bookworm/)
