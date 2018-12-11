---
date: 2018-12-06T08:00:00+08:00
title: kubectl
weight: 610
description : "kubectl介绍"
---


kubectl是官方提供的客户端工具，可直接以命令行的方式同集群交互。

## 资料

- 官方文档
	- [Overview of kubectl](https://kubernetes.io/docs/user-guide/kubectl-overview/)
	- [kubectl](https://kubernetes.io/docs/user-guide/kubectl/)
	- [v1.8 Commands](https://kubernetes.io/docs/user-guide/kubectl/v1.8/)
	- [v1.7 Commands](https://kubernetes.io/docs/user-guide/kubectl/v1.7/)
	- [Kubectl Cheatsheet](https://kubernetes.io/docs/user-guide/kubectl-cheatsheet/)
- [Kubectl Cheatsheet 中文版本](https://www.tuicool.com/articles/qm2A3qJ)

## 用法

### 自动补全

执行下列命令，可以在bash下启动kubectl的自动命令补全，使用时特别的方便，强烈推荐：

```bash
source <(kubectl completion bash)
```

方便起见，可以直接修改`/etc/profile`，加入sourcek8s别名。

```bash
alias sourcek8s='source <(kubectl completion bash)'
```



## 备忘



```bash
$ kubectl cluster-info
Kubernetes master is running at https://192.168.0.10:6443
KubeDNS is running at https://192.168.0.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```bash
$ kubectl get services kubernetes-dashboard -n kube-system
$ kubectl get pods --namespace=kube-system
kubectl describe po kubernetes-dashboard-79ff88449c-hj72p --namespace kube-system
$ kubectl get nodes
$ kubectl get pods --all-namespaces
```