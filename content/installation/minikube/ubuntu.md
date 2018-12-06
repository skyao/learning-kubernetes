---
date: 2018-12-06T08:00:00+08:00
title: ubuntu下用minikube安装
menu:
  main:
    parent: "installation-minikube"
weight: 211
description : "ubuntu下用minikube安装kubernetes"
---


参考官方资料:

https://kubernetes.io/docs/tasks/tools/install-minikube/

## 准备

1. VT-x 或 AMD-v 虚拟化支持必须在电脑的bios中开启
2. 安装虚拟机: 对于 Linux, 可以安装 VirtualBox 或 KVM

### 安装VirtualBox

具体操作参考这里：

https://skyao.io/learning-linux-mint/daily/system/virtualbox.html

### 安装kubectl

参考：

https://kubernetes.io/docs/tasks/tools/install-kubectl/

执行命令如下：

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version
```

注意：如果更新了Minikube，务必重新再执行上述步骤以便更新kubectl到最新版本，否则可能出现问题，比如`minikube dashboard`打不开浏览器。

## 安装Minikube

参考文档:

- [安装](https://github.com/kubernetes/minikube/releases): 注意检查最新版本
- [使用](https://github.com/kubernetes/minikube/blob/v0.23.0/README.md)

安装命令：

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.25.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/

minikube version
```

如果有更新，修改上面命令中的版本号即可。

## 通过minikube运行k8s

参考：　https://kubernetes.io/docs/getting-started-guides/minikube/

> #### warn::一定要设置代理
>
> 切记要设置代理，否则会因为网络被墙导致无法获取镜像，`kubectl get pod` 会发现一直阻塞在 status　`ContainerCreating`．

```bash
minikube start --docker-env http_proxy=http://192.168.31.152:8123 --docker-env https_proxy=http://192.168.31.152:8123 --docker-env no_proxy=localhost,127.0.0.1,::1,192.168.31.0/24,192.168.99.0/24
```

如果有全局翻墙，就可以简单的`minikube start`启动。

> 注意：这里的代理一定要是http代理，因此不能直接输入shadowsocks的地址，要用pilipo提供的http代理，而且要设置pilipo的proxyAddress，不能只监听127.0.0.1．

如果没有正确设置代理就执行了 `minikube start`，则只能删除虚拟机然后重新来过，后面再设置代理是没有效果的：

```bash
minikube stop
minikube delete
```

按照上面的文章可以测试minikube是否可以正常工作．如果想查看k8s的控制台，输入下面命令：

```bash
minikube dashboard
```

## 备忘

用到的一些的命令，备用．

kubectl：

- kubectl get pod
- kubectl get pods --all-namespaces
- kubectl get service
- kubectl describe po hello-minikube-180744149-lj0rd

minikube：

- minikube dashboard
- minikube status
- minikube service hello-minikube --url
- curl $(minikube service hello-minikube --url)
