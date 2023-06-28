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

## 安装

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

