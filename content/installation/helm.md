---
date: 2018-12-06T08:00:00+08:00
title: 安装helm
menu:
  main:
    parent: "installation"
weight: 140
description : "安装helm"
---

参考：

https://docs.helm.sh/using_helm/#installing-helm

步骤：

1. 下载需要的二进制文件

	https://github.com/helm/helm/releases

2. 解压缩

	```bash
	tar -zxvf helm-v2.0.0-linux-amd64.tgz
	```

3. 复制到/usr/local/bin

	```bash
	sudo mv linux-amd64/helm /usr/local/bin
	sudo mv linux-amd64/tiller /usr/local/bin
	```

4. 检验

	```bash
	helm version
	```

	