---
title: "初始化集群"
linkTitle: "初始化"
weight: 30
date: 2025-11-24
description: >
  在 debian13 上初始化 kubernetes 集群
---

参考官方文档：

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

## 初始化集群

pod-network-cidr 尽量用 10.244.0.0/16 这个范围，不然有些网络插件会需要额外的配置。

cri-socket 的配置参考：

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime

因为前面用的 Docker Engine 和 cri-dockerd ，因此这里的 cri-socket 需要指定为 "unix:///var/run/cri-dockerd.sock"。

apiserver-advertise-address 需要指定为当前节点的 IP 地址，因为当前节点是单节点，因此这里指定为当前机器的 IP 地址如 192.168.3.10。

```bash
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock --apiserver-advertise-address=192.168.3.100 --image-repository=192.168.3.193:5000/k8s-proxy
```

### pause 镜像的版本问题

遇到警告: 

```bash
W1127 11:14:12.643662   17241 checks.go:827] detected that the sandbox image "registry.k8s.io/pause:3.10" of the container runtime is inconsistent with that used by kubeadm. It is recommended to use "192.168.3.193:5000/k8s-proxy/pause:3.10.1" as the CRI sandbox image.
```

这个警告的意思是：容器运行时（CRI）默认使用的 sandbox 镜像版本和 kubeadm 期望的不一致。

Kubernetes 在运行 Pod 时，会用一个特殊的“pause”容器作为 Pod 的基础（sandbox）. 目前我安装的 CRI（cri-dockerd 或 containerd）默认拉的是：

```bash
registry.k8s.io/pause:3.10
```

而 kubeadm 在 v1.34.2 对应的版本期望的是： 

```bash
registry.k8s.io/pause:3.10.1
```

因为版本号不一致，就出现了这个警告。

解决方法, 修改 CRI 默认 sandbox 镜像, 对于 cri-dockerd, 需要在 cri-dockerd 的 systemd 启动参数里指定

```bash
sudo vi /lib/systemd/system/cri-docker.service
```

找到这行:

```bash
ExecStart=/usr/bin/cri-dockerd --container-runtime-endpoint fd://
```

在后面加入内容:

```bash
 --pod-infra-container-image=192.168.3.193:5000/k8s-proxy/pause:3.10.1
```

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart cri-docker
```

验证:

```bash
ps -ef | grep cri-dockerd

root       19302       1  0 11:40 ?        00:00:00 /usr/bin/cri-dockerd --container-runtime-endpoint fd:// --pod-infra-container-image=192.168.3.193:5000/k8s-proxy/pause:3.10.1
```

### coredns 的特殊处理

```bash
[sudo] password for sky: 
[init] Using Kubernetes version: v1.34.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
W1127 11:14:12.643662   17241 checks.go:827] detected that the sandbox image "registry.k8s.io/pause:3.10" of the container runtime is inconsistent with that used by kubeadm. It is recommended to use "192.168.3.193:5000/k8s-proxy/pause:3.10.1" as the CRI sandbox image.
[preflight] Some fatal errors occurred:
	[ERROR ImagePull]: failed to pull image 192.168.3.193:5000/k8s-proxy/coredns:v1.12.1: failed to pull image 192.168.3.193:5000/k8s-proxy/coredns:v1.12.1: Error response from daemon: failed to resolve reference "192.168.3.193:5000/k8s-proxy/coredns:v1.12.1": 192.168.3.193:5000/k8s-proxy/coredns:v1.12.1: not found
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
error: error execution phase preflight: preflight checks failed
To see the stack trace of this error execute with --v=5 or higher
```

这里 coredns 报错, 我们用 `kubeadm config images pull` 可以看到需要用到的镜像:

```bash
$ kubeadm config images pull

[config/images] Pulled registry.k8s.io/kube-apiserver:v1.34.2
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.34.2
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.34.2
[config/images] Pulled registry.k8s.io/kube-proxy:v1.34.2
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.12.1
[config/images] Pulled registry.k8s.io/pause:3.10.1
[config/images] Pulled registry.k8s.io/etcd:3.6.5-0
```

Kubernetes 大多数核心组件镜像（kube-apiserver、kube-scheduler、etcd、pause 等）都是单层路径，而 CoreDNS 特别用了两层路径 coredns/coredns。这会导致在使用代码仓库时出错:

```bash
$ kubeadm config images pull --image-repository=192.168.3.193:5000/k8s-proxy

[config/images] Pulled 192.168.3.193:5000/k8s-proxy/kube-apiserver:v1.34.2
[config/images] Pulled 192.168.3.193:5000/k8s-proxy/kube-controller-manager:v1.34.2
[config/images] Pulled 192.168.3.193:5000/k8s-proxy/kube-scheduler:v1.34.2
[config/images] Pulled 192.168.3.193:5000/k8s-proxy/kube-proxy:v1.34.2
error: failed to pull image "192.168.3.193:5000/k8s-proxy/coredns:v1.12.1": failed to pull image 192.168.3.193:5000/k8s-proxy/coredns:v1.12.1: Error response from daemon: failed to resolve reference "192.168.3.193:5000/k8s-proxy/coredns:v1.12.1": 192.168.3.193:5000/k8s-proxy/coredns:v1.12.1: not found
To see the stack trace of this error execute with --v=5 or higher
```

要修订这个问题,就需要告知 kubeadm coredns 的代理仓库

```bash
vi kubeadm.yaml
```

内容为:

```bash
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  criSocket: unix:///var/run/cri-dockerd.sock

---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.34.2
imageRepository: 192.168.3.193:5000/k8s-proxy
networking:
  podSubnet: 10.244.0.0/16
dns:
  imageRepository: 192.168.3.193:5000/k8s-proxy/coredns
  imageTag: v1.12.1
etcd:
  local:
    imageRepository: 192.168.3.193:5000/k8s-proxy
    imageTag: 3.6.5-0
```

### 执行 init

执行:

```bash
sudo kubeadm init --config=kubeadm.yaml
```

输出为：

```bash
W1127 11:54:29.969274    1961 common.go:100] your configuration file uses a deprecated API spec: "kubeadm.k8s.io/v1beta3" (kind: "ClusterConfiguration"). Please use 'kubeadm config migrate --old-config old-config-file --new-config new-config-file', which will write the new, similar spec using a newer API version.
W1127 11:54:29.969428    1961 common.go:100] your configuration file uses a deprecated API spec: "kubeadm.k8s.io/v1beta3" (kind: "InitConfiguration"). Please use 'kubeadm config migrate --old-config old-config-file --new-config new-config-file', which will write the new, similar spec using a newer API version.
[init] Using Kubernetes version: v1.34.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [debian13 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.3.100]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [debian13 localhost] and IPs [192.168.3.100 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [debian13 localhost] and IPs [192.168.3.100 127.0.0.1 ::1]
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
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/instance-config.yaml"
[patches] Applied patch of type "application/strategic-merge-patch+json" to target "kubeletconfiguration"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 501.044301ms
[control-plane-check] Waiting for healthy control plane components. This can take up to 4m0s
[control-plane-check] Checking kube-apiserver at https://192.168.3.100:6443/livez
[control-plane-check] Checking kube-controller-manager at https://127.0.0.1:10257/healthz
[control-plane-check] Checking kube-scheduler at https://127.0.0.1:10259/livez
[control-plane-check] kube-controller-manager is healthy after 1.030105473s
[control-plane-check] kube-scheduler is healthy after 1.587157411s
[control-plane-check] kube-apiserver is healthy after 3.000562739s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node debian13 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node debian13 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: npub0l.tdxgmyjagob3npq9
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

kubeadm join 192.168.3.100:6443 --token npub0l.tdxgmyjagob3npq9 \
	--discovery-token-ca-cert-hash sha256:366824f19b48b07d5e69fd2e928343b28b949cf161a5ff9ec10bf960ff4636c9
```

根据提示操作：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

对于测试用的单节点，去除 master/control-plane 的污点：

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

执行：

```bash
kubectl get node  
```

能看到此时节点的状态会是 NotReady：

```bash
NAME       STATUS     ROLES           AGE     VERSION
debian12   NotReady   control-plane   3m49s   v1.32.2
```

执行：

```bash
kubectl describe node debian12
```

能看到节点的错误信息：


```bash
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Tue, 04 Mar 2025 20:28:00 +0800   Tue, 04 Mar 2025 20:23:53 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Tue, 04 Mar 2025 20:28:00 +0800   Tue, 04 Mar 2025 20:23:53 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Tue, 04 Mar 2025 20:28:00 +0800   Tue, 04 Mar 2025 20:23:53 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Tue, 04 Mar 2025 20:28:00 +0800   Tue, 04 Mar 2025 20:23:53 +0800   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

需要继续安装网络插件。

## 安装网络插件

### 安装 flannel

参考官方文档： https://github.com/flannel-io/flannel#deploying-flannel-with-kubectl

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

如果一切正常，就能看到 k8s 集群内的 pod 都启动完成状态为 Running：

```bash
$ k get pods -A

NAMESPACE      NAME                               READY   STATUS    RESTARTS   AGE
kube-flannel   kube-flannel-ds-8p8b8              1/1     Running   0          39s
kube-system    coredns-848fbff4f8-phhgv           1/1     Running   0          2m27s
kube-system    coredns-848fbff4f8-wlc2s           1/1     Running   0          2m27s
kube-system    etcd-debian13                      1/1     Running   0          2m36s
kube-system    kube-apiserver-debian13            1/1     Running   0          2m36s
kube-system    kube-controller-manager-debian13   1/1     Running   0          2m36s
kube-system    kube-proxy-vknr2                   1/1     Running   0          2m28s
kube-system    kube-scheduler-debian13            1/1     Running   0          2m37s
```

如果发现 kube-flannel-ds pod 的状态总是 CrashLoopBackOff：

```bash
 k get pods -A
NAMESPACE      NAME                               READY   STATUS              RESTARTS        AGE
kube-flannel   kube-flannel-ds-ts6n8              0/1     CrashLoopBackOff    2 (22s ago)     42s
```

继续查看 pod 的具体错误信息：

```bash
k describe pods -n kube-flannel kube-flannel-ds-ts6n8
```

发现报错 "Back-off restarting failed container kube-flannel in pod kube-flannel"：

```bash
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  117s                default-scheduler  Successfully assigned kube-flannel/kube-flannel-ds-ts6n8 to debian12
  Normal   Pulled     116s                kubelet            Container image "ghcr.io/flannel-io/flannel-cni-plugin:v1.6.2-flannel1" already present on machine
  Normal   Created    116s                kubelet            Created container: install-cni-plugin
  Normal   Started    116s                kubelet            Started container install-cni-plugin
  Normal   Pulled     115s                kubelet            Container image "ghcr.io/flannel-io/flannel:v0.26.4" already present on machine
  Normal   Created    115s                kubelet            Created container: install-cni
  Normal   Started    115s                kubelet            Started container install-cni
  Normal   Pulled     28s (x5 over 114s)  kubelet            Container image "ghcr.io/flannel-io/flannel:v0.26.4" already present on machine
  Normal   Created    28s (x5 over 114s)  kubelet            Created container: kube-flannel
  Normal   Started    28s (x5 over 114s)  kubelet            Started container kube-flannel
  Warning  BackOff    2s (x10 over 110s)  kubelet            Back-off restarting failed container kube-flannel in pod kube-flannel-ds-ts6n8_kube-flannel(1e03c200-2062-4838
```

此时应该去检查准备工作中 "开启模块" 一节的内容是不是有疏漏。

补救之后，就能看到 kube-flannel-ds 这个 pod 正常运行了：

```bash
k get pods -A
NAMESPACE      NAME                               READY   STATUS    RESTARTS        AGE
kube-flannel   kube-flannel-ds-ts6n8              1/1     Running   7 (9m27s ago)   15m
```

### 安装 Calico

https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico

查看最新版本，当前最新版本是 v3.29.2:

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.2/manifests/tigera-operator.yaml
```

TODO：用了 flannel， Calico 后面再验证。 