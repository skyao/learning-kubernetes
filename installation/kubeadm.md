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

1. 如果在使用 kubeadm init 命令是 提示以下内容:

    ```bash
    [preflight] Some fatal errors occurred:
        /proc/sys/net/bridge/bridge-nf-call-iptables contents are not set to 1
    ```

    执行:
    ```bash
    echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
    echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
    ```

1. 禁用虚拟内存swap

	如果报错如下：

	```bash
    [preflight] Some fatal errors occurred:
	running with swap on is not supported. Please disable swap
	```

	这说明我们开启了linux的swap虚拟内存,这里要求关闭。

    参考文章： [Ubuntu 16.04 禁用启用虚拟内存swap](http://blog.csdn.net/csdn_duomaomao/article/details/75142769)

    实测不重启电脑的方案无法生效，只好用需要重启的方案了。

    ```bash
    mount -n -o remount,rw /
    vi /etc/fstab
    # 在swap分区这行前加 # 禁用掉，保存退出
    reboot
    # 看看是否生效
    free -m
	```

	实测发现，虽然当时生效了，但是过一段时间，虚拟内存又出现了。解决方式：通过磁盘工具将swap分区删除。

## 安装kubeadm

> 切记： 想办法搞定全局翻墙，不然kubeadm安装是比较麻烦的。

按照[官方文档](https://kubernetes.io/docs/setup/independent/install-kubeadm/)的指示，执行如下命令：

```bash
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
```

## 安装k8s

> 同样： 想办法搞定全局翻墙。

```bash
sudo kubeadm init
```

输出如下：

```bash
[init] Using Kubernetes version: v1.9.2
[init] Using Authorization modes: [Node RBAC]
......
[addons] Applied essential addon: kube-dns
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token d05a21.dfad690e77acc878 192.168.1.244:6443 --discovery-token-ca-cert-hash sha256:263c07847848652711ecbe62b128d4c7e4a24418995a49c78f4ec3753cf111d4
```

为了使用普通用户，按照上面的提示执行：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

此时如果遇到执行命令时报如下错误：

```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"9", GitVersion:"v1.9.2", GitCommit:"5fa2db2bd46ac79e5e00a4e6ed24191080aa463b", GitTreeState:"clean", BuildDate:"2018-01-18T10:09:24Z", GoVersion:"go1.9.2", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

需要修改`/etc/kubernetes/manifests/kube-apiserver.yaml`文件，修改下列参数：

```bash
- --insecure-port=8080
```

默认是0不打开，修改为8080即可。

