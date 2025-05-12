---
title: "初始化集群"
linkTitle: "初始化"
weight: 30
date: 2025-03-04
description: >
  在 debian12 上初始化 kubernetes 集群
---

参考官方文档：

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

## 初始化集群

pod-network-cidr 尽量用 10.244.0.0/16 这个范围，不然有些网络插件会需要额外的配置。

cri-socket 的配置参考：

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-runtime

因为前面用的 Docker Engine 和 cri-dockerd ，因此这里的 cri-socket 需要指定为 "unix:///var/run/cri-dockerd.sock"。

apiserver-advertise-address 需要指定为当前节点的 IP 地址，因为当前节点是单节点，因此这里指定为 192.168.3.179。

```bash
sudo kubeadm init --pod-network-cidr 10.244.0.0/16 --cri-socket unix:///var/run/cri-dockerd.sock --apiserver-advertise-address=192.168.3.179 --image-repository=192.168.3.91:5000/k8s-proxy
```

输出为：

```bash
W0511 22:22:37.653053    1276 version.go:109] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.txt": Get "https://dl.k8s.io/release/stable-1.txt": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
W0511 22:22:37.653104    1276 version.go:110] falling back to the local client version: v1.33.0
[init] Using Kubernetes version: v1.33.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [debian12 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.3.179]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [debian12 localhost] and IPs [192.168.3.179 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [debian12 localhost] and IPs [192.168.3.179 127.0.0.1 ::1]
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
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests"
[kubelet-check] Waiting for a healthy kubelet at http://127.0.0.1:10248/healthz. This can take up to 4m0s
[kubelet-check] The kubelet is healthy after 501.341993ms
[control-plane-check] Waiting for healthy control plane components. This can take up to 4m0s
[control-plane-check] Checking kube-apiserver at https://192.168.3.179:6443/livez
[control-plane-check] Checking kube-controller-manager at https://127.0.0.1:10257/healthz
[control-plane-check] Checking kube-scheduler at https://127.0.0.1:10259/livez
[control-plane-check] kube-controller-manager is healthy after 1.002560331s
[control-plane-check] kube-scheduler is healthy after 1.156287353s
[control-plane-check] kube-apiserver is healthy after 2.500905726s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node debian12 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node debian12 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: 5dlwbv.j26vzkb6uvf9yqv6
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

kubeadm join 192.168.3.179:6443 --token 5dlwbv.j26vzkb6uvf9yqv6 \
	--discovery-token-ca-cert-hash sha256:0d1be37706d728f6c09dbcff86614c6fe04c536d969371400f4d3551f197c6e4
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
k get pods -A
NAMESPACE      NAME                               READY   STATUS    RESTARTS        AGE
kube-flannel   kube-flannel-ds-ts6n8              1/1     Running   7 (9m27s ago)   15m
kube-system    coredns-668d6bf9bc-rbkzb           1/1     Running   0               3h55m
kube-system    coredns-668d6bf9bc-vbltg           1/1     Running   0               3h55m
kube-system    etcd-debian12                      1/1     Running   0               3h55m
kube-system    kube-apiserver-debian12            1/1     Running   1 (5h57m ago)   3h55m
kube-system    kube-controller-manager-debian12   1/1     Running   0               3h55m
kube-system    kube-proxy-95ccr                   1/1     Running   0               3h55m
kube-system    kube-scheduler-debian12            1/1     Running   1 (6h15m ago)   3h55m
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