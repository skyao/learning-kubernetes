---
title: "准备工作"
linkTitle: "准备"
weight: 10
date: 2025-03-04
description: >
  在 debian12 上安装 kubenetes 之前的准备工作
---

## 系统更新

确保更新debian系统到最新，移除不再需要的软件，清理无用的安装包：

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt autoremove
sudo apt autoclean
```

如果更新了内核，最好重启一下。

## swap 分区

安装 Kubernetes 要求机器不能有 swap 分区。

参考：

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#swap-configuration

## 开启模块

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

## container runtime

Kubernetes 支持多种 container runtime，这里暂时继续使用 docker engine + cri-dockerd。

参考：

https://kubernetes.io/docs/setup/production-environment/container-runtimes/

### 安装 docker + cri-dockerd

docker 的安装参考：

https://skyao.io/learning-docker/docs/installation/debian12/

cri-dockerd 的安装参考：

https://mirantis.github.io/cri-dockerd/usage/install/

从 release 页面下载：

https://github.com/Mirantis/cri-dockerd/releases

debian 12 选择下载文件

https://github.com/Mirantis/cri-dockerd/releases/download/v0.4.0/cri-dockerd_0.4.0.3-0.debian-bookworm_amd64.deb

下载后安装：

```bash
sudo dpkg -i ./cri-dockerd_0.4.0.3-0.debian-bookworm_amd64.deb
```

安装后会提示：

```bash
Selecting previously unselected package cri-dockerd.
(Reading database ... 68005 files and directories currently installed.)
Preparing to unpack .../cri-dockerd_0.4.0.3-0.debian-bookworm_amd64.deb ...
Unpacking cri-dockerd (0.4.0~3-0~debian-bookworm) ...
Setting up cri-dockerd (0.4.0~3-0~debian-bookworm) ...
Created symlink /etc/systemd/system/multi-user.target.wants/cri-docker.service → /lib/systemd/system/cri-docker.service.
Created symlink /etc/systemd/system/sockets.target.wants/cri-docker.socket → /lib/systemd/system/cri-docker.socket.
```

安装后查看状态：

```bash
sudo systemctl status cri-docker.service
```

如果成功则状态为：

```bash
● cri-docker.service - CRI Interface for Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/cri-docker.service; enabled; preset: enabled)
     Active: active (running) since Sat 2025-05-10 20:39:41 CST; 19s ago
TriggeredBy: ● cri-docker.socket
       Docs: https://docs.mirantis.com
   Main PID: 8294 (cri-dockerd)
      Tasks: 9
     Memory: 13.1M
        CPU: 18ms
     CGroup: /system.slice/cri-docker.service
             └─8294 /usr/bin/cri-dockerd --container-runtime-endpoint fd://

May 10 20:39:41 debian12 cri-dockerd[8294]: time="2025-05-10T20:39:41+08:00" level=info msg="Hairpin mode is set to none"
May 10 20:39:41 debian12 cri-dockerd[8294]: time="2025-05-10T20:39:41+08:00" level=info msg="The binary conntrack is not installed, this can cau>
May 10 20:39:41 debian12 cri-dockerd[8294]: time="2025-05-10T20:39:41+08:00" level=info msg="The binary conntrack is not installed, this can cau>
May 10 20:39:41 debian12 cri-dockerd[8294]: time="2025-05-10T20:39:41+08:00" level=info msg="Loaded network plugin cni"
May 10 20:39:41 debian12 cri-dockerd[8294]: time="2025-05-10T20:39:41+08:00" level=info msg="Docker cri networking managed by network plugin cni"
May 10 20:39:41 debian12 cri-dockerd[8294]: time="2025-05-10T20:39:41+08:00" level=info msg="Setting cgroupDriver systemd"
May 10 20:39:41 debian12 cri-dockerd[8294]: time="2025-05-10T20:39:41+08:00" level=info msg="Docker cri received runtime config &RuntimeConfig{N>
May 10 20:39:41 debian12 cri-dockerd[8294]: time="2025-05-10T20:39:41+08:00" level=info msg="Starting the GRPC backend for the Docker CRI interf>
May 10 20:39:41 debian12 cri-dockerd[8294]: time="2025-05-10T20:39:41+08:00" level=info msg="Start cri-dockerd grpc backend"
May 10 20:39:41 debian12 systemd[1]: Started cri-docker.service - CRI Interface for Docker Application Container Engine.
```

### 安装 containerd

TODO：后面考虑换 containerd

## 安装 helm

参考：

https://helm.sh/docs/intro/install/#from-apt-debianubuntu

安装：

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

安装后取消 helm 的自动更新：

```bash
sudo vi /etc/apt/sources.list.d/helm-stable-debian.list
```

查看安装的版本：

```bash
$ helm version
version.BuildInfo{Version:"v3.17.3", GitCommit:"e4da49785aa6e6ee2b86efd5dd9e43400318262b", GitTreeState:"clean", GoVersion:"go1.23.7"}
```



