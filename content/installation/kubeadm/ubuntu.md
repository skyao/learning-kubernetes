---
date: 2018-12-06T08:00:00+08:00
title: ubuntu下用kubeadm安装
menu:
  main:
    parent: "installation-kubeadm"
weight: 121
description : "ubuntu下用kubeadm安装kubernetes"
---

参考 Kubernetes 官方文档:

- [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
- https://my.oschina.net/u/2306127/blog/1922542
- https://linuxconfig.org/how-to-install-kubernetes-on-ubuntu-18-04-bionic-beaver-linux
- https://kubernetes.io/docs/setup/independent/install-kubeadm/

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
Docker version 18.09.1, build 4c52b90
```

bridge-utils可以通过apt安装：

```bash
sudo apt-get install bridge-utils
```

### 设置 systemd 为 Docker cgroup driver

Docker cgroup driver 默认为 cgroupfs， kubeadm init 时会给出警告：


```
[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
```

修改`/etc/docker/daemon.json`文件

```
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

然后重启docker：

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```

参考： https://blog.csdn.net/Andriy_dangli/article/details/85062983

有建议是说要给 kubeadm 设置配置文件，但我操作下来只要修改玩docker配置就可以了，以下内容仅作为参考：

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#configure-cgroup-driver-used-by-kubelet-on-control-plane-node

新建 /var/lib/kubelet/config.yaml 文件，内容为：

```bash
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
```

### 其他注意事项

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

按照 [官方文档](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) 的指示，执行如下命令：

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

这会安装最新版本的kubernetes，如果希望按照特定版本：

```bash
apt-get install kubelet=1.12.5-00 kubeadm=1.12.5-00 kubectl=1.12.5-00
```

具体有哪些可用的版本，可以看这里：https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-arm64/Packages

## 安装k8s

> 同样： 想办法搞定全局翻墙。

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

注意后面为了使用 CNI network 和 Flannel，我们在这里设置了 `--pod-network-cidr=10.244.0.0/16`，如果不加这个设置，Flannel 会一直报错。

输出如下：

```bash
[init] Using Kubernetes version: v1.20.2
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.2. Latest validated version: 19.03
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'

......
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.0.10:6443 --token cpjwcw.nl439516resja838 \
    --discovery-token-ca-cert-hash sha256:582fee685a01cadcf5f33fcfd46d1dc9bec21e4a607de0c18ce4ded978df833d 
```

为了使用普通用户，按照上面的提示执行：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

安装完成后，node处于NotReady状态：

```bash
$ kubectl get node
NAME      STATUS     ROLES    AGE     VERSION
skywork   NotReady   master   6m12s   v1.13.2
```

kubectl describe 可以看到是因为没有安装 network plugin

```bash
kubectl describe node skywork
Name:               skywork
Roles:              master
......
  Ready            False   Mon, 01 Feb 2021 10:36:34 +0800   Mon, 01 Feb 2021 10:36:22 +0800   KubeletNotReady              runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

安装flannel：

https://github.com/coreos/flannel/blob/master/Documentation/kube-flannel.yml

```bash
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

稍等就可以看到 node 的状态变为 Ready了 ()：

```bash
$ kubectl get node
NAME      STATUS   ROLES    AGE   VERSION
skywork   Ready    master   42m   v1.13.2
```

如果一直看不到Readky，可以重启 kubelet 试试(我在ubuntu 20.04 + k8s 1.20上遇到这个问题)

```bash
systemctl restart kubelet
```

node ready 之后，可以看到：

```bash
$ kubectl get pods -A

NAMESPACE     NAME                              READY   STATUS    RESTARTS   AGE
kube-system   coredns-74ff55c5b-8jvml           1/1     Running   0          19m
kube-system   coredns-74ff55c5b-9q6pf           1/1     Running   0          19m
kube-system   etcd-skywork                      1/1     Running   0          19m
kube-system   kube-apiserver-skywork            1/1     Running   0          19m
kube-system   kube-controller-manager-skywork   1/1     Running   0          19m
kube-system   kube-flannel-ds-vz6z7             1/1     Running   0          13m
kube-system   kube-proxy-pdvdw                  1/1     Running   0          19m
kube-system   kube-scheduler-skywork            1/1     Running   0          19m
```

最后，如果是测试用的单节点，为了让负载可以跑在k8s的master节点上，执行下列命令去除master的污点：

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### network cidr 没有设置的问题

但是，如果没有设置 `--pod-network-cidr=10.244.0.0/16`，则kube-flannel-ds 的 pod，总是报错：

```bash
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                              READY   STATUS              RESTARTS   AGE
......
kube-system   kube-flannel-ds-amd64-jqrk8       0/1     Error               6          6m48s
```

从日志上看：

```bash
$ kubectl -n kube-system logs kube-flannel-ds-amd64-jqrk8
I0201 15:33:40.001592       1 main.go:475] Determining IP address of default interface
I0201 15:33:40.001794       1 main.go:488] Using interface with name wlp6s0 and address 192.168.0.108
I0201 15:33:40.001805       1 main.go:505] Defaulting external address to interface address (192.168.0.108)
I0201 15:33:40.102463       1 kube.go:131] Waiting 10m0s for node controller to sync
I0201 15:33:40.102495       1 kube.go:294] Starting kube subnet manager
I0201 15:33:41.102611       1 kube.go:138] Node controller sync successful
I0201 15:33:41.102633       1 main.go:235] Created subnet manager: Kubernetes Subnet Manager - skywork
I0201 15:33:41.102638       1 main.go:238] Installing signal handlers
I0201 15:33:41.102710       1 main.go:353] Found network config - Backend type: vxlan
I0201 15:33:41.102748       1 vxlan.go:120] VXLAN config: VNI=1 Port=0 GBP=false DirectRouting=false
E0201 15:33:41.102897       1 main.go:280] Error registering network: failed to acquire lease: node "skywork" pod cidr not assigned
I0201 15:33:41.102925       1 main.go:333] Stopping shutdownHandler...
```

就只能运行 `kubeadm reset` ，然后删除 .kube 目录，再次执行 `kubeadm init`。

### coredns crash的问题

如果遇到 coredns 总是CrashLoopBackOff：

```bash
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                              READY   STATUS             RESTARTS   AGE
kube-system   coredns-fb8b8dccf-cfkcd           0/1     CrashLoopBackOff   2          113s
kube-system   coredns-fb8b8dccf-t2tq5           0/1     CrashLoopBackOff   2          113s
```

可以查看log：

```bash
$ kubectl logs -f -n kube-system coredns-fb8b8dccf-cfkcd

.:53
2019-06-09T13:54:40.288Z [INFO] CoreDNS-1.3.1
2019-06-09T13:54:40.288Z [INFO] linux/amd64, go1.11.4, 6b56a9c
CoreDNS-1.3.1
linux/amd64, go1.11.4, 6b56a9c
2019-06-09T13:54:40.288Z [INFO] plugin/reload: Running configuration MD5 = 599b9eb76b8c147408aed6a0bbe0f669
2019-06-09T13:54:41.751Z [FATAL] plugin/loop: Loop (127.0.0.1:57365 -> :53) detected for zone ".", see https://coredns.io/plugins/loop#troubleshooting. Query: "HINFO 3116341138164139510.3359948439978079754."
```
通常是和本机的DNS设置有关，比如我曾经设置了 `nameserver 127.0.0.1`，这条需要删除：

```bash
sudo vi /etc/resolv.conf

sudo systemctl daemon-reload
sudo systemctl restart docker
```





### 参考资料

- [Creating a single master cluster with kubeadm](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/): 这里有详细的flannel等network addon的安装设置
- [Installing a pod network add-on](https://ithelp.ithome.com.tw/articles/10209632?sc=iThelpR)