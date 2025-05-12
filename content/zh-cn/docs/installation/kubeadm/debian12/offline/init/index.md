---
title: "在 debian12 上离线安装 k8s"
linkTitle: "init 集群"
weight: 20
date: 2025-05-11
description: >
  在 debian12 上用 kubeadm 离线安装 k8s
---

## 指定镜像仓库进行离线安装

新建一个 kubeadm-config.yaml 文件：

```bash
vi kubeadm-config.yaml
```

内容设置为：

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.3.179"
  bindPort: 6443
nodeRegistration:
  criSocket: "unix:///var/run/containerd/containerd.sock"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
imageRepository: "192.168.3.91:5000/k8s-proxy"
dns:
  imageRepository: "192.168.3.91:5000/k8s-proxy/coredns"
  imageTag: "v1.12.0"
etcd:
  local:
    imageRepository: "192.168.3.91:5000/k8s-proxy"
    imageTag: "3.5.21-0"
```













使用提前准备好的 harbor 代理仓库 192.168.3.91:5000/k8s-proxy 进行 kubeadm 的 init 操作：

```bash
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock --apiserver-advertise-address=192.168.3.179 --image-repository=192.168.3.91:5000/k8s-proxy
```

但是报错了：

```bash
W0511 21:34:13.908906   60242 version.go:109] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get "https://dl.k8s.io/release/stable-1.txt": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
W0511 21:34:13.908935   60242 version.go:110] falling back to the local client version: v1.33.0
[init] Using Kubernetes version: v1.33.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
W0511 21:34:13.944144   60242 checks.go:846] detected that the sandbox image "registry.k8s.io/pause:3.10" of the container runtime is inconsistent with that used by kubeadm.It is recommended to use "192.168.3.91:5000/k8s-proxy/pause:3.10" as the CRI sandbox image.
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR ImagePull]: failed to pull image 192.168.3.91:5000/k8s-proxy/coredns:v1.12.0: failed to pull image 192.168.3.91:5000/k8s-proxy/coredns:v1.12.0: Error response from daemon: unknown: repository k8s-proxy/coredns not found
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```

不知道为什么 coredns 的镜像路径和其他的不一样。这是 k8s 所有的镜像的路径，可以看到正常的名称是 "registry.k8s.io/xxxx:version"，只有 coredns 是 `registry.k8s.io/coredns/coredns:v1.12.0`, 多一个路径：


```bash
kubeadm config images pull
[config/images] Pulled registry.k8s.io/kube-apiserver:v1.33.0
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.33.0
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.33.0
[config/images] Pulled registry.k8s.io/kube-proxy:v1.33.0
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.12.0
[config/images] Pulled registry.k8s.io/pause:3.10
[config/images] Pulled registry.k8s.io/etcd:3.5.21-0
```

指定镜像仓库再做一次镜像下载，验证一下，发现也是同样报错：

```bash
kubeadm config images pull --image-repository=192.168.3.91:5000/k8s-proxy
[config/images] Pulled 192.168.3.91:5000/k8s-proxy/kube-apiserver:v1.33.0
[config/images] Pulled 192.168.3.91:5000/k8s-proxy/kube-controller-manager:v1.33.0
[config/images] Pulled 192.168.3.91:5000/k8s-proxy/kube-scheduler:v1.33.0
[config/images] Pulled 192.168.3.91:5000/k8s-proxy/kube-proxy:v1.33.0
failed to pull image "192.168.3.91:5000/k8s-proxy/coredns:v1.12.0": failed to pull image 192.168.3.91:5000/k8s-proxy/coredns:v1.12.0: Error response from daemon: unknown: resource not found: repo k8s-proxy/coredns, tag v1.12.0 not found
To see the stack trace of this error execute with --v=5 or higher
```



