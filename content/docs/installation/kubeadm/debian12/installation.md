---
title: "在debian12上安装kubenetes"
linkTitle: "安装"
weight: 10
date: 2024-06-01
description: >
  在debian12上安装kubenetes
---



## 准备

### 系统更新

确保更新debian系统到最新，移除不再需要的软件，清理无用的安装包：

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt autoremove
sudo apt autoclean
```

如果更新了内核，最好重启一下。

### swap分区

安装 Kubernetes 要求机器不能有 swap 分区。

### 开启模块



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





### 安装 docker

卸载非官方版本 Docker：

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

安装：

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

启动 docker 并设置开机自动运行: 

```bash
sudo systemctl enable docker --now
```

### 安装 goalng 1.20 或者更高版本

这是为了给下面的手工 build cri-dockerd 做准备。

https://golang.org/dl/

下载最新版本的 golang。

```bash
mkdir -p ~/temp
mkdir -p ~/work/soft/gopath
cd ~/temp
wget https://go.dev/dl/go1.22.4.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.4.linux-amd64.tar.gz
```

修改

```bash
vi ~/.zshrc
```

加入以下内容：

```bash
export GOPATH=/home/sky/work/soft/gopath
export PATH=/usr/local/go/bin:$GOPATH/bin:$PATH
```

执行：

```bash
source ~/.zshrc

go version
go env
```



### 安装 cri-dockerd

注意需要先安装 goalng 1.20 或者更高版本。

```bash
mkdir -p ~/temp
cd ~/temp
git clone https://github.com/Mirantis/cri-dockerd.git
cd cri-dockerd
make cri-dockerd
sudo mkdir -p /usr/local/bin
sudo install -o root -g root -m 0755 cri-dockerd /usr/local/bin/cri-dockerd
sudo install packaging/systemd/* /etc/systemd/system
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
```



### 安装 helm

为后面安装 dashboard 做准备：

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```



## 安装 kubernetes

### 安装 kubeadm / kubelet / kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https
```

假定要安装的 kubernetes 版本为 1.29: 

```bash
export K8S_VERSION=1.29

curl -fsSL https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

```

开始安装 kubelet kubeadm kubectl：

```bash
sudo apt update
sudo apt install kubelet kubeadm kubectl -y
```

禁止这三个程序的自动更新：

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

验证安装：

```bash
kubectl version --client && echo && kubeadm version
Client Version: v1.29.6
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3

kubeadm version: &version.Info{Major:"1", Minor:"29", GitVersion:"v1.29.6", GitCommit:"062798d53d83265b9e05f14d85198f74362adaca", GitTreeState:"clean", BuildDate:"2024-06-11T20:22:13Z", GoVersion:"go1.21.11", Compiler:"gc", Platform:"linux/amd64"}
```

### 优化zsh

在 `~/.zshrc` 中增加以下内容:

```bash
# k8s auto complete
alias k=kubectl
complete -F __start_kubectl k
```

`source ~/.zshrc `之后即可使用，此时用 k 这个别名来执行 kubectl 命令时也可以实现自动完成，非常的方便。

### 取消自动更新

docker / helm / kubernetes 这些的版本没有必要升级到最新，因此可以取消他们的自动更新。

```bash
cd /etc/apt/sources.list.d
ls
docker.list  helm-stable-debian.list  kubernetes.list
```



### 初始化集群

```bash
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock --apiserver-advertise-address=192.168.6.224

```

输出为：

```bash
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock --apiserver-advertise-address=192.168.6.224
I0621 05:51:22.665581   20837 version.go:256] remote version is much newer: v1.30.2; falling back to: stable-1.29
[init] Using Kubernetes version: v1.29.6
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'

[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [debian12 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.6.224]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [debian12 localhost] and IPs [192.168.6.224 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [debian12 localhost] and IPs [192.168.6.224 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 3.500697 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node debian12 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node debian12 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: 1x7fi1.kjxn00med7dd3xwx
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
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

kubeadm join 192.168.6.224:6443 --token 1x7fi1.kjxn00med7dd3xwx \
	--discovery-token-ca-cert-hash sha256:51037fa4e37f485e10cb8ddfe8ec23e57d0dcd6698e5982f01449b6b6ca843e5
```

根据提示操作：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

此时节点的状态会是 NotReady：

```bash
kubectl get node  
NAME       STATUS     ROLES           AGE    VERSION
debian12   NotReady   control-plane   4m7s   v1.29.6
```

需要继续安装网络插件，可以选择 flannel 或者 Calico。

对于测试用的单节点，去除 master/control-plane 的污点：

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### (可选)安装 flannel

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

### (可选)安装 Calico

https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico

查看最新版本，当前最新版本是 v3.28:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

## 安装 dashboard

在下面地址上查看当前dashboard的版本：

https://github.com/kubernetes/dashboard/releases

根据对kubernetes版本的兼容情况选择对应的dashboard的版本：

- dashboard 7.5 ： 全面兼容 k8s 1.29

最新版本需要用 helm 进行安装：

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

输出为：

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
"kubernetes-dashboard" has been added to your repositories
Release "kubernetes-dashboard" does not exist. Installing it now.
NAME: kubernetes-dashboard
LAST DEPLOYED: Fri Jun 21 06:23:53 2024
NAMESPACE: kubernetes-dashboard
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*************************************************************************************************
*** PLEASE BE PATIENT: Kubernetes Dashboard may need a few minutes to get up and become ready ***
*************************************************************************************************

Congratulations! You have just installed Kubernetes Dashboard in your cluster.

To access Dashboard run:
  kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443

NOTE: In case port-forward command does not work, make sure that kong service name is correct.
      Check the services in Kubernetes Dashboard namespace using:
        kubectl -n kubernetes-dashboard get svc

Dashboard will be available at:
  https://localhost:8443
```

此时 dashboard 的 service 和 pod 情况：

```bash
kubectl -n kubernetes-dashboard get services 
NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                         AGE
kubernetes-dashboard-api               ClusterIP   10.107.22.93     <none>        8000/TCP                        17m
kubernetes-dashboard-auth              ClusterIP   10.102.201.198   <none>        8000/TCP                        17m
kubernetes-dashboard-kong-manager      NodePort    10.103.64.84     <none>        8002:30161/TCP,8445:31811/TCP   17m
kubernetes-dashboard-kong-proxy        ClusterIP   10.97.134.204    <none>        443/TCP                         17m
kubernetes-dashboard-metrics-scraper   ClusterIP   10.98.177.211    <none>        8000/TCP                        17m
kubernetes-dashboard-web               ClusterIP   10.109.72.203    <none>        8000/TCP                        17m
```

可以访问 http://ip:31811 来访问 dashboard 中的 kong manager。

以前的版本是要访问 kubernetes-dashboard service，现在新版本修改为要访问 kubernetes-dashboard-kong-proxy。

为了方便，使用 node port 来访问 dashboard，需要执行

```bash
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard-kong-proxy
```

然后修改 `type: ClusterIP` 为 `type: NodePort`。然后看一下具体分配的 node port 是哪个：

```bash
kubectl -n kubernetes-dashboard get service kubernetes-dashboard-kong-proxy
```

输出为：

```bash
$ kubectl -n kubernetes-dashboard get service kubernetes-dashboard-kong-proxy 

NAME                              TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard-kong-proxy   NodePort   10.97.134.204   <none>        443:30730/TCP   24m
```

直接就可以用浏览器直接访问：

https://192.168.0.101:30730/

### 创建用户并登录 dashboard

参考：[Creating sample user](https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md)

创建 admin-user 用户：

```bash
vi admin-user-ServiceAccount.yaml
```

内容为:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

执行：

```bash
k create -f admin-user-ServiceAccount.yaml
```

然后绑定角色：

``` bash
vi admin-user-ClusterRoleBinding.yaml
```

内容为:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

执行：

```bash
k create -f admin-user-ClusterRoleBinding.yaml
```

然后创建 token ：

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

输出为:

```bash
$ kubectl -n kubernetes-dashboard create token admin-user
eyJhbGciOiJSUzI1NiIsImtpZCI6IjdGczc3STI1VVA1OFpKdF9zektMVVFtZjd1NXRDRU8xTTZpZ1VYbDdKWFEifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzE5MDM2NDM0LCJpYXQiOjE3MTkwMzI4MzQsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJhZG1pbi11c2VyIiwidWlkIjoiNGY4YmQ3YjAtZjM2OS00MjgzLWJlNmItMThjNjUyMzE0YjQ0In19LCJuYmYiOjE3MTkwMzI4MzQsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDphZG1pbi11c2VyIn0.GOYLXoCCeaZPQ-kuJgx0d4KzRnLkHDHJArAjOwRqg49WIhAl3Hb8O2oD6at2jFgItO-xihFm3D3Ru2jXnPnMhvir0BJ5LBnumH0xDakZ4PrwvCAQADv8KR1ZuzMHlN5yktJ14eSo_UN1rZarq5P1DnbAIHRmgtIlRL2Hfl_Bamkuoxpwr06v50nJHskW7K3A2LjUlgv5rdS7FckIPaD5apmag7NyUi7FP1XEItUX20tF7jy5E5Gv9mI_HDGMTVMxawY4IAvipRcKVQ3tAypVOOMhrqGsfBprtWUkwmyWW8p0jHcAmqq-WX-x-vN70qI4Y2RipKGd4d6z39zPEPCsow
```

这个 token 就可以用在 kubernetes-dashboard 的登录页面上了。

为了方便，将这个 token 存储在 Secret ：

``` bash
vi admin-user-Secret.yaml
```

内容为:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "admin-user"   
type: kubernetes.io/service-account-token
```

执行：

```bash
k create -f admin-user-Secret.yaml
```

之后就可以用命令随时获取这个 token 了：

```bash
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
```

## 安装 metrics server

下载：

```bash
cd ~/work/soft/k8s
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

修改下载下来的 components.yaml， 增加 `--kubelet-insecure-tls` 并修改 `--kubelet-preferred-address-types`：

```yaml
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP   # 修改这行，默认是InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls  # 增加这行
```

然后安装：

```bash
k apply -f components.yaml
```

稍等片刻看是否启动:

```bash
$ kubectl get pod -n kube-system | grep metrics-server
```

验证一下，查看 service 信息

```bash
kubectl describe svc metrics-server -n kube-system
```

简单验证一下基本使用:

```bash
kubectl top nodes
kubectl top pods -n kube-system 
```





## 参考资料



- [Debian 12 安装 Kubernetes(k8s) + docker ，配置主从节点](https://acytoo.com/ladder/debian12-kubernetes-installation-and-config-master-worker/)

- [Install Kubernetes Cluster on Debian 12 (Bookworm) servers](https://computingforgeeks.com/install-kubernetes-cluster-on-debian-12-bookworm/)
