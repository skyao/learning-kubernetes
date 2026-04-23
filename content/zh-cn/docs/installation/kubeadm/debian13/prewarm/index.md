---
title: "预热安装 kubenetes"
linkTitle: "预热安装"
weight: 20
date: 2025-05-12
description: >
  在 debian13 上用 kubeadm 预热安装 kubenetes
---

## 原理

所谓预热安装，就是在在线安装的基础上，在执行 `kubeadmin init` 之前，提前准备好所有的安装文件和镜像文件，然后制作成 pve 模板。

之后就可以重用该模板，在需要时创建虚拟机，在虚拟机中执行 `kubeadmin init` 即可快速安装 kubenetes。

原则上，在执行 `kubeadmin init` 之前的各种准备工作都可以参考在线安装的方式。而在 `kubeadmin init` 之后的安装工作，就只能通过提前准备安装文件和提前下载镜像文件等方式来加速。为最大化提升速度，所有的镜像文件都将通过 habor 进行代理。

## 准备工作

- 安装 docker： 参考 https://skyao.net/learning-docker/docs/installation/debian13/ ，在线安装和离线安装都可以。

- 安装 kubeadm： 参考前面的在线安装方式，或者直接用后面的离线安装方式，将 cri-dockerd / helm 和kubeadm / kubelete / kubectl 安装好。

## 预下载镜像文件

### k8s cluster

```bash
kubeadm config images pull --cri-socket unix:///var/run/cri-dockerd.sock --config=kubeadm.yaml
```

这样就可以提前下载好 kubeadm init 时需要的镜像文件：

```bash
[config/images] Pulled 192.168.3.193:5000/k8s-proxy/kube-apiserver:v1.35.4
[config/images] Pulled 192.168.3.193:5000/k8s-proxy/kube-controller-manager:v1.35.4
[config/images] Pulled 192.168.3.193:5000/k8s-proxy/kube-scheduler:v1.35.4
[config/images] Pulled 192.168.3.193:5000/k8s-proxy/kube-proxy:v1.35.4
[config/images] Pulled 192.168.3.193:5000/k8s-proxy/coredns/coredns:v1.13.1
[config/images] Pulled 192.168.3.193:5000/k8s-proxy/pause:3.10.1
[config/images] Pulled 192.168.3.193:5000/k8s-proxy/etcd:3.6.6-0
```

准备 kubeadm.yaml 文件备用，内容同在线安装：

```bash
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock

---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.35.4
imageRepository: 192.168.3.193:5000/k8s-proxy
networking:
  podSubnet: 10.244.0.0/16
dns:
  imageRepository: 192.168.3.193:5000/k8s-proxy/coredns
  imageTag: v1.13.1
etcd:
  local:
    imageRepository: 192.168.3.193:5000/k8s-proxy
    imageTag: 3.6.6-0
```

### flannel

下载原始的 kube-flannel.yml 文件：

```bash
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

修改 kube-flannel.yml, 将原有的镜像文件地址从

```yaml
image: ghcr.io/flannel-io/flannel:v0.28.4
image: ghcr.io/flannel-io/flannel-cni-plugin:v1.9.1-flannel1
```

修改为：

```yaml
image: 192.168.3.193:5000/ghcr.io/flannel-io/flannel:v0.28.4
image: 192.168.3.193:5000/ghcr.io/flannel-io/flannel-cni-plugin:v1.9.1-flannel1
```

下载 flannel 需要的镜像文件：

```bash
docker pull 192.168.3.193:5000/ghcr.io/flannel-io/flannel-cni-plugin:v1.9.1-flannel1
docker pull 192.168.3.193:5000/ghcr.io/flannel-io/flannel:v0.28.4
```

### metrics-server

下载原始的 components.yaml 文件, 保存为 metrics-server-components.yaml：

```bash
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml 
mv components.yaml metrics-server-components.yaml
```

修改内容如下：

```yaml
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP   # 修改
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls # 添加
        image: 192.168.3.193:5000/registry.k8s.io/metrics-server/metrics-server:v0.8.1 # 修改
```

下载 metrics-server 需要的镜像文件：

```bash
docker pull 192.168.3.193:5000/registry.k8s.io/metrics-server/metrics-server:v0.8.1
```

如果制作过程中，下载了多余的镜像，可以用如下命令先清空，再重新拉取需要的镜像：

```bash
docker rmi -f $(docker images -q)
```

## 安装

### 手工安装

执行 `kubeadm init` 命令：

```bash
cd ~/work/soft/k8s/

sudo kubeadm init --config=kubeadm.yaml
```

配置 kube config：

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

配置 flannel 网络：

```bash
kubectl apply -f ~/work/soft/k8s/kube-flannel.yml
```

去除污点：

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

安装 metrics-server：

```bash
kubectl apply -f ~/work/soft/k8s/metrics-server-components.yaml

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
cd ~/work/soft/k8s/
vi install_k8s_prewarm.zsh
```

内容如下：

```bash
#!/usr/bin/env zsh

# Kubernetes 自动化安装脚本 (Debian 13 + Helm + Metrics Server)
# 使用方法: sudo ./install_k8s_prewarm.zsh

# 获取脚本所在绝对路径
K8S_INSTALL_PATH=$(cd "$(dirname "$0")"; pwd)
MANIFESTS_PATH="$K8S_INSTALL_PATH/menifests"
echo "🔍 检测到安装文件目录: $K8S_INSTALL_PATH"

# 检查是否以 root 执行
if [[ $EUID -ne 0 ]]; then
  echo "❌ 此脚本必须以 root 身份运行"
  exit 1
fi

# 安装日志
mkdir -p "$K8S_INSTALL_PATH/logs"
LOG_FILE="$K8S_INSTALL_PATH/logs/k8s_install_$(date +%Y%m%d_%H%M%S).log"
exec > >(tee -a "$LOG_FILE") 2>&1

echo "📅 开始安装 Kubernetes 集群 - $(date)"
echo "📁 资源目录: $K8S_INSTALL_PATH"

# 步骤1: kubeadm 初始化
echo "🚀 正在初始化 Kubernetes 控制平面..."
kubeadm_init() {
  sudo kubeadm init --config="$MANIFESTS_PATH/kubeadm.yaml"

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
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
echo "⚙️ 为当前用户配置 kubectl..."
CURRENT_USER_HOME=$(getent passwd $SUDO_USER | cut -d: -f6)
mkdir -p $CURRENT_USER_HOME/.kube
cp /etc/kubernetes/admin.conf $CURRENT_USER_HOME/.kube/config
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

# 步骤5: 安装 Metrics Server
echo "📈 正在安装 Metrics Server..."
kubectl apply -f "$MANIFESTS_PATH/metrics-server-components.yaml" || {
  echo "❌ Metrics Server 安装失败"
  exit 1
}

# 等待 Metrics Server 就绪
echo "⏳ 等待 Metrics Server 就绪 (最多5分钟)..."
kubectl rollout status deployment metrics-server \
  --namespace kube-system \
  --timeout=300s || {
  echo "❌ Metrics Server 启动超时"
  exit 1
}

# 验证安装
echo "✅ 安装完成!"
sleep 5
echo ""
echo "🛠️  验证命令:"
echo "kubectl top nodes"
kubectl top nodes
echo ""
echo "kubectl top pods -n kube-system"
kubectl top pods -n kube-system

echo ""
echo "安装日志: $LOG_FILE"

```

增加执行权限：

```bash
chmod +x install_k8s_prewarm.zsh
```

执行：

```bash
sudo ./install_k8s_prewarm.zsh
```