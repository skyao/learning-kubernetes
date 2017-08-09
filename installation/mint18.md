# linux mint 18.1 安装

参考官方介绍的 ubuntu 安装方法:

- [Kubernetes on Ubuntu](https://www.ubuntu.com/containers/kubernetes)
- [The Canonical Distribution of Kubernetes](https://www.ubuntu.com/cloud/kubernetes)

## 安装准备

1. 安装snap

	安装方式参考下文:

	https://skyao.gitbooks.io/learning-linux-mint/admin/snap/

2. 安装conjure-up

	```bash
    sudo snap install conjure-up --classic
    ```

    重新登录(或者重启，否则 conjure-up 会报错说找不到 conjure-up 命令)，

## 安装 kubernetes

参考这里的步骤：

https://kubernetes.io/docs/getting-started-guides/ubuntu/

执行命令:

```bash
conjure-up kubernetes
```

弹出窗口，选择安装类型，第一个选项是"Kubernetes Core":

![](images/mint18-1.jpg)

第二个选项是"The Canonical Distribution of Kubernetes":

![](images/mint18-22.jpg)

简单测试用选 "Kubernetes Core"，回车，然后选择云，这里我们选择"localhost" 来创建一个新的云：

![](images/mint18-2.jpg)

陆续出现了很多错误，

1. 报错说版本不对

	![](images/mint18-3.jpg)

	这个错误是用了 `snappy-dev/edge` 这个ppa, 安装上去的 snap 和 snapd 的版本大于 2.25但是也会如上报错。

    解决的方法就是卸载 snap 然后修改ppa 为　`ppa:snappy-dev/tools` 再重新安装。

2. lxd设置

	![](images/mint18-32.jpg)

    安装提示，执行命令:

    ```bash
    sudo usermod -a -G lxd $USER
    newgrp lxd
    ```

3. lxd权限不够

    ![](images/mint18-33.jpg)

    这个是当前账号的group不对，重启就好了。

解决上面的问题后，就可以继续安装，

![](images/mint18-4.jpg)

可以参考下文：

https://kubernetes.io/docs/getting-started-guides/ubuntu/local/

![](images/mint18-5.jpg)

耐心等待，安装很慢，大概要等5-10分钟。(TBD: 怀疑是忘了设置代理，可能下载镜像太慢)

![](images/mint18-52.jpg)

![](images/mint18-53.jpg)