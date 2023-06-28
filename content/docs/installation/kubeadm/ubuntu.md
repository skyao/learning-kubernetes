---
title: "ubuntu 下用 kubeadm 安装 kubenetes"
linkTitle: "ubuntu"
weight: 10
date: 2021-02-01
description: >
  通过 kubeadm 在 ubuntu 上安装 kubenetes
---

以 ubuntu server 22.04 为例，参考 Kubernetes 官方文档:

- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/

## 前期准备

### 检查 docker 版本

{{% alert title="注意" color="warning" %}}
暂时固定使用 docker 和 k8s 的特定版本搭配：

- docker： 20.10.21
- k8s： 1.23.14

具体原因请见最下面的解释。

{{% /alert %}}

### 检查 container 配置

```bash
sudo vi /etc/containerd/config.toml
```

确保文件不存在或者一下这行内容被注释:

```bash
# disabled_plugins = ["cri"]
```

修改之后需要重启 containerd：

```bash
sudo systemctl restart containerd.service
```

> 备注：如果不做这个修改，k8s 安装时会报错 “CRI v1 runtime API is not implemented”。

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

## 安装kubeadm

{{% alert title="切记" color="warning" %}}
想办法搞定全局翻墙，不然 kubeadm 安装是非常麻烦的。
{{% /alert %}}

执行如下命令：

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
```

### ~~安装最新版本~~

```bash
sudo apt-get install -y kubelet kubeadm kubectl
```

安装完成后

```bash
kubectl version --output=yaml
```

查看 kubectl 版本：

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

查看 kubeadm 版本：

```bash
kubeadm version 
kubeadm version: &version.Info{Major:"1", Minor:"27", GitVersion:"v1.27.3", GitCommit:"25b4e43193bcda6c7328a6d147b1fb73a33f1598", GitTreeState:"clean", BuildDate:"2023-06-14T09:52:26Z", GoVersion:"go1.20.5", Compiler:"gc", Platform:"linux/amd64"}
```

查看 kubelet 版本：

```bash
kubelet --version
Kubernetes v1.27.3
```

### 安装特定版本

如果希望安装特定版本：

```bash
sudo apt-get install kubelet=1.23.14-00 kubeadm=1.23.14-00 kubectl=1.23.14-00
```

具体有哪些可用的版本，可以看这里：

https://packages.cloud.google.com/apt/dists/kubernetes-xenial/main/binary-amd64/Packages

## 安装k8s

> 参考：https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

{{% alert title="同样切记" color="warning" %}}
想办法搞定全局翻墙。
{{% /alert %}}

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 -v=9
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.100.40 -v=9
```

注意后面为了使用 CNI network 和 Flannel，我们在这里设置了 `--pod-network-cidr=10.244.0.0/16`，如果不加这个设置，Flannel 会一直报错。如果机器上有多个网卡，可以用 `--apiserver-advertise-address` 指定要使用的IP地址。

kubeadm init 输出如下：

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

kubeadm join 192.168.0.57:6443 --token gwr923.gctdq2sr423mrwp7 \
	--discovery-token-ca-cert-hash sha256:ad86f4eb0d430fc1bdf784ae655dccdcb14881cd4ca8d03d84cd2135082c4892 
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
$ kubectl describe node ubuntu2204 
Name:               ubuntu2204
Roles:              control-plane

......

  Ready            False   Wed, 28 Jun 2023 16:53:27 +0000   Wed, 28 Jun 2023 16:52:41 +0000   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
```

安装 flannel 作为 pod network add-on：

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

> 备注：有时会遇到 raw.githubusercontent.com 这个域名被污染，解析为 127.0.0.1，导致无法访问。解决方法是访问 https://ipaddress.com/website/raw.githubusercontent.com 然后查看可用的IP地址，找一个速度最快的，在 `/etc/hosts` 文件中加入一行记录即可，如 `185.199.111.133         raw.githubusercontent.com` 。

稍等就可以看到 node 的状态变为 Ready了：

```bash
$ kubectl get node                                                                                           
NAME        STATUS   ROLES                  AGE     VERSION
skyserver   Ready    control-plane,master   4m52s   v1.23.5
```

最后，如果是测试用的单节点，为了让负载可以跑在 k8s 的 master 节点上，执行下列命令去除 master/control-plane 的污点：

```bash
# 以前的污点名为 master
# kubectl taint nodes --all node-role.kubernetes.io/master-
# 新版本污点名改为 control-plane （master政治不正确）
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
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



## 常见问题

### CRI v1 runtime API is not implemented

如果类似的报错（新版本）：

```bash
[preflight] Some fatal errors occurred:
	[ERROR CRI]: container runtime is not running: output: time="2023-06-28T16:12:49Z" level=fatal msg="validate service connection: CRI v1 runtime API is not implemented for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unimplemented desc = unknown service runtime.v1.RuntimeService"
, error: exit status 1
```

或者报错（老一些的版本）:

```bash
[preflight] Some fatal errors occurred:
	[ERROR CRI]: container runtime is not running: output: E1125 11:16:01.799551   14661 remote_runtime.go:948] "Status from runtime service failed" err="rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService"
time="2022-11-25T11:16:01+08:00" level=fatal msg="getting status of runtime: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService"
, error: exit status 1
```

这都是因为 containerd 的默认配置文件中 disable 了 CRI 的原因，可以打开文件 `/etc/containerd/config.toml ` 看到这行

```properties
disabled_plugins = ["cri"]
```

将这行注释之后，重启 containerd ：

```bash
sudo systemctl restart containerd.service
```

之后重新尝试 kubeadm init。

参考:

- https://forum.linuxfoundation.org/discussion/862825/kubeadm-init-error-cri-v1-runtime-api-is-not-implemented

### 控制平面不启动或者异常重启

安装最新版本（1.27 / 1.25）完成显示成功，但是控制平面没有启动，6443 端口无法连接：

```bash
k get node       

E0628 16:34:50.966940    6581 memcache.go:265] couldn't get current server API group list: Get "https://192.168.0.57:6443/api?timeout=32s": read tcp 192.168.0.57:41288->192.168.0.1:7890: read: connection reset by peer - error from a previous attempt: read tcp 192.168.0.57:41276->192.168.0.1:7890: read: connection reset by peer
```

使用中发现控制平面经常不稳定， 大量的 pod 在反复重启，日志中有提示：pod sandbox changed。

记录测试验证有问题的版本：

- kubeadm： 1.27.3 / 1.25.6
- kubelet：1.27.3 / 1.25.6
- docker： 24.0.2 / 20.10.21

尝试回退 docker 版本，k8s 1.27 的 changelog 中，

https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.27.md 提到的 docker 版本是 v20.10.21 （incompatible 是什么鬼?) ：

```
github.com/docker/docker: v20.10.18+incompatible → v20.10.21+incompatible
```

这个 v20.10.21 版本我翻了一下我之前的安装记录，非常凑巧之前是有使用这个 docker 版本的，而且稳定没出问题。因此考虑换到这个版本：

```bash
VERSION_STRING=5:20.10.21~3-0~ubuntu-jammy
sudo apt-get install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin
```

k8s 暂时固定选择 1.23.14 这个经过验证的版本：

```bash
sudo apt-get install kubelet=1.23.14-00 kubeadm=1.23.14-00 kubectl=1.23.14-00
```

> 备注： 1.27.3 / 1.25.6 这两个 k8s 的版本都验证过会有问题，暂时不清楚原因，先固定用 1.23.14。
>
> 后续再排查。

## 失败重来

如果遇到安装失败，需要重新开始，或者想铲掉现有的安装，则可以：

1. 运行`kubeadm reset` 
2. 删除 `.kube` 目录
3. 再次执行 `kubeadm init`

如果网络设置有改动，则需要彻底的重置网络。具体见下一章。

## 将节点加入到集群

如果有多个kubenetes节点（即多台机器），则需要将其他节点加入到集群中。具体见下一章。

