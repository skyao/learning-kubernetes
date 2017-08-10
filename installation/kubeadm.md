# 使用kubeadm安装

参考 Kubernetes 官方文档:

- [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

## 前期准备

### docker和bridge-utils

要求节点上安装有 docker version 1.2+ 和 bridge-utils (用来操作linux bridge).

查看 docker 版本：

```bash
$ docker --version
Docker version 17.06.0-ce, build 02c1d87
```

bridge-utils可以通过apt安装：

```bash
sudo apt-get install bridge-utils
```
