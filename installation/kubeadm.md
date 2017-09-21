# 使用kubeadm安装

参考 Kubernetes 官方文档:

- [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

## 前期准备

### 关闭防火墙
```bash
systemctl disable firewalld && systemctl stop firewalld
```

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

### 注意事项
1.在国内是无法访问google yum 源,需要添加 https 代理

```bash
 export https_proxy="http://xxx.xx.xx.xx:xx"
```
在安装完成后需要关闭代理

```bash
 unset https_proxy
```

2.如果在使用 kubeadm init 命令是 提示以下内容:

```bash
[preflight] Some fatal errors occurred:
	/proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1
```

执行:
```bash
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
```
