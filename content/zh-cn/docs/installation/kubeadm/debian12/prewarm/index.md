---
title: "预热安装 kubenetes"
linkTitle: "预热安装"
weight: 20
date: 2025-05-12
description: >
  在 debian12 上用 kubeadm 预热安装 kubenetes
---

## 原理

所谓预热安装，就是在在线安装的基础上，在执行 `kubeadmin init` 之前，提前准备好所有的安装文件和镜像文件，然后制造成 pve 模板。

之后就可以重用该模板，在需要时创建虚拟机，在虚拟机中执行 `kubeadmin init` 即可快速安装 kubenetes。

原则上，在执行 `kubeadmin init` 之前的各种准备工作都可以参考在线安装的方式。而在 `kubeadmin init` 之后的安装工作，就只能通过提前准备安装文件，提前下载镜像文件等方式来加速。

## 准备工作

- 安装 docker： 参考 https://skyao.io/learning-docker/docs/installation/debian12/ ，在线安装和离线安装都可以。

- 安装 kubeadm： 参考前面的在线安装方式，或者直接用后面的离线安装方式，将 cri-dockerd / helm 和kubeadm / kubelete / kubectl 安装好。

## 预下载镜像文件

### k8s cluster

```bash
kubeadm config images pull --cri-socket unix:///var/run/cri-dockerd.sock
```

这样就可以提前下载好 kubeadm init 时需要的镜像文件：

```bash
[config/images] Pulled registry.k8s.io/kube-apiserver:v1.33.0
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.33.0
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.33.0
[config/images] Pulled registry.k8s.io/kube-proxy:v1.33.0
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.12.0
[config/images] Pulled registry.k8s.io/pause:3.10
[config/images] Pulled registry.k8s.io/etcd:3.5.21-0
```

### flannel

下载 flannel 需要的镜像文件：

```bash
docker pull ghcr.io/flannel-io/flannel-cni-plugin:v1.6.2-flannel1
docker pull ghcr.io/flannel-io/flannel:v0.26.7
```

参考在线安装文档准备以下 yaml 文件：

- `~/work/soft/k8s/menifests/kube-flannel.yml`

### dashboard

查看 dashboard 的最新版本：

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm repo update
helm search repo kubernetes-dashboard -l
```

发现 dashboard 的最新版本是 7.12.0，所以下载 dashboard 需要的 charts 文件：

```bash
helm pull kubernetes-dashboard/kubernetes-dashboard --version 7.12.0 --untar --untardir ~/work/soft/k8s/charts
```

下载 dashboard 需要的镜像文件：

```bash
docker pull docker.io/kubernetesui/dashboard-api:1.12.0
docker pull docker.io/kubernetesui/dashboard-auth:1.2.4
docker pull docker.io/kubernetesui/dashboard-web:1.6.2
docker pull docker.io/kubernetesui/dashboard-metrics-scraper:1.2.2
```

参考在线安装文档准备以下 yaml 文件：

- `~/work/soft/k8s/menifests/dashboard-adminuser-binding.yaml`
- `~/work/soft/k8s/menifests/dashboard-adminuser.yaml`
- `~/work/soft/k8s/menifests/dashboard-adminuser-secret.yaml`

### metrics-server

下载 metrics-server 需要的镜像文件：

```bash
docker pull registry.k8s.io/metrics-server/metrics-server:v0.7.2
docker pull docker.io/kubernetesui/dashboard-metrics-scraper:1.2.2
```

参考在线安装文档准备以下 yaml 文件：

- `~/work/soft/k8s/menifests/metrics-server-components.yaml`

## 安装

### 手工安装

执行 `kubeadm init` 命令， 注意检查并修改 IP 地址为实际 IP 地址：

```bash
NODE_IP=192.168.3.175

sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock --apiserver-advertise-address=$NODE_IP
```

配置 kube config：

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

配置 flannel 网络：

```bash
kubectl apply -f ~/work/soft/k8s/menifests/kube-flannel.yml
```

去除污点：

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

安装 dashboard ：

```bash
helm upgrade --install kubernetes-dashboard \
  ~/work/soft/k8s/charts/kubernetes-dashboard \
  --create-namespace \
  --namespace kubernetes-dashboard
```

准备用于登录 dashboard 的 admin-user 用户：

```bash
kubectl apply -f ~/work/soft/k8s/menifests/dashboard-adminuser.yaml
kubectl apply -f ~/work/soft/k8s/menifests/dashboard-adminuser-binding.yaml

kubectl -n kubernetes-dashboard create token admin-user
kubectl apply -f ~/work/soft/k8s/menifests/dashboard-adminuser-secret.yaml

ADMIN_USER_TOKEN=$(kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 -d)
echo $ADMIN_USER_TOKEN > ~/work/soft/k8s/dashboard-admin-user-token.txt
echo "admin-user token is: $ADMIN_USER_TOKEN"
```

将 kubernetes-dashboard-kong-proxy 设置为 NodePort 类型：

```bash
kubectl -n kubernetes-dashboard patch service kubernetes-dashboard-kong-proxy -p '{"spec":{"type":"NodePort"}}'
```

获取 NodePort：

```bash
NODE_PORT=$(kubectl -n kubernetes-dashboard get service kubernetes-dashboard-kong-proxy \
  -o jsonpath='{.spec.ports[0].nodePort}')
echo "url is: https://$NODE_IP:$NODE_PORT"
```

安装 metrics-server：

```bash
kubectl apply -f ~/work/soft/k8s/menifests/metrics-server-components.yaml

kubectl wait --namespace kube-system \
  --for=condition=Ready \
  --selector=k8s-app=metrics-server \
  --timeout=300s pod
echo "metrics-server installed, have a try:"
echo
echo "kubectl top nodes"
echo
kubectl top nodes
echo
echo "kubectl top pods -n kube-system"
echo
kubectl top pods -n kube-system
```

### 脚本自动安装

```bash
#!/usr/bin/env zsh

# Kubernetes 自动化安装脚本 (Debian 12 + Helm + Dashboard + Metrics Server)
# 使用方法: sudo ./install_k8s_prewarm.zsh <NODE_IP>

# 获取脚本所在绝对路径
K8S_INSTALL_PATH=$(cd "$(dirname "$0")"; pwd)
MANIFESTS_PATH="$K8S_INSTALL_PATH/menifests"
CHARTS_PATH="$K8S_INSTALL_PATH/charts"
echo "🔍 检测到安装文件目录: $K8S_INSTALL_PATH"

# 检查是否以 root 执行
if [[ $EUID -ne 0 ]]; then
  echo "❌ 此脚本必须以 root 身份运行" 
  exit 1
fi

# 获取节点 IP
if [[ -z "$1" ]]; then
  echo "ℹ️ 用法: $0 <节点IP>"
  exit 1
fi
NODE_IP=$1

# 安装日志
LOG_FILE="$K8S_INSTALL_PATH/k8s_install_$(date +%Y%m%d_%H%M%S).log"
exec > >(tee -a "$LOG_FILE") 2>&1

echo "📅 开始安装 Kubernetes 集群 - $(date)"
echo "🔧 节点IP: $NODE_IP"
echo "📁 资源目录: $K8S_INSTALL_PATH"

# 步骤1: kubeadm 初始化
echo "🚀 正在初始化 Kubernetes 控制平面..."
kubeadm_init() {
  kubeadm init \
    --pod-network-cidr 10.244.0.0/16 \
    --cri-socket unix:///var/run/cri-dockerd.sock \
    --apiserver-advertise-address=$NODE_IP
  
  if [[ $? -ne 0 ]]; then
    echo "❌ kubeadm init 失败"
    exit 1
  fi
}
kubeadm_init
sleep 3

# 步骤2: 配置 kubectl
echo "⚙️ 为 root 用户配置 kubectl..."
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
echo "⚙️ 为当前用户配置 kubectl..."
CURRENT_USER_HOME=$(getent passwd $SUDO_USER | cut -d: -f6)
mkdir -p $CURRENT_USER_HOME/.kube
cp -i /etc/kubernetes/admin.conf $CURRENT_USER_HOME/.kube/config
chown $(id -u $SUDO_USER):$(id -g $SUDO_USER) $CURRENT_USER_HOME/.kube/config

# 步骤3: 安装 Flannel 网络插件
echo "🌐 正在安装 Flannel 网络..."
kubectl apply -f "$MANIFESTS_PATH/kube-flannel.yml" || {
  echo "❌ Flannel 安装失败"
  exit 1
}
sleep 3

# 步骤4: 去除控制平面污点
echo "✨ 去除控制平面污点..."
kubectl taint nodes --all node-role.kubernetes.io/control-plane- || {
  echo "⚠️ 去除污点失败 (可能不影响功能)"
}

# 步骤5: 从本地安装 Dashboard
echo "📊 正在从本地安装 Kubernetes Dashboard..."
helm upgrade --install kubernetes-dashboard \
  "$CHARTS_PATH/kubernetes-dashboard" \
  --create-namespace \
  --namespace kubernetes-dashboard || {
  echo "❌ Dashboard 安装失败"
  exit 1
}
sleep 3

# 步骤6: 配置 Dashboard 管理员用户
echo "👤 创建 Dashboard 管理员用户..."
kubectl apply -f "$MANIFESTS_PATH/dashboard-adminuser.yaml" || {
  echo "❌ 创建 admin-user 失败"
  exit 1
}

kubectl apply -f "$MANIFESTS_PATH/dashboard-adminuser-binding.yaml" || {
  echo "❌ 创建 RBAC 绑定失败"
  exit 1
}

kubectl apply -f "$MANIFESTS_PATH/dashboard-adminuser-secret.yaml" || {
  echo "⚠️ 创建 Secret 失败 (可能已存在)"
}

# 获取并保存 Token
echo "🔑 获取管理员 Token..."
ADMIN_TOKEN=$(kubectl -n kubernetes-dashboard create token admin-user 2>/dev/null || \
  kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 -d)

if [[ -z "$ADMIN_TOKEN" ]]; then
  echo "❌ 获取 Token 失败"
  exit 1
fi

echo "$ADMIN_TOKEN" > "$K8S_INSTALL_PATH/dashboard-admin-user-token.txt"
echo "✅ Token 已保存到: $K8S_INSTALL_PATH/dashboard-admin-user-token.txt"

# 步骤7: 修改 Dashboard Service 类型
echo "🔧 修改 Dashboard 服务类型为 NodePort..."
kubectl -n kubernetes-dashboard patch service kubernetes-dashboard-kong-proxy \
  -p '{"spec":{"type":"NodePort"}}' || {
  echo "❌ 修改服务类型失败"
  exit 1
}
sleep 3

# 获取 NodePort
NODE_PORT=$(kubectl -n kubernetes-dashboard get service kubernetes-dashboard-kong-proxy \
  -o jsonpath='{.spec.ports[0].nodePort}')

echo "🌍 Dashboard 访问地址: https://$NODE_IP:$NODE_PORT"
echo "🔑 登录 Token: $ADMIN_TOKEN"

# 步骤8: 安装 Metrics Server
echo "📈 正在安装 Metrics Server..."
kubectl apply -f "$MANIFESTS_PATH/metrics-server-components.yaml" || {
  echo "❌ Metrics Server 安装失败"
  exit 1
}

# 等待 Metrics Server 就绪
echo "⏳ 等待 Metrics Server 就绪 (最多5分钟)..."
kubectl wait --namespace kube-system \
  --for=condition=Ready \
  --selector=k8s-app=metrics-server \
  --timeout=300s pod || {
  echo "❌ Metrics Server 启动超时"
  exit 1
}

# 验证安装
echo "✅ 安装完成!"
sleep 5
echo ""
echo "🛠️ 验证命令:"
echo "kubectl top nodes"
kubectl top nodes
echo ""
echo "kubectl top pods -n kube-system"
kubectl top pods -n kube-system

echo ""
echo "📌 重要信息:"
echo "Dashboard URL: https://$NODE_IP:$NODE_PORT"
echo "Token 文件: $K8S_INSTALL_PATH/dashboard-admin-user-token.txt"
echo "安装日志: $LOG_FILE"
```
