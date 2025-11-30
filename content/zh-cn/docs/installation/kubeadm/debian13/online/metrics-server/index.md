---
title: "安装 metrics server"
linkTitle: "metrics server"
weight: 50
date: 2025-03-04
description: >
  安装 kubernetes 的 metrics server
---

参考：https://github.com/kubernetes-sigs/metrics-server/#installation

## 安装 metrics server

下载：

```bash
mkdir -p ~/work/soft/k8s
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

metrics-server-7c9977449d-h4psq    1/1     Running   0          34s
```

验证一下，查看 service 信息

```bash
$ kubectl describe svc metrics-server -n kube-system

Name:                     metrics-server
Namespace:                kube-system
Labels:                   k8s-app=metrics-server
Annotations:              <none>
Selector:                 k8s-app=metrics-server
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.97.226.236
IPs:                      10.97.226.236
Port:                     https  443/TCP
TargetPort:               https/TCP
Endpoints:                10.244.0.9:10250
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```

简单验证一下基本使用:

```bash
kubectl top nodes
kubectl top pods -n kube-system 
```

正常能看到类似如下的输出：

```bash
$ kubectl top nodes
NAME       CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)   
debian13   161m         4%       1040Mi          13%   

$ kubectl top pods -n kube-system 
NAME                               CPU(cores)   MEMORY(bytes)   
coredns-848fbff4f8-2lx6w           1m           15Mi            
coredns-848fbff4f8-lgr6d           1m           16Mi            
etcd-debian13                      7m           47Mi            
kube-apiserver-debian13            13m          241Mi           
kube-controller-manager-debian13   6m           53Mi            
kube-proxy-xc4mn                   1m           17Mi            
kube-scheduler-debian13            3m           23Mi            
metrics-server-7c9977449d-h4psq    1m           18Mi
```

如果出现下面的错误：

```bash
error: Metrics API not available
```

可以稍等片刻，等 metrics-server 启动后，再尝试查看。

## 参考资料

- [Debian 12 安装 Kubernetes(k8s) + docker ，配置主从节点](https://acytoo.com/ladder/debian12-kubernetes-installation-and-config-master-worker/)

- [Install Kubernetes Cluster on Debian 12 (Bookworm) servers](https://computingforgeeks.com/install-kubernetes-cluster-on-debian-12-bookworm/)
