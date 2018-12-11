---
date: 2018-12-08T09:00:00+08:00
title: Annotation
menu:
  main:
    parent: "introduction-object"
weight: 254
description : "Kubernetes对象的Annotation属性"
---

> 备注： 内容来自 [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)

 您可以使用Kubernetes annotation将任意非标识元数据附加到对象。工具和类库等客户端可以检索此元数据。

### 附加元数据到对象

您可以使用 label 或 annotation 将元数据附加到Kubernetes对象。 Label 可用于选择对象和查找满足特定条件的对象集合。 相反，annotation不用于识别和选择对象。 Annotation 中的元数据可大可小，结构化的或非结构化的，并且可以包括 label 不允许的字符。

Annotation（和 label 类似）是 key/value map：

```json
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

以下是一些可以在 annotation 中记录的信息示例：

- 由声明性配置层管理的字段。将这些字段作为 annotation 附加，可以将它们与以下情况区分开来：客户端或服务器设置的默认值，自动生成的字段，自动调整大小或自动伸缩系统设置的字段。

- 构建，发布或镜像信息，如时间戳，版本ID，git分支，PR编号，镜像哈希和注册表地址。

- 指向日志记录，监视，分析或审计存储库的指针。

- 可用于调试目的的客户端库或工具信息：例如，name, version和构建信息。

- 用户或工具/系统出处信息，例如来自其他生态系统组件的相关对象的URL。

- 轻量推出工具元数据：例如，配置或检查点。

- 负责人的电话或寻呼机号码，或指定可在何处找到该信息的目录条目，例如团队网站。

您可以将此类信息存储在外部数据库或目录中，而不是使用 annotation，但这会使得生成用于部署，管理，内省等的共享客户端类库和工具变得更加困难。









