---
title: "ubuntu 20.04下用 kubeadm 安装 kubenetes"
linkTitle: "ubuntu 20.04"
weight: 2110
date: 2021-02-01
description: >
  通过 kubeadm 在 ubuntu 上安装 kubenetes
---


参考 Kubernetes 官方文档:

- [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
- https://kubernetes.io/docs/setup/independent/install-kubeadm/

## 前期准备

### 关闭防火墙
```bash
systemctl disable firewalld && systemctl stop firewalld
```

### 安装docker和bridge-utils

要求节点上安装有 docker (或者其他container runtime）和 bridge-utils (用来操作linux bridge).

查看 docker 版本：

```bash
$ docker --version
Docker version 20.10.12, build e91ed57
```

bridge-utils可以通过apt安装：

```bash
sudo apt-get install bridge-utils
```

### 设置iptables

要确保 `br_netfilter` 模块已经加载,可以通过运行 `lsmod | grep br_netfilter`来完成。

```bash
$ lsmod | grep br_netfilter
br_netfilter           28672  0
bridge                266240  1 br_netfilter
```

如需要明确加载，请调用 `sudo modprobe br_netfilter`。

为了让作为Linux节点的iptables能看到桥接流量，应该确保 `net.bridge.bridge-nf-call-iptables` 在 sysctl 配置中被设置为1，执行命令：

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

### 禁用虚拟内存swap

执行 `free -m` 命令检测:

```bash
$ free -m        
              total        used        free      shared  buff/cache   available
Mem:          15896        1665       11376          20        2854       13819
Swap:             0           0           0
```

如果Swap这一行不是0，则说明虚拟内存swap被开启了，需要关闭。

需要做两个事情：

1. 操作系统安装时就不要设置swap分区，如果有，删除该swap分区

2. 即使没有swap分区，也会开启swap，需要通过 `sudo vi /etc/fstab ` 找到swap 这一行：

    ```properties
    # 在swap分区这行前加 # 禁用掉swap
    /swapfile                                 none            swap    sw              0       0
    ```

    重启之后再用 `free -m` 命令检测。

### 设置docker的cgroup driver

docker 默认的 cgroup driver 是 cgroupfs，可以通过 docker info 命令查看：

```bash
$ docker info | grep "Cgroup Driver"
 Cgroup Driver: cgroupfs
```

而 kubernetes 在v1.22版本之后，如果用户没有在 KubeletConfiguration 下设置 cgroupDriver 字段，则 kubeadm 将默认为 `systemd`。

需要修改 docker 的 cgroup driver 为 `systemd`, 方式为打开 docker 的配置文件（如果不存在则创建）

```bash
sudo vi /etc/docker/daemon.json
```

增加内容：

```json
{
"exec-opts": ["native.cgroupdriver=systemd"]
}
```

修改完成后重启 docker：

```bash
systemctl restart docker

# 重启后检查一下
docker info | grep "Cgroup Driver"
```

否则，在安装过程中，由于 cgroup driver 的不一致，`kubeadm init` 命令会因为 kubelet 无法启动而超时失败，报错为:

```bash
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.

Unfortunately, an error has occurred:
        timed out waiting for the condition

This error is likely caused by:
        - The kubelet is not running
        - The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
        - 'systemctl status kubelet'
        - 'journalctl -xeu kubelet'
```

执行 `systemctl status kubelet` 会发现 kubelet 因为报错而退出，执行 `journalctl -xeu kubelet` 会发现有如下的错误信息：

```
Dec 26 22:31:21 skyserver2 kubelet[132861]: I1226 22:31:21.438523  132861 docker_service.go:264] "Docker Info" dockerInfo=&{ID:AEON:SBVF:43UK:WASV:YIQK:QGGA:7RU3:IIDK:DV7M:6QLH:5ICJ:KT6R Containers:2 ContainersRunning:0 ContainersPaused:>
Dec 26 22:31:21 skyserver2 kubelet[132861]: E1226 22:31:21.438616  132861 server.go:302] "Failed to run kubelet" err="failed to run Kubelet: misconfiguration: kubelet cgroup driver: \"systemd\" is different from docker cgroup driver: \"c>
Dec 26 22:31:21 skyserver2 systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
-- Subject: Unit process exited
-- Defined-By: systemd
-- Support: http://www.ubuntu.com/support
-- 
-- An ExecStart= process belonging to unit kubelet.service has exited.
-- 
-- The process' exit code is 'exited' and its exit status is 1.
```

参考：

* https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/
* https://blog.51cto.com/riverxyz/2537914

## 安装kubeadm

{{% alert title="切记" color="warning" %}}
想办法搞定全局翻墙，不然kubeadm安装是非常麻烦的。
{{% /alert %}}

按照[官方文档](https://kubernetes.io/docs/setup/independent/install-kubeadm/)的指示，执行如下命令：

```bash
sudo -i
apt-get update
apt-get install -y apt-transport-https ca-certificates curl
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl
```

这会安装最新版本的kubernetes:

```bash
......
Setting up conntrack (1:1.4.5-2) ...
Setting up kubectl (1.23.5-00) ...
Setting up ebtables (2.0.11-3build1) ...
Setting up socat (1.7.3.3-2) ...
Setting up cri-tools (1.23.0-00) ...
Setting up kubernetes-cni (0.8.7-00) ...
Setting up kubelet (1.23.5-00) ...
Created symlink /etc/systemd/system/multi-user.target.wants/kubelet.service → /lib/systemd/system/kubelet.service.
Setting up kubeadm (1.23.5-00) ...
Processing triggers for man-db (2.9.1-1) ...

# 查看版本
$ kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.5", GitCommit:"c285e781331a3785a7f436042c65c5641ce8a9e9", GitTreeState:"clean", BuildDate:"2022-03-16T15:57:37Z", GoVersion:"go1.17.8", Compiler:"gc", Platform:"linux/amd64"}

$ kubelet --version
Kubernetes v1.23.5

$ kubectl version
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.5", GitCommit:"c285e781331a3785a7f436042c65c5641ce8a9e9", GitTreeState:"clean", BuildDate:"2022-03-16T15:58:47Z", GoVersion:"go1.17.8", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

如果希望安装特定版本：

```bash
apt-get install kubelet=1.23.5-00 kubeadm=1.23.5-00 kubectl=1.23.5-00
```

具体有哪些可用的版本，可以看这里：

https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-amd64/Packages

## 安装k8s

{{% alert title="同样切记" color="warning" %}}
想办法搞定全局翻墙。
{{% /alert %}}

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 -v=9
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.100.40 -v=9
```

注意后面为了使用 CNI network 和 Flannel，我们在这里设置了 `--pod-network-cidr=10.244.0.0/16`，如果不加这个设置，Flannel 会一直报错。如果机器上有多个网卡，可以用 `--apiserver-advertise-address` 指定要使用的IP地址。

输出如下：

```bash
......

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

kubeadm join 192.168.100.40:6443 --token uq5nqn.bppygpcqty6icec4 \
	--discovery-token-ca-cert-hash sha256:51c13871cd25b122f3a743040327b98b1c19466d01e1804aa2547c047b83632b 
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
NAME        STATUS     ROLES                  AGE    VERSION
skyserver   NotReady   control-plane,master   3m7s   v1.23.5
```

kubectl describe 可以看到是因为没有安装 network plugin

```bash
$ kubectl describe node skyserver
Name:               skyserver
Roles:              control-plane,master
......
  Ready            False   Thu, 24 Mar 2022 13:57:21 +0000   Thu, 24 Mar 2022 13:57:06 +0000   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

安装flannel：

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

稍等就可以看到 node 的状态变为 Ready了：

```bash
$ kubectl get node                                                                                           
NAME        STATUS   ROLES                  AGE     VERSION
skyserver   Ready    control-plane,master   4m52s   v1.23.5
```

最后，如果是测试用的单节点，为了让负载可以跑在k8s的master节点上，执行下列命令去除master的污点：

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

可以通过 `kubectl describe node skyserver` 对比去除污点前后 node 信息中的 Taints 部分，去除污点前：

```yaml
Taints:             node.kubernetes.io/not-ready:NoExecute
                    node-role.kubernetes.io/master:NoSchedule
                    node.kubernetes.io/not-ready:NoSchedule
```

去除污点后：

```yaml
Taints:             <none>
```

### 失败重来

如果遇到安装失败，需要重新开始，或者想铲掉现有的安装，则可以：

1. 运行`kubeadm reset` 
2. 删除 `.kube` 目录
3. 再次执行 `kubeadm init`

如果网络设置有改动，则需要彻底的重置网络。

## 将节点加入到集群

如果有多个kubenetes节点（即多台机器），则需要将其他节点加入到集群中。具体见下一章。

