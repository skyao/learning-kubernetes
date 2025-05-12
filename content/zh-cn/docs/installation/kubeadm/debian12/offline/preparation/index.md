---
title: "在 debian12 上离线安装 kubeadmin"
linkTitle: "安装 kubeadmin"
weight: 10
date: 2025-03-04
description: >
  在 debian12 上用 kubeadm 离线安装 kubeadmin
---

## 制作离线安装包

```bash
mkdir -p ~/temp/k8s-offline/
cd ~/temp/k8s-offline/
```

### cri-dockerd

下载安装包：

```bash
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.4.0/cri-dockerd_0.4.0.3-0.debian-bookworm_amd64.deb
```

### helm

参考在线安装的方式， 同样需要先添加 helm 的 apt 仓库，

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
```

然后找到需要安装的版本， 下载离线安装包。

```bash
apt-get download helm
```

### kubeadmin

添加 k8s 的 keyrings：

```bash
export K8S_VERSION=1.33

# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list


sudo apt-get update
```

下载安装包：

```bash
# 下载 k8s 的 .deb 包
apt-get download kubelet kubeadm kubectl

# 下载所有依赖（可能需要运行多次直到无新依赖）
apt-get download $(apt-cache depends kubelet kubeadm kubectl | grep -E 'Depends|Recommends' | cut -d ':' -f 2 | tr -d ' ' | grep -v "^kube" | sort -u)

rm -r iptables*.deb
```

下载 kubernetes-cni:

```bash
apt-get download kubernetes-cni
```

完成后的离线安装包如下：

```bash
ls -lh
total 124M
-rw-r--r-- 1 sky sky   35K Mar 23  2023 conntrack_1%3a1.4.7-1+b2_amd64.deb
-rw-r--r-- 1 sky sky   11M Apr 16 15:43 cri-dockerd_0.4.0.3-0.debian-bookworm_amd64.deb
-rw-r--r-- 1 sky sky   17M Apr 22 16:56 cri-tools_1.33.0-1.1_amd64.deb
-rw-r--r-- 1 sky sky  193K Dec 20  2022 ethtool_1%3a6.1-1_amd64.deb
-rw-r--r-- 1 sky sky   17M Apr 29 21:31 helm_3.17.3-1_amd64.deb
-rw-r--r-- 1 sky sky 1022K May 22  2023 iproute2_6.1.0-3_amd64.deb
-rw-r--r-- 1 sky sky  352K Jan 16  2023 iptables_1.8.9-2_amd64.deb
-rw-r--r-- 1 sky sky   13M Apr 24 02:07 kubeadm_1.33.0-1.1_amd64.deb
-rw-r--r-- 1 sky sky   12M Apr 24 02:07 kubectl_1.33.0-1.1_amd64.deb
-rw-r--r-- 1 sky sky   16M Apr 24 02:08 kubelet_1.33.0-1.1_amd64.deb
-rw-r--r-- 1 sky sky   37M Feb  5 17:03 kubernetes-cni_1.6.0-1.1_amd64.deb
-rw-r--r-- 1 sky sky  2.7M Mar  8 05:26 libc6_2.36-9+deb12u10_amd64.deb
-rw-r--r-- 1 sky sky  131K Dec  9 20:54 mount_2.38.1-5+deb12u3_amd64.deb
-rw-r--r-- 1 sky sky  1.2M Dec  9 20:54 util-linux_2.38.1-5+deb12u3_amd64.deb
```

将这个离线安装包压缩成一个 tar 包：

```bash
cd ~/temp/
tar -czvf k8s-offline-v1.33.tar.gz k8s-offline
```

## 离线安装

下载离线安装包到本地：

```bash
mkdir -p ~/temp/ && cd ~/temp/
scp sky@192.168.3.179:/home/sky/temp/k8s-offline-v1.33.tar.gz .
```

解压离线安装包：

```bash
tar -xvf k8s-offline-v1.33.tar.gz
cd k8s-offline
```

### 手工安装 cri-dockerd

```bash
sudo dpkg -i cri-dockerd*.deb 
```

### 手工安装 helm

```bash
sudo dpkg -i helm*.deb 
```

### 手工安装 kubeadm

安装 kubeadm 的依赖：

```bash
sudo dpkg -i util-linux*.deb
sudo dpkg -i conntrack*.deb
sudo dpkg -i cri-tools*.deb
sudo dpkg -i libc6*.deb
sudo dpkg -i ethtool*.deb
sudo dpkg -i mount*.deb
sudo dpkg -i iproute2*.deb
sudo dpkg -i iptables*.deb

sudo dpkg -i kubernetes-cni*.deb
```

安装 kubectl / kubelet / kubeadm ：

```bash
sudo dpkg -i kubectl*.deb
sudo dpkg -i kubelet*.deb
sudo dpkg -i kubeadm*.deb
```

修改 `~/.zshrc` 文件，添加 alias ：

```bash
if ! grep -qF "# k8s auto complete" ~/.zshrc; then
    cat >> ~/.zshrc << 'EOF'

# k8s auto complete
alias k=kubectl
complete -F __start_kubectl k
EOF
fi

source ~/.zshrc
```

## 制作离线安装脚本

离线安装避免在线安装的网络问题，非常方便，考虑写一个离线安装脚本，方便以后使用。

```bash
vi install_k8s_offline.zsh
```

内容为：

```bash
#!/usr/bin/env zsh

# ------------------------------------------------------------
# kubeadm 离线安装脚本 (Debian 12)
# 前提条件：
# 1. 所有 .deb 文件已放在 ~/k8s-offline
# 2. 已经
# ------------------------------------------------------------

set -e  # 遇到错误立即退出

K8S_OFFLINE_DIR="./k8s-offline"

# 检查是否在 Debian 12 上运行
if ! grep -q "Debian GNU/Linux 12" /etc/os-release; then
    echo "❌ 错误：此脚本仅适用于 Debian 12！"
    exit 1
fi

# 检查是否已安装 kubeadm
if command -v kubeadm &>/dev/null; then
    echo "⚠️ kubeadm 已安装，跳过安装步骤。"
    exit 0
fi

# 检查离线目录是否存在
if [[ ! -d "$K8S_OFFLINE_DIR" ]]; then
    echo "❌ 错误：离线目录 $K8S_OFFLINE_DIR 不存在！"
    exit 1
fi

echo "🔧 开始离线安装 kubeadm..."

# ------------------------------------------------------------
# 1. 开启模块
# ------------------------------------------------------------
echo "🔧 开启模块..."
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system


# ------------------------------------------------------------
# 2. 安装 cri-dockerd
# ------------------------------------------------------------
echo "📦 安装 cri-dockerd..."

cd "$K8S_OFFLINE_DIR"
sudo dpkg -i cri-tools*.deb
sudo dpkg -i cri-dockerd*.deb

# ------------------------------------------------------------
# 3. 安装 kubeadm 的依赖
# ------------------------------------------------------------
echo "📦 安装 kubeadm 的依赖包..."

# 按顺序安装依赖（防止 dpkg 报错）
for pkg in util-linux conntrack libc6 ethtool mount iproute2 iptables helm kubernetes-cni; do
    if ls "${pkg}"*.deb &>/dev/null; then
        echo "➡️ 正在安装: ${pkg}"
        sudo dpkg -i "${pkg}"*.deb || true  # 忽略部分错误，后续用 apt-get -f 修复
    fi
done

# 修复依赖关系
echo "🛠️ 修复依赖关系..."
sudo apt-get -f install -y

# ------------------------------------------------------------
# 4. 安装 kubeadm
# ------------------------------------------------------------
# 按顺序安装 kubeadm 组件（防止 dpkg 报错）
echo "📦 安装 kubeadm 组件..."
for pkg in kubectl kubelet kubeadm; do
    if ls "${pkg}"*.deb &>/dev/null; then
        echo "➡️ 正在安装: ${pkg}"
        sudo dpkg -i "${pkg}"*.deb || true  # 忽略部分错误，后续用 apt-get -f 修复
    fi
done

# 修复依赖关系
echo "🛠️ 修复依赖关系..."
sudo apt-get -f install -y


# ------------------------------------------------------------
# 5. 配置 kubectl
# ------------------------------------------------------------
echo "⚙️ 配置 kubectl 使用 alias..."

if ! grep -qF "# k8s auto complete" ~/.zshrc; then
    cat >> ~/.zshrc << 'EOF'

# k8s auto complete
alias k=kubectl
complete -F __start_kubectl k
EOF
fi

# ------------------------------------------------------------
# 6. 验证安装
# ------------------------------------------------------------
echo "✅ 安装完成！验证版本："
kubectl version --client && echo && kubelet --version && echo && kubeadm version && echo

echo "✨ kubeadm 安装完成！"
echo "👥 然后重新登录，或者执行命令以便 k alias 立即生效： source ~/.zshrc"
echo "🟢 之后请运行测试 kubectl 的别名 k： k version --client"

```


