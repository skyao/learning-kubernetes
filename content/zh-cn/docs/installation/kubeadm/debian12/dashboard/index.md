---
title: "安装 dashboard"
linkTitle: "dashboard"
weight: 40
date: 2025-03-04
description: >
  安装 kubernetes 的 dashboard
---


## 安装 dashboard

在下面地址上查看当前dashboard的版本：

https://github.com/kubernetes/dashboard/releases

根据对kubernetes版本的兼容情况选择对应的dashboard的版本：

- kubernetes-dashboard-7.11.0  ，兼容 k8s 1.32

最新版本需要用 helm 进行安装：

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

输出为：

```bash
"kubernetes-dashboard" has been added to your repositories
Release "kubernetes-dashboard" does not exist. Installing it now.
NAME: kubernetes-dashboard
LAST DEPLOYED: Tue Mar  4 17:28:19 2025
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
