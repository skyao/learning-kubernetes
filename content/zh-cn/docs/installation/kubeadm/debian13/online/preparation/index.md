---
title: "准备工作"
linkTitle: "准备"
weight: 10
date: 2025-11-13
description: >
  在 debian13 上安装 kubenetes 之前的准备工作
---

## 系统更新

确保更新debian系统到最新，移除不再需要的软件，清理无用的安装包：

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt autoremove
sudo apt autoclean
```

如果更新了内核，最好重启一下。

## swap 分区

安装 Kubernetes 要求机器不能有 swap 分区。

参考：

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#swap-configuration

## 开启模块

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

## container runtime

Kubernetes 支持多种 container runtime，这里暂时继续使用 docker engine + cri-dockerd。

参考：

https://kubernetes.io/docs/setup/production-environment/container-runtimes/

### 安装 docker

docker 的安装参考：

https://skyao.net/learning-docker/docs/installation/debian13/

最方便的方式就是使用离线安装包进行安装。


### 安装 containerd

TODO：后面考虑换 containerd

## 安装 helm

参考：

https://helm.sh/docs/intro/install/#from-apt-debianubuntu

选择用官方脚本安装 Helm（以便绕过无法访问的 apt 仓库）:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

输出为：

```bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 11929  100 11929    0     0  48738      0 --:--:-- --:--:-- --:--:-- 48889
Downloading https://get.helm.sh/helm-v3.20.2-linux-amd64.tar.gz
Verifying checksum... Done.
Preparing to install helm into /usr/local/bin
helm installed into /usr/local/bin/helm
```

验证版本：

```bash
helm version
```

版本显示为 v3.20.2：

```bash
version.BuildInfo{Version:"v3.20.2", GitCommit:"8fb76d6ab555577e98e23b7500009537a471feee", GitTreeState:"clean", GoVersion:"go1.25.9"}
```