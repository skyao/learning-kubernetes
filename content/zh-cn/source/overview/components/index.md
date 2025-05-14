---
title: "组件"
linkTitle: "组件"
weight: 10
date: 2025-05-14
description: >
  k8s 的组件
---

## 有哪些组件？

如何得知 k8s 有哪些组件？ 

### 看 k8s 进程

在 k8s 的 master 上，用 ps 命令查看 k8s 相关的进程：

```bash
ps -ef | grep -v kworker | grep -v nfs | grep -v nginx | grep -v pause | grep -v containerd | grep -v smbd | grep -v cpuhd | grep -v sshd  | grep -v /usr/bin | grep -v /usr/sbin | grep -v scsi | grep -v /sbin
```

排除和k8s无关的进程，能看到：

```bash
root        1388    1299  0 May13 ?        00:04:25 kube-scheduler --authentication-kubeconfig=/etc/kubernetes/scheduler.conf --authorization-kubeconfig=/etc/kubernetes/scheduler.conf --bind-address=127.0.0.1 --kubeconfig=/etc/kubernetes/scheduler.conf --leader-elect=true
root        1398    1297  0 May13 ?        00:17:47 kube-apiserver --advertise-address=192.168.3.112 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
root        1399    1295  0 May13 ?        00:08:55 etcd --advertise-client-urls=https://192.168.3.112:2379 --cert-file=/etc/kubernetes/pki/etcd/server.crt --client-cert-auth=true --data-dir=/var/lib/etcd --experimental-initial-corrupt-check=true --experimental-watch-progress-notify-interval=5s --initial-advertise-peer-urls=https://192.168.3.112:2380 --initial-cluster=k8s112=https://192.168.3.112:2380 --key-file=/etc/kubernetes/pki/etcd/server.key --listen-client-urls=https://127.0.0.1:2379,https://192.168.3.112:2379 --listen-metrics-urls=http://127.0.0.1:2381 --listen-peer-urls=https://192.168.3.112:2380 --name=k8s112 --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt --peer-client-cert-auth=true --peer-key-file=/etc/kubernetes/pki/etcd/peer.key --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt --snapshot-count=10000 --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
root        1401    1293  0 May13 ?        00:05:33 kube-controller-manager --allocate-node-cidrs=true --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf --bind-address=127.0.0.1 --client-ca-file=/etc/kubernetes/pki/ca.crt --cluster-cidr=10.244.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt --cluster-signing-key-file=/etc/kubernetes/pki/ca.key --controllers=*,bootstrapsigner,tokencleaner --kubeconfig=/etc/kubernetes/controller-manager.conf --leader-elect=true --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --root-ca-file=/etc/kubernetes/pki/ca.crt --service-account-private-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --use-service-account-credentials=true
root        3114    2988  0 May13 ?        00:00:07 /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/config.conf --hostname-override=k8s112
root        5283    5118  0 May13 ?        00:00:39 /opt/bin/flanneld --ip-masq --kube-subnet-mgr
1001        8052    7828  0 May13 ?        00:00:06 /dashboard-auth
65532       8054    7829  0 May13 ?        00:00:51 /coredns -conf /etc/coredns/Corefile
1001        8069    7948  0 May13 ?        00:00:06 /dashboard-api --insecure-bind-address=0.0.0.0 --bind-address=0.0.0.0 --namespace=kubernetes-dashboard --metrics-scraper-service-name=kubernetes-dashboard-metrics-scraper
1001        8070    7940  0 May13 ?        00:00:05 /dashboard-metrics-scraper
sky         8072    7968  0 May13 ?        00:01:29 /metrics-server --cert-dir=/tmp --secure-port=10250 --kubelet-preferred-address-types=InternalIP --kubelet-insecure-tls --kubelet-use-node-status-port --metric-resolution=15s
1001        8073    7860  0 May13 ?        00:00:05 /dashboard-web --insecure-bind-address=0.0.0.0 --bind-address=0.0.0.0 --namespace=kubernetes-dashboard --settings-config-map-name=kubernetes-dashboard-web-settings
65532       8083    7937  0 May13 ?        00:00:53 /coredns -conf /etc/coredns/Corefile

```

这里面直接和k8s相关的进程有：

- coredns
- etcd
- kube-controller-manager
- kube-proxy
- kube-apiserver
- kube-scheduler

k8s 扩展之一的 dashboard:

- dashboard-api
- dashboard-metrics-scraper
- dashboard-web
- dashboard-auth

k8s 扩展之一的 metrics-server:

- metrics-server

### 看 k8s build 的产物

clone k8s 的源码：

https://github.com/kubernetes/kubernetes/

然后 build 一下：

```bash
git clone https://github.com/kubernetes/kubernetes
cd kubernetes
make
```

可以看到构建的日志：

```bash
+++ [0514 21:34:35] Building go targets for linux/amd64
    github.com/onsi/ginkgo/v2/ginkgo (non-static)
    k8s.io/apiextensions-apiserver (static)
    k8s.io/component-base/logs/kube-log-runner (static)
    k8s.io/kube-aggregator (static)
    k8s.io/kubernetes/cluster/gce/gci/mounter (static)
    k8s.io/kubernetes/cmd/kubeadm (static)
    k8s.io/kubernetes/cmd/kube-apiserver (static)
    k8s.io/kubernetes/cmd/kube-controller-manager (static)
    k8s.io/kubernetes/cmd/kubectl (static)
    k8s.io/kubernetes/cmd/kubectl-convert (static)
    k8s.io/kubernetes/cmd/kubelet (non-static)
    k8s.io/kubernetes/cmd/kubemark (static)
    k8s.io/kubernetes/cmd/kube-proxy (static)
    k8s.io/kubernetes/cmd/kube-scheduler (static)
    k8s.io/kubernetes/test/conformance/image/go-runner (non-static)
    k8s.io/kubernetes/test/e2e/e2e.test (test)
    k8s.io/kubernetes/test/e2e_node/e2e_node.test (test)
```

构建的产物在 `_output/bin` 目录下，可以看到：

```bash
$ pwd
/home/sky/work/code/kubernetes/kubernetes/_output/bin

$ ls -la
total 1069404
drwxrwxr-x 2 sky sky      4096 May 14 21:34 .
drwxrwxr-x 3 sky sky      4096 May 14 20:50 ..
-rwxr-xr-x 1 sky sky  75759800 May 14 21:34 apiextensions-apiserver
-rwxr-xr-x 1 sky sky 117823416 May 14 21:34 e2e_node.test
-rwxr-xr-x 1 sky sky 126209360 May 14 21:34 e2e.test
-rwxr-xr-x 1 sky sky  10580260 May 14 21:34 ginkgo
-rwxr-xr-x 1 sky sky   2056484 May 14 21:34 go-runner
-rwxr-xr-x 1 sky sky  74133688 May 14 21:34 kubeadm
-rwxr-xr-x 1 sky sky  73158840 May 14 21:34 kube-aggregator
-rwxr-xr-x 1 sky sky  98341048 May 14 21:34 kube-apiserver
-rwxr-xr-x 1 sky sky  91111608 May 14 21:34 kube-controller-manager
-rwxr-xr-x 1 sky sky  59900088 May 14 21:34 kubectl
-rwxr-xr-x 1 sky sky  58917048 May 14 21:34 kubectl-convert
-rwxr-xr-x 1 sky sky  82080036 May 14 21:34 kubelet
-rwxr-xr-x 1 sky sky   1839288 May 14 21:34 kube-log-runner
-rwxr-xr-x 1 sky sky  80322744 May 14 21:34 kubemark
-rwxr-xr-x 1 sky sky  70992056 May 14 21:34 kube-proxy
-rwxr-xr-x 1 sky sky  70004920 May 14 21:34 kube-scheduler
-rwxr-xr-x 1 sky sky   1757368 May 14 21:34 mounter
```

可以看到，k8s 的组件有：

- apiextensions-apiserver
- kubeadm
- kube-aggregator
- kube-apiserver
- kube-controller-manager
- kubectl
- kubectl-convert
- kubelet
- kube-log-runner
- kubemark
- kube-proxy
- kube-scheduler




