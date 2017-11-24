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

或者如果之前设置了hp/sp别名，可以如下操作：

```bash
sudo -i
hp apt-get install bridge-utils
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

## 安装kubeadm

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

通常在`apt-get update`时会遇到问题，因为kubernetes的网站被墙了。

之后会遇到一个尴尬的问题：如果不翻墙，kubernetes等被墙的网站无法访问，update也就没有意义。如果翻墙，则原来设置的国内的镜像源无法访问（我的翻墙工具有这问题）。后来想到版本，将镜像源设置为台湾，再翻墙，就都可以访问了。

```bash
hp apt-get update
hp apt-get install -y kubelet kubeadm kubectl
```

终于把kubeadm安装好了。

## 安装k8s

### 配置代理

kubeadm使用时，还是有个翻墙的问题，尤其由于在ubuntu下，需要sudo，所以必须在sudo之后设置代理。

```bash
sudo -i
export http_proxy=http://192.168.31.152:8123
export HTTP_PROXY=$http_proxy
export https_proxy=$http_proxy
export HTTPS_PROXY=$http_proxy
export no_proxy="localhost,127.0.0.1,localaddress,.localdomain.com,example.com,10.18.17.16,::1,192.168.31.152"

kubeadm init
```

备注:

1. `192.168.31.152`是物理机的网卡IP，注意修改
1. `192.168.31.0/24`和`192.168.31.*`这样的写法在这里无法生效
1. 如果代理设置不正确，会有如下提示：

	```bash
    [preflight] WARNING: Connection to "https://192.168.31.152:6443" uses proxy "http://192.168.31.152:8123". If that is not intended, adjust your proxy settings
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



