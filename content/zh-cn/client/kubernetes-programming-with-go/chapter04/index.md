---
title: "[第4章]使用通用类型"
linkTitle: "使用通用类型"
weight: 40
date: 2021-02-01
description: >
  使用通用类型
---

上一章介绍了如何使用Go结构来定义Kubernetes资源。特别是，它解释了Kubernetes API 库的包的内容，以及与 Kubernetes Kind 相关的每个资源的常见字段。

本章研究了在定义 Kubernetes 资源时可在不同地方使用的常见类型。

## 指针

Go 结构体中的可选值通常被声明为指向一个值的指针。因此，如果你不想指定可选值，你只需要把它作为一个 nil 指针，如果你需要指定一个值，你必须创建该值并传递其引用。

Kubernetes Utils 库的 pointer 包定义了声明这种可选值的实用函数。

```go
import (
     "k8s.io/utils/pointer"
)
```

### 获取值的引用

Int, Int32, Int64, Bool, String, Float32, Float64, 和 Duration 函数接受一个相同类型的参数，并返回一个指向作为参数的值的指针。例如，Int32 函数的定义如下：

```go
func Int32(i int32) *int32 {
     return &i
}
```

然后，你可以这样使用它：

```go
spec := appsv1.DeploymentSpec{
     Replicas: pointer.Int32(3),
     [...]
}
```

### 对指针的解引用

另一种方法是，你可以得到指针所引用的值，或者在指针为零时得到一个默认值。

IntDeref、Int32Deref、Int64Deref、BoolDeref、StringDeref、Float32Deref、Float64Deref 和 DurationDeref 函数接受一个指针和一个默认值作为同一类型的参数，如果指针不为零，它就返回引用的值，否则就是默认值。例如，Int32Deref 函数定义如下：

```go
func Int32Deref(ptr *int32, def int32) int32 {
     if ptr != nil {
          return *ptr
     }
     return def
}
```

然后，你可以这样使用它：

```go
replicas := pointer.Int32Deref(spec.Replicas, 1)
```

### 比较两个引用的值

比较两个指针值可能很有用，考虑到如果它们都是 nil，或者它们引用了两个相等的值，那么它们就是相等的。

Int32Equal, Int64Equal, BoolEqual, StringEqual, Float32Equal, Float64Equal, 和 DurationEqual 函数接受两个相同类型的指针，如果这两个指针为 nil，或者它们引用的值相同，则返回真。例如，Int32Equal 函数定义如下：

```go
func Int32Equal(a, b *int32) bool {
     if (a == nil) != (b == nil) {
          return false
     }
     if a == nil {
          return true
     }
     return *a == *b
}
```

然后，你可以这样使用它：

```go
eq := pointer.Int32Equal(
     spec1.Replicas,
     spec2.Replicas,
)
```

请注意，要测试一个可选值的平等性，同时考虑其默认值，你应该使用以下方法：

```go
isOne := pointer.Int32Deref(spec.Replicas, 1) == 1
```

## Quantities

Quantities 是一个数字的定点（fixed-point）表示，用于定义要分配的资源数量（例如，内存、CPU等）。Quantities 能代表的最小值是一纳米（10-9）。

在内部，Quantities 由一个Integer（64位）和一个 Scale 表示，或者，如果 int64 不够大，则由一个 inf.Dec 值表示（由软件包定义在 https://github.com/go-inf/inf ）

```go
import (
     "k8s.io/apimachinery/pkg/api/resource"
)
```

### 将字符串解析为数量

定义 **Quantity** 的第一种方法是通过使用以下函数之一从字符串中解析其值：

- `func MustParse(str string) Quantity` - 从字符串中解析出 Quantity，如果字符串不代表一个 Quantity，则会出现 panic。当你应用一个你知道是有效的硬编码值时，就可以使用它。
- `func ParseQuantity(str string) (Quantity, error) `- 从字符串中解析 Quantity，如果字符串不代表一个 Quantity，则返回错误。当你不确定该值是否有效时，可以使用该方法。

Quantity 可以用一个符号、一个数字和一个后缀来写。符号和后缀是可选的。后缀可以是二进制，十进制，或十进制指数。

定义的二进制后缀是 Ki (210), Mi (220), Gi (230), Ti (240), Pi (250) 和  Ei (260)。定义的十进制后缀是 n（10-9），u（10-6），m（10-3），""（100），k（103），M（106），G（109），T（1012），P（1015），和E（1018）。十进制指数后缀用 e 或 E 符号书写，后面是十进制指数--例如，E2代表102。

后缀格式（二进制、十进制或十进制指数）被保存在 Quantity 中，并在序列化 Quantity 时使用。

使用这些函数，数量的内部表示方式（要么是按比例的整数，要么是inf.Dec）将根据解析的值是否可以表示为按比例的整数来决定。

### 使用 inf.Dec 作为Quantity

使用下面的方法来使用 inf.Dec 作为一个Quantity：

- `func NewDecimalQuantity(b inf.Dec, format Format) *Quantity` - 通过给出一个 `inf.Dec` 值，并指出你希望它被序列化的后缀格式来声明 Quantity。
- `func (q *Quantity) ToDec() *Quantity` - 通过解析一个字符串或使用一个新的函数来初始化一个 Quantity，强制将它存储为一个 inf.Dec。
- `func (q *Quantity) AsDec() *inf.Dec` - 获得一个作为 `inf.Dec` 的 Quantity 表示，而不修改内部表示。

### 使用带刻度的整数作为Quantity

使用下面的方法来使用一个带尺度的整数作为 Quantity：

- `func NewScaledQuantity(value int64, scale Scale) *Quantity` - 通过给出一个 int64 值和一个刻度来声明 Quantity。后缀的格式将是十进制格式。
- `func (q *Quantity) SetScaled(value int64, scale Scale) ` - 用一个带刻度的整数覆盖 Quantity 值。后缀的格式将保持不变。
- `func (q *Quantity) ScaledValue(scale Scale) int64` - 获得带刻度的整数表示，考虑到给定的刻度，不修改内部表示。
- `func NewQuantity(value int64, format Format) *Quantity` - 通过给出一个 int64 的值来声明 Quantity，刻度被固定为0，并且在序列化过程中使用一个后缀格式。
- `func (q *Quantity) Set(value int64)` - 用一个整数和一个固定为0的刻度来重写 Quantity 值，后缀的格式将保持不变。
- `func (q *Quantity) Value() int64` - 获得一个数量的整数表示，比例为0，不修改内部表示。
- func NewMilliQuantity(value int64, format Format) *Quantity - 通过给出一个int64的值来声明 Quantity，比例固定为-3，后缀格式在序列化时使用。

- func (q *Quantity) SetMilli(value int64) - 用整数和固定为-3的刻度重写一个 Quantity 值。

- func (q *Quantity) MilliValue() int64 - 得到 Quantity 的整数表示，比例为-3，不需要修改内部表示。

### 对Quantity的操作

以下是对 "数量 "进行操作的方法：

- func (q *Quantity) Add(y Quantity) - 将 y Quantity加入到 q Quantity 中。
- func (q *Quantity) Sub(y Quantity) - 从q数量中减去y数量。

- func (q *Quantity) Cmp(y Quantity) int - 比较q和y的数量。如果两个数量相等返回0，如果q大于y返回1，如果q小于y返回-1。

- func (q *Quantity) CmpInt64(y int64) int - 比较q数量和y的整数。如果两个数量相等，返回0；如果q大于y，返回1；如果q小于y，返回1。

- func (q *Quantity) Neg() - 使q成为它的负值。

- func (q 数量) Equal(v 数量) bool - 测试q和v数量是否相等。

## IntOrString

Kubernetes 资源的一些字段接受整数值或字符串值。例如，端口可以用端口号或 IANA 服务名称来定义（如https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml）。

另一个例子是可以接受整数或百分比的字段。

```go
import (
     "k8s.io/apimachinery/pkg/util/intstr"
)
```

IntOrString结构定义如下：

```go
type IntOrString struct {
     Type   Type
     IntVal int32
     StrVal string
}
```

TBD

## Time

TBD

## 总结

本章介绍了在Go中定义Kubernetes资源时使用的常见类型。指针值用于指定可选的值，数量用于指定内存和CPU数量，IntOrString类型用于可写为整数或字符串的值（例如，可以通过数字或名称定义的端口），以及时间--一种可序列化的类型，包装Go时间类型。