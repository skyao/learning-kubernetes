---
title: "安装命令行"
linkTitle: "安装"
weight: 20
date: 2025-03-04
description: >
  在 debian12 上安装 kubeadm / kubelet / kubectl
---

参考： https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

## 安装 kubeadm / kubelet / kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

假定要安装的 kubernetes 版本为 1.33: 

```bash
export K8S_VERSION=1.33

# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

开始安装 kubelet kubeadm kubectl：

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

禁止这三个程序的自动更新：

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

验证安装：

```bash
kubectl version --client && echo && kubeadm version
```

输出为：

```bash
Client Version: v1.33.0
Kustomize Version: v5.6.0

kubeadm version: &version.Info{Major:"1", Minor:"33", EmulationMajor:"", EmulationMinor:"", MinCompatibilityMajor:"", MinCompatibilityMinor:"", GitVersion:"v1.33.0", GitCommit:"60a317eadfcb839692a68eab88b2096f4d708f4f", GitTreeState:"clean", BuildDate:"2025-04-23T13:05:48Z", GoVersion:"go1.24.2", Compiler:"gc", Platform:"linux/amd64"}
```

在运行 kubeadm 之前，先启动 kubelet 服务：

```bash
sudo systemctl enable --now kubelet
```

## 安装后配置

### 优化 zsh

```bash
vi ~/.zshrc
```

增加以下内容:

```bash
# k8s auto complete
alias k=kubectl
complete -F __start_kubectl k
```

执行：

```bash
source ~/.zshrc
```

之后即可使用，此时用 k 这个别名来执行 kubectl 命令时也可以实现自动完成，非常的方便。

### 取消更新

kubeadm / kubelet / kubectl 的版本没有必要升级到最新，因此可以取消他们的自动更新。

```bash
sudo vi /etc/apt/sources.list.d/kubernetes.list
```

注释掉里面的内容。

> 备注：前面执行 apt-mark hold 后已经不会再更新了，但依然会拖慢 apt update 的速度，因此还是需要手动注释。

## 常见问题

### prod-cdn.packages.k8s.io 无法访问

偶然会遇到 prod-cdn.packages.k8s.io 无法访问的问题，此时的报错如下：

```bash
sudo apt-get update
Hit:1 http://mirrors.ustc.edu.cn/debian bookworm InRelease
Hit:2 http://mirrors.ustc.edu.cn/debian bookworm-updates InRelease
Hit:3 http://security.debian.org/debian-security bookworm-security InRelease
Ign:4 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.32/deb  InRelease
Ign:4 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.32/deb  InRelease
Ign:4 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.32/deb  InRelease
Err:4 https://prod-cdn.packages.k8s.io/repositories/isv:/kubernetes:/core:/stable:/v1.32/deb  InRelease
  Could not connect to prod-cdn.packages.k8s.io:443 (221.228.32.13), connection timed out
Reading package lists... Done
W: Failed to fetch https://pkgs.k8s.io/core:/stable:/v1.32/deb/InRelease  Could not connect to prod-cdn.packages.k8s.io:443 (221.228.32.13), connection timed out
W: Some index files failed to download. They have been ignored, or old ones used instead.
```

首先排除是网络问题，因为实际配好网络代理，也依然无法访问。

后来发现，在不同地区的机器上 ping prod-cdn.packages.k8s.io 的 ip 地址是不一样的，

```bash
$ ping prod-cdn.packages.k8s.io

Pinging dkhzw6k7x6ord.cloudfront.net [108.139.10.84] with 32 bytes of data:
Reply from 108.139.10.84: bytes=32 time=164ms TTL=242
Reply from 108.139.10.84: bytes=32 time=166ms TTL=242
......

# 这个地址无法访问
$ ping prod-cdn.packages.k8s.io
PING dkhzw6k7x6ord.cloudfront.net (221.228.32.13) 56(84) bytes of data.
64 bytes from 221.228.32.13 (221.228.32.13): icmp_seq=1 ttl=57 time=9.90 ms
64 bytes from 221.228.32.13 (221.228.32.13): icmp_seq=2 ttl=57 time=11.4 ms
......
```

因此考虑通过修改 /etc/hosts 文件来避开 dns 解析的问题：

```bash
sudo vi /etc/hosts
```

添加如下内容：

```bash
108.139.10.84 prod-cdn.packages.k8s.io
```

这样在出现问题的这台机器上，强制将 prod-cdn.packages.k8s.io 解析到 108.139.10.84 这个 ip 地址，这样就可以访问了。

