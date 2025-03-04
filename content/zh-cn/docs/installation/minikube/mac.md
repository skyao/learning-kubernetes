---
title: "MacOS下用minikube安装"
linkTitle: "MacOS"
weight: 233
date: 2021-02-01
description: >
  MacOS下用minikube安装kubernetes
---

### 安装VirtualBox

https://www.virtualbox.org/wiki/Downloads

下载安装for mac的版本，如 “ VirtualBox 6.0.4 platform packages” 下 的 OSX hosts，然后安装下载的img文件。

之后在下载 VirtualBox 6.0.4 Oracle VM VirtualBox Extension Pack，双击安装。

### 安装kubectl

```bash
brew install kubernetes-cli
```

### 安装minikube

```bash
brew cask install minikube
```

完成后测试一下：

```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.2", GitCommit:"cff46ab41ff0bb44d8584413b598ad8360ec1def", GitTreeState:"clean", BuildDate:"2019-01-13T23:15:13Z", GoVersion:"go1.11.4", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.2", GitCommit:"cff46ab41ff0bb44d8584413b598ad8360ec1def", GitTreeState:"clean", BuildDate:"2019-01-10T23:28:14Z", GoVersion:"go1.11.4", Compiler:"gc", Platform:"linux/amd64"}
```

### 启动minikube

如果是在可以翻墙的环境中，安装最新版本的kubernetes，可以简单执行命令：

```bash
minikube start
```

如果不支持全局翻墙，可以指定镜像地址，也可以指定要安装的kubernetes版本：

```bash
minikube start --memory=8192 --cpus=4 --disk-size=20g  --registry-mirror=https://docker.mirrors.ustc.edu.cn --kubernetes-version=v1.12.5 --docker-env http_proxy=http://192.168.0.40:8123 --docker-env https_proxy=http://192.168.0.40:8123 --docker-env no_proxy=localhost,127.0.0.1,::1,192.168.0.0/24,192.168.99.0/24


minikube start --memory=8192 --cpus=4 --disk-size=20g --kubernetes-version=v1.12.5 

```

实测下载还是出问题，怀疑是不是要在minikuber start 前面再加 `http_proxy=http://192.168.0.40:8123 http_proxys=http://192.168.0.40:8123 ` 稍后验证。

### 启动dashborad

标准方式，执行命令 `minikube dashboard` ，然后就会自动打开浏览器访问地址 http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login 

备注：之前各个版本都正常，最近用v0.33.1的minikuber安装的kubernetes 1.13.2版本，遇到问题，报错如下：

```bash
$ kubectl version
Client Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.2", GitCommit:"cff46ab41ff0bb44d8584413b598ad8360ec1def", GitTreeState:"clean", BuildDate:"2019-01-13T23:15:13Z", GoVersion:"go1.11.4", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"13", GitVersion:"v1.13.2", GitCommit:"cff46ab41ff0bb44d8584413b598ad8360ec1def", GitTreeState:"clean", BuildDate:"2019-01-10T23:28:14Z", GoVersion:"go1.11.4", Compiler:"gc", Platform:"linux/amd64"}

$ minikube version
minikube version: v0.33.1

$ minikube dashboard
Enabling dashboard ...
Verifying dashboard health ...
Launching proxy ...
Verifying proxy health ...

http://127.0.0.1:51695/api/v1/namespaces/kube-system/services/http:kubernetes-dashboard:/proxy/ is not responding properly: Temporary Error: unexpected response code: 503
Temporary Error: unexpected response code: 503
Temporary Error: unexpected response code: 503
Temporary Error: unexpected response code: 503
```

导致无法打开dashboard，只好用 kubeproxy 的方式：

```bash
$ kubectl proxy
Starting to serve on 127.0.0.1:8001
```

然后手动打开地址： http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/login

参考：https://stackoverflow.com/questions/52916548/minikube-dashboard-returns-503-error-on-macos

然后又是遇到 dashboard 登录的问题。