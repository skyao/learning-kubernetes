---
title: "在 ubuntu 上安装 kubectl"
linkTitle: "ubuntu"
weight: 10
date: 2021-02-01
description: >
  安装配置 kubectl
---

参考 Kubernetes 官方文档:

- https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

## 分步骤安装

和后面安装 kubeadm 方式一样，只是这里只需要安装 kubectl 一个工具，不需要安装 kubeadm 和 kublete

执行如下命令：

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```

k8s 暂时固定使用 1.23.14 版本：

```bash
sudo apt-get install kubectl=1.23.14-00
# sudo apt-get install kubelet=1.23.14-00 kubeadm=1.23.14-00 kubectl=1.23.14-00
```

## ~~直接安装~~

不推荐这样安装，会安装最新版本，而且安装目录是  `/usr/local/bin/` 。

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

如果 `/usr/local/bin/` 不在 path 路径下，则需要修改一下 path：

```bash
export PATH=/usr/local/bin:$PATH
```

验证一下：

```bash
kubectl version --output=yaml
```

输出为：

```bash
clientVersion:
  buildDate: "2023-06-14T09:53:42Z"
  compiler: gc
  gitCommit: 25b4e43193bcda6c7328a6d147b1fb73a33f1598
  gitTreeState: clean
  gitVersion: v1.27.3
  goVersion: go1.20.5
  major: "1"
  minor: "27"
  platform: linux/amd64
kustomizeVersion: v5.0.1

The connection to the server localhost:8080 was refused - did you specify the right host or port?

```

## 配置

### oh-my-zsh自动完成

在使用 oh-my-zsh 之后，会更加的简单（强烈推荐使用 oh-my-zsh ），只要在 oh-my-zsh 的 plugins 列表中增加 kubectl 即可。

然后，在 `~/.zshrc` 中增加以下内容:

```bash
# k8s auto complete
alias k=kubectl
complete -F __start_kubectl k
```

`source ~/.zshrc `之后即可使用，此时用 k 这个别名来执行 kubectl 命令时也可以实现自动完成，非常的方便。

