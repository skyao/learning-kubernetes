---
title: "部署 metrics-server"
linkTitle: "metrics-server"
weight: 2170
date: 2021-12-28
description: >
  metrics-server
---


## 安装 metrics-server

通过 kubeadm 安装的 k8s 集群默认是没有安装 metrics-server，因此需要手工安装。

> 注意：不要按照官方文档所说的那样直接安装，会不可用的。

### 修改 api server

先检查 k8s 集群的 api server 是否有启用API Aggregator:

```bash
ps -ef | grep apiserver 
```

对比：

```bash
ps -ef | grep apiserver | grep enable-aggregator-routing
```

默认是没有开启的。因此需要修改 k8s apiserver 的配置文件：

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

增加 `--enable-aggregator-routing=true`

```yaml
apiVersion: v1
kind: Pod
......
spec:
  containers:
  - command:
    - kube-apiserver
	......
    - --enable-bootstrap-token-auth=true
    - --enable-aggregator-routing=true  # 增加这行
```

api server 会自动重启，稍后用命令验证一下：

```bash
ps -ef | grep apiserver | grep enable-aggregator-routing
```

### 下载并修改安装文件

先下载安装文件，直接用最新版本：

```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

如果要安装指定版本，请查看 https://github.com/kubernetes-sigs/metrics-server/releases/ 页面。

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
$ k apply -f components.yaml

serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

稍等片刻看是否启动:

```bash
$ kubectl get pod -n kube-system | grep metrics-server
metrics-server-5979f785c8-lmtq5     1/1     Running   0                46s
```

验证一下，查看 service 信息

```bash
$ kubectl describe svc metrics-server -n kube-system

Name:              metrics-server
Namespace:         kube-system
Labels:            k8s-app=metrics-server
Annotations:       <none>
Selector:          k8s-app=metrics-server
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.98.127.10
IPs:               10.98.127.10
Port:              https  443/TCP
TargetPort:        https/TCP
Endpoints:         10.244.0.37:4443		# ping 一下这个 IP 地址
Session Affinity:  None
Events:            <none>
```

## 使用

简单验证一下基本使用。

```bash
$ kubectl top nodes
NAME        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
skyserver   384m         1%     1687Mi          1% 

$ kubectl top pods -n kube-system 
NAME                                CPU(cores)   MEMORY(bytes)   
coredns-64897985d-9z82d             2m           19Mi            
coredns-64897985d-wkzc7             2m           20Mi            
etcd-skyserver                      23m          77Mi            
kube-apiserver-skyserver            74m          282Mi           
kube-controller-manager-skyserver   24m          58Mi            
kube-flannel-ds-lnl72               4m           39Mi            
kube-proxy-8g26s                    1m           37Mi            
kube-scheduler-skyserver            5m           23Mi            
metrics-server-5979f785c8-lmtq5     4m           21Mi 
```



