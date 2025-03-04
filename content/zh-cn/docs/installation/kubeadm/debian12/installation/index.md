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

假定要安装的 kubernetes 版本为 1.32: 

```bash
export K8S_VERSION=1.32

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
Client Version: v1.32.2
Kustomize Version: v5.5.0

kubeadm version: &version.Info{Major:"1", Minor:"32", GitVersion:"v1.32.2", GitCommit:"67a30c0adcf52bd3f56ff0893ce19966be12991f", GitTreeState:"clean", BuildDate:"2025-02-12T21:24:52Z", GoVersion:"go1.23.6", Compiler:"gc", Platform:"linux/amd64"}
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

