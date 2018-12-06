---
date: 2018-12-06T08:00:00+08:00
title: 快速开启dashboard
menu:
  main:
    parent: "addons-dashboard"
weight: 511
description : "快速安装配置dashboard"
---

在不考虑安全和生产可以的情况下， 比如开发，测试和学习时，可以快速的安装配置dashboard，跳过安全相关的复杂配置。

参考信息：

- [官方安装文档](https://github.com/kubernetes/dashboard/wiki/Installation): 
- [README @ github](https://github.com/kubernetes/dashboard/blob/master/README.md#getting-started): 这里有快速安装的介绍

### 安装Dashboard

执行下面命令，安装dashboard：

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

### 配置权限

参考 [Access Control](https://github.com/kubernetes/dashboard/wiki/Access-control) 文档中的 [Admin privileges](https://github.com/kubernetes/dashboard/wiki/Access-control#admin-privileges) 一节，用最简单（当然最不安全）的方式。

新建 dashboard-admin.yaml 文件，内容为：

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```

然后执行 `kubectl create -f dashboard-admin.yaml` 命令。

在后面访问dashboard时，在login界面，可以点 "Skip" 跳过配置，直接使用。 

### 访问方式

访问dashboard最简单的方式是通过 `kubectl proxy`:

```bash
kubectl proxy
```

然后就可以使用浏览器通过下列的地址访问dashboard：

http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/

注意此时是用的localhost地址，因此只有在执行 `kubectl proxy` 命令的机器上才能访问。如果需要暴露给其他机器，需要指定address:

```bash
$ kubectl proxy --address='0.0.0.0' --port=8001 --accept-hosts='^*$'
Starting to serve on [::]:8001
```

然后就可以在其他机器上通过这个机器的ip地址访问了：

http://192.168.0.10:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/#!/namespace?namespace=default



