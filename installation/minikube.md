# Minikube安装

> #### info::参考官方资料
>
> https://kubernetes.io/docs/tasks/tools/install-minikube/

## 准备

1. VT-x 或 AMD-v 虚拟化支持必须在电脑的bios中开启
2. 安装虚拟机: 对于 Linux, 可以安装 VirtualBox 或 KVM

### 安装VirtualBox

下载地址:

https://www.virtualbox.org/wiki/Downloads

- VirtualBox 5.1.26 platform packages
- VirtualBox 5.1.26 Oracle VM VirtualBox Extension Pack

安装下载的　`virtualbox-5.1_5.1.26-117224-Ubuntu-xenial_amd64.deb`

完成后，双击 `Oracle_VM_VirtualBox_Extension_Pack-5.1.26-117224.vbox-extpack` 再安装扩展包.

###　安装kubectl

https://kubernetes.io/docs/tasks/tools/install-kubectl/

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

## 安装Minikube

参考文档:

- [安装](https://github.com/kubernetes/minikube/releases)
- [使用](https://github.com/kubernetes/minikube/blob/v0.21.0/README.md)

安装命令：

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.21.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

## 通过minikube运行k8s

参考：　https://kubernetes.io/docs/getting-started-guides/minikube/

> #### warn::一定要设置代理
>
> 切记要设置代理，否则会因为网络被墙导致无法获取镜像，`kubectl get pod` 会发现一直阻塞在 status　`ContainerCreating`．

```bash
minikube start --docker-env http_proxy=http://192.168.31.152:8123 --docker-env https_proxy=http://192.168.31.152:8123 --docker-env no_proxy=localhost,127.0.0.1,::1,192.168.31.152,192.168.31.34,192.168.31.31,192.168.99.1
```

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
