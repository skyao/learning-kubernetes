---
date: 2018-12-08T09:00:00+08:00
title: Label和选择器
menu:
  main:
    parent: "introduction-object"
weight: 253
description : "Kubernetes对象的Label和选择器"
---

> 备注： 内容来自 [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

标签是附加到对象（例如pod）的键/值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接影响核心系统的语义。 标签可用于组织和选择对象的子集。 标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。 每个Key对于给定对象必须是唯一的。

```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

我们最终将索引和反向索引标签以用于高效查询和监视，使用它们在UI和CLI中进行排序和分组等。我们不希望使用具有非标识，特别是大型和/或结构化的数据来污染标签。 非识别信息应使用 annotation 记录。 

### 动机

标签使用户能够以松散耦合的方式将他们自己的组织结构映射到系统对象，而无需客户端存储这些映射。

服务部署和批处理流水线通常是多维实体（例如，多个分区或部署，多个发布轨道，多个层，每层多个微服务）。管理通常需要交叉操作，这打破了严格的层次表示的封装，特别是由基础设施而不是用户确定的严格的层次结构。

示例标签：

- `"release" : "stable"`, `"release" : "canary"`
- `"environment" : "dev"`, `"environment" : "qa"`, `"environment" : "production"`
- `"tier" : "frontend"`, `"tier" : "backend"`, `"tier" : "cache"`
- `"partition" : "customerA"`, `"partition" : "customerB"`
- `"track" : "daily"`, `"track" : "weekly"`

这些只是常用标签的例子;你可以自由地制定自己的约定。请记住，标签Key对于给定对象必须是唯一的。

### 语法和字符集

标签是键/值对。有效标签键有两个段：可选前缀和名称，用斜杠（/）分隔。名称段是必需的，必须是63个字符或更少，以字母数字字符（[a-z0-9A-Z]）开头和结尾，中间有破折号（ - ），下划线（_），点（.）和字母数字。前缀是可选的。如果指定，前缀必须是DNS子域：由点（.）分隔的一系列DNS标签，总共不超过253个字符，后跟斜杠（/）。如果省略前缀，则假定标签Key对用户是私有的。向最终用户对象添加标签的自动系统组件（例如，kube-scheduler，kube-controller-manager，kube-apiserver，kubectl或其他第三方自动化）必须指定前缀。 `kubernetes.io/`前缀保留给Kubernetes核心组件。

有效标签值必须为63个字符或更少，并且必须以字母数字字符（[a-z0-9A-Z]）开头和结尾，中间有破折号（ - ），下划线（_），点（.）和字母数字。

### 标签选择器

与名称和UID不同，标签不提供唯一性。通常，我们希望许多对象携带相同的标签。

通过标签选择器，客户端/用户可以识别一组对象。标签选择器是Kubernetes中的核心分组原语。

API目前支持两种类型的选择器：基于等式和基于集合。标签选择器可以由逗号分隔的多个条件组成。 在多个条件的情况下，必须满足所有要求，因此逗号分隔符充当逻辑AND（&&）运算符。

空标签选择器（即，条件为零的选择器）选择集合中的每个对象。

空标签选择器（仅可用于可选选择器字段）不选择任何对象。

> 注意：两个控制器的标签选择器不得在命名空间内重叠，否则它们将相互冲突。

#### 基于等式的条件

基于相等或不相等的条件允许按标签键和值进行过滤。匹配对象必须满足所有指定的标签约束，尽管它们也可能具有其他标签。 允许三种运算符=，==，！=。 前两个代表相等（简单地说是同义词），而后者代表不相等。 例如：

```bash
environment = production
tier != frontend
```

前者选择 key 等于 `environment` 而值等于 `production` 的所有资源。后者选择 key 等于 `tier` 和值与 `frontend` 不同的所有资源，以及所有没有带 `tier` 键标签的资源。可以使用逗号运算符过滤在 `production` 中而又不在 `frontend` 的资源：`environment=production,tier!=frontend`

基于相等的标签条件的一种使用场景是用于 Pods 指定节点选择标准。例如，下面的示例 Pod 选择带有标签“`accelerator=nvidia-tesla-p100`”的节点。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-test
spec:
  containers:
    - name: cuda-test
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
  nodeSelector:
    accelerator: nvidia-tesla-p100
```

#### 基于集合的条件

基于集合的标签要求允许根据一组值过滤key。支持三种运算符：in，notin和exists（仅key标识符）。 例如：

```bash
environment in (production, qa)
tier notin (frontend, backend)
partition
!partition
```

第一个示例选择 key 等于 `environment` 且值等于 `production` 或 `qa` 的所有资源。第二个示例选择 key 等于 `tier` 而值不是 `frontend` 或者 `backend` 的所有资源，以及没有带 key 为 `tier` 的标签的所有资源。第三个例子选择带有key为`partition` 的标签的所有资源，不检查值。第四个示例选择没有带key为`partition` 的标签的所有资源，不检查值。类似地，逗号分隔符充当AND运算符。因此，使用 `partition,environment notin (qa)` 可以实现这样的资源过滤：带有 `partition`  key（无论值是多少）和 带有`environment` key而值不是 `qa` 。基于集合的标签选择器是一般的相等形式，因为`environment=production`等同于`environment in (production)`; 同样的`!=`和`notin`也是如此。

基于集合的条件可以与基于相等的条件相结合。 例如：`partition in (customerA, customerB),environment!=qa`。

### API

#### 列出和查看过滤

LIST和WATCH操作可以指定标签选择器来过滤使用查询参数返回的对象集。以下两个条件都容许（在此处显示为出现在URL查询字符串中）：

- 基于等式的条件: ?labelSelector=environment%3Dproduction,tier%3Dfrontend
- 基于集合的条件: ?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29

两种标签选择器样式都可用于通过REST客户端列出或查看资源。例如，使用 `kubectl` 定位 `apiserver` 并使用基于等式的条件可能会编写为：

```bash
$ kubectl get pods -l environment=production,tier=frontend
```

或者使用基于集合的条件：

```bash
$ kubectl get pods -l 'environment in (production),tier in (frontend)'
```

如前所述，基于集合的条件更具表现力。 例如，他们可以在值上实现OR运算符：

```bash
$ kubectl get pods -l 'environment in (production, qa)'
```

或通过 `exists` 运算符限制不匹配：

```bash
$ kubectl get pods -l 'environment,environment notin (frontend)'
```

#### 在API对象中设置引用

某些Kubernetes对象（例如 `service` 和 `replicationcontrollers`）也使用标签选择器来指定其他资源集，例如`pod`。

##### 服务和副本控制器

服务所针对的一组pod使用标签选择器进行定义。类似地，副本控制器应该管理的pod的数量也使用标签选择器定义。

两个对象的标签选择器都是在 `json` 或 `yaml` 文件中定义的，并且只支持基于等式的条件选择器：

```json
"selector": {
    "component" : "redis",
}
```

或者

```json
selector:
    component: redis
```

这个选择器（分别以`json`或`yaml`格式）等同于`component=redis`或 `component in (redis)`。

### 支持基于集合的条件的资源

较新的资源（如Job, Deployment, Replica Set, 和 Daemon Set,）支持基于集合的条件。

```bash
selector:
  matchLabels:
    component: redis
  matchExpressions:
    - {key: tier, operator: In, values: [cache]}
    - {key: environment, operator: NotIn, values: [dev]}
```

`matchLabels`是 `{key，value}` 对的map。 `matchLabels` map 中的单个 `{key，value}` 等同于 `matchExpressions` 的元素，其 key 字段为“key”，运算符为“In”，values 数组仅包含 “value”。 `matchExpressions`是pod选择器的条件列表。有效的运算符包括In，NotIn，Exists和DoesNotExist。 在In和NotIn的情况下，设置的值必须是非空的。 来自matchLabels和matchExpressions的所有条件都是 AND 在一起 - 它们必须全部满足才能匹配。

#### 选择节点集合

使用选择标签的一个用例是约束pod可以调度的节点集。有关更多信息，请参阅有关 [节点选择](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/) 的文档。









