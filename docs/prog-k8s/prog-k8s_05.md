# 第五章：自动化代码生成

在本章中，您将学习如何在 Go 项目中使用 Kubernetes 代码生成器以一种自然的方式编写自定义资源。代码生成器在实现本机 Kubernetes 资源时被广泛使用，我们也将在这里使用完全相同的生成器。

# 为什么要代码生成

Go 是一种设计简单的语言。它缺乏高级或者类似元编程的机制来以通用的方式表达对不同数据类型的算法。在 Go 中，“Go way” 是使用外部代码生成。

Kubernetes 开发的早期阶段，随着系统增加更多资源，越来越多的代码需要重写。代码生成大大简化了这些代码的维护。早期，创建了[Gengo 库](http://bit.ly/2L9kwNJ)，并最终基于 Gengo 开发了[*k8s.io/code-generator*](http://bit.ly/2Kw8I8U)，作为一个可以外部使用的生成器集合。我们将在接下来的章节中使用这些生成器来处理 CRs。

# 调用生成器

通常情况下，几乎每个控制器项目中调用代码生成器的方式都是大致相同的。只有包、组名和 API 版本不同。从构建系统调用 *k8s.io/code-generator/generate-groups.sh* 或类似 *hack/update-codegen.sh* 的 bash 脚本是向 CR Go 类型添加代码生成的最简单方法（参见[该书的 GitHub 仓库](http://bit.ly/2J0s2YL)）。

请注意，由于非常特殊的需求和历史原因，一些项目直接调用生成器二进制文件。对于构建 CRs 控制器的用例，直接从 *k8s.io/code-generator* 仓库调用 *generate-groups.sh* 脚本要简单得多：

```
$ vendor/k8s.io/code-generator/generate-groups.sh all \
    github.com/programming-kubernetes/cnat/cnat-client-go/pkg/generated
    github.com/programming-kubernetes/cnat/cnat-client-go/pkg/apis \
    cnat:v1alpha1 \
    --output-base "${GOPATH}/src" \
    --go-header-file "hack/boilerplate.go.txt"
```

在这里，`all` 表示调用所有四个用于 CRs 的标准代码生成器：

`deepcopy-gen`

生成 `func` `(t *T)` `DeepCopy()` `*T` 和 `func` `(t *T)` `DeepCopyInto(*T)` 方法。

`client-gen`

创建类型化的客户端集。

`informer-gen`

为 CRs 创建 informers，提供基于事件的接口以便在服务器上对 CRs 的更改做出响应。

`lister-gen`

为 CRs 创建 listers，为 `GET` 和 `LIST` 请求提供只读缓存层。

最后两个方法是构建控制器的基础（参见“控制器和操作员”）。这四个代码生成器构成了使用相同机制和包构建功能齐全、生产就绪的控制器的强大基础，与 Kubernetes 上游控制器使用的相同。

###### 注意

*k8s.io/code-generator* 中还有一些其他生成器，主要用于其他情境。例如，如果您构建自己的聚合 API 服务器（参见第八章），您将同时使用内部类型和版本化类型，并且必须定义默认函数。那么，您可以通过调用 *k8s.io/code-generator/generate-internal-groups.sh* 脚本来访问这两个生成器，它们将变得相关：

`conversion-gen`

创建用于在内部类型和外部类型之间进行转换的函数。

`defaulter-gen`

处理某些字段的默认值。

现在让我们详细看看 `generate-groups.sh` 的参数：

+   生成的客户端、列表和通知者的目标包名是第二个参数。

+   API 组的基础包是第三个参数。

+   第四个参数是 API 组与其版本的空格分隔列表。

+   `--output-base` 作为一个标志传递给所有生成器，用来定义给定包所在的基础目录。

+   `--go-header-file` 允许我们将版权头放入生成的代码中。

一些生成器，比如 `deepcopy-gen`，直接在 API 组包内部创建文件。这些文件遵循一个以 *zz_generated.* 前缀命名的标准命名方案，因此很容易通过版本控制系统（例如 *.gitignore* 文件）排除它们，尽管大多数项目决定检查生成的文件，因为围绕代码生成器的 Go 工具尚未很好地发展。^(1)

如果项目遵循 [*k8s.io/sample-controller*](http://bit.ly/2UppsTN) 的模式 —— `sample-controller` 是一个蓝图项目，复制了 Kubernetes 自身构建的许多控制器的模式 —— 那么代码生成从以下内容开始：

```
$ hack/update-codegen.sh
```

在 “Following sample-controller” 中的 `sample-controller+client-go` 变体中，`cnat` 示例沿着这条路线前进。

###### 提示

通常，除了 [`hack/update-codegen.sh`](http://bit.ly/2J0s2YL) 脚本之外，还有一个名为 [`hack/verify-codegen.sh`](http://bit.ly/2IXUWsy) 的第二个脚本。

此脚本调用 `hack/update-codegen.sh` 脚本并检查是否有任何更改，如果生成的文件中有任何文件不是最新的，则以非零返回码终止。

在持续集成（CI）脚本中非常有帮助：如果开发人员意外修改了文件或者文件仅仅是过时的，CI 将注意到并提出投诉。

# 使用标签控制生成器

虽然某些代码生成器的行为是通过命令行标志控制的（如前面描述的特别是要处理的包），但更多的属性是通过 *标签* 在您的 Go 文件中控制的。标签是以下格式的特殊格式化 Go 注释：

```
// +some-tag
// +some-other-tag=value
```

有两种类型的标签：

+   文件 *doc.go* 中 `package` 行上方的全局标签

+   类型声明（例如结构定义）之前的本地标签

根据问题中的标签，注释的位置可能很重要。

## 全局标签

全局标签写入包的 *doc.go*。一个典型的 *pkg/apis/`group`/`version`/doc.go* 文件如下所示：

```
// +k8s:deepcopy-gen=package

// Package v1 is the v1alpha1 version of the API.
// +groupName=cnat.programming-kubernetes.info
package v1alpha1
```

此文件的第一行告诉`deepcopy-gen`默认为该包中的每种类型创建深度复制方法。如果您有不需要、不希望或甚至不可能进行深度复制的类型，您可以通过本地标签`// +k8s:deepcopy-gen=false`为其选择退出。如果您没有启用包范围的深度复制，您必须通过`// +k8s:deepcopy-gen=true`为每种所需类型选择深度复制。

第二个标签，`// +groupName=example.com`，定义了完全限定的 API 组名。如果 Go 父包名称与组名不匹配，则此标签是必需的。

这里显示的文件实际上来自[`cnat client-go`示例 *pkg/apis/cnat/v1alpha1/doc.go*文件](http://bit.ly/2L6M9ad)（请参阅“跟踪示例控制器”）。在那里，`cnat`是父包，但`cnat.programming-kubernetes.info`是组名。

使用`// +groupName`标签，客户端生成器（请参阅“通过 client-gen 创建的类型化客户端”）将生成一个使用正确 HTTP 路径 */apis/foo.project.example.com*的客户端。除了`+groupName`之外，还有`+groupGoName`，它定义了一个自定义的 Go 标识符（用于变量和类型名称），用于替代父包名称。例如，生成器默认使用大写的父包名称作为标识符，在我们的示例中是`Cnat`。一个更好的标识符将是`CNAt`，代表“Cloud Native At”。使用`// +groupGoName=CNAt`，我们可以使用它而不是`Cnat`（尽管在此示例中我们没有这样做——我们仍然使用了`Cnat`），`client-gen`生成的结果将如下所示：

```
type Interface interface {
    Discovery() discovery.DiscoveryInterface
    CNatV1() atv1alpha1.CNatV1alpha1Interface
}
```

## 本地标签

本地标签可以直接写在 API 类型的上方，也可以写在其上方的第二个注释块中。以下是[`cnat`示例](http://bit.ly/31QosJw)中*types.go*文件中的主要类型：

```
// AtSpec defines the desired state of At
type AtSpec struct {
    // Schedule is the desired time the command is supposed to be executed.
    // Note: the format used here is UTC time https://www.utctime.net
    Schedule string `json:"schedule,omitempty"`
    // Command is the desired command (executed in a Bash shell) to be executed.
    Command string `json:"command,omitempty"`
    // Important: Run "make" to regenerate code after modifying this file
}

// AtStatus defines the observed state of At
type AtStatus struct {
    // Phase represents the state of the schedule: until the command is executed
    // it is PENDING, afterwards it is DONE.
    Phase string `json:"phase,omitempty"`
    // Important: Run "make" to regenerate code after modifying this file
}

// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// At runs a command at a given schedule.
type At struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   AtSpec   `json:"spec,omitempty"`
    Status AtStatus `json:"status,omitempty"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// AtList contains a list of At
type AtList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []At `json:"items"`
}
```

在以下各节中，我们将详细介绍本示例的标签。

###### 提示

在这个示例中，API 文档位于第一个注释块中，而我们将标签放在第二个注释块中。如果您使用某些工具来提取 Go doc 注释，这有助于将标签排除在 API 文档之外。

## deepcopy-gen 标签

通常通过全局`// +k8s:deepcopy-gen=package`标签为所有类型启用深度复制方法（请参阅“全局标签”），即通过可能的选择退出。然而，在前述示例文件中（实际上是整个包中），所有 API 类型都需要深度复制方法。因此，我们不必在本地选择退出。

如果我们在 API 类型包中有一个辅助结构体（通常不鼓励这样做以保持 API 包的清洁），我们将不得不禁用深度复制生成。例如：

```
// +k8s:deepcopy-gen=false
//
// Helper is a helper struct, not an API type.
type Helper struct {
    ...
}
```

## runtime.Object 和 DeepCopyObject

有一个需要更多解释的特殊深度复制标签：

```
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
```

在《Go 中的 Kubernetes 对象》中，我们看到`runtime.Object`必须实现`DeepCopyObject() runtime.Object`方法。原因是 Kubernetes 内部的通用代码必须能够创建对象的深拷贝。这个方法允许了这一点。

`DeepCopyObject()`方法除了调用生成的`DeepCopy`方法外什么也不做。后者的签名因类型而异（`DeepCopy()` `*T` 取决于 `T`）。前者的签名始终是 `DeepCopyObject()` `runtime.Object`：

```
func (in *T) DeepCopyObject() runtime.Object {
    if c := in.DeepCopy(); c != nil {
        return c
    } else {
        return nil
    }
}
```

在你的顶层 API 类型上方放置本地标签 `//` `+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object` 可以生成带有`deepcopy-gen`的这种方法。这告诉`deepcopy-gen`为`runtime.Object`创建这样一个方法，称为`DeepCopyObject()`。

###### 提示

在前面的例子中，`At`和`AtList`都是顶层类型，因为它们作为`runtime.Object`使用。

作为一个经验法则，顶层类型是那些嵌入了`metav1.TypeMeta`的类型。

其他接口需要一种方式进行深拷贝。例如，如果 API 类型有一个接口类型的字段`Foo`，通常情况下就会是这样：

```
type SomeAPIType struct {
  Foo Foo `json:"foo"`
}
```

正如我们所见，API 类型必须能够进行深拷贝，因此字段`Foo`也必须进行深拷贝。在不使用类型转换的情况下，您如何以通用方式实现这一点，而不是将`DeepCopyFoo() Foo`添加到`Foo`接口中呢？

```
type Foo interface {
    ...
    DeepCopyFoo() Foo
}
```

在那种情况下，可以使用相同的标签：

```
// +k8s:deepcopy-gen:interfaces=<package>.Foo
type FooImplementation struct {
    ...
}

```

在 Kubernetes 源码中，实际上有几个超出`runtime.Object`范围的示例使用了此标签：

```
// +k8s:deepcopy-gen:interfaces=.../pkg/registry/rbac/reconciliation.RuleOwner
// +k8s:deepcopy-gen:interfaces=.../pkg/registry/rbac/reconciliation.RoleBinding
```

## client-gen 标签

最后，有许多标签来控制`client-gen`，我们在早期关于`At`和`AtList`的示例中看到了其中之一：

```
// +genclient
```

它告诉`client-gen`为此类型创建一个客户端（这总是可选择的）。请注意，您不必，实际上*不得*将其放在 API 对象的`List`类型之上。

在我们的`cnat`示例中，我们使用*/status*子资源并使用客户端的`UpdateStatus`方法来更新 CR 的状态（参见“状态子资源”）。存在没有状态或没有规范-状态分离的 CR 实例。在这些情况下，以下标签避免了生成`UpdateStatus()`方法：

```
// +genclient:noStatus
```

###### 警告

没有这个标签，`client-gen`将盲目地生成`UpdateStatus()`方法。然而，重要的是要理解，仅当 CustomResourceDefinition 清单中实际启用了*/status*子资源时，规范-状态分离才起作用（参见“子资源”）。

仅仅是客户端中存在该方法本身并没有什么影响。在没有清单中的更改的情况下，对它的请求甚至会失败。

客户端生成器必须选择正确的 HTTP 路径，无论是否有命名空间。对于集群范围的资源，您必须使用此标签：

```
// +genclient:nonNamespaced
```

默认生成的是命名空间客户端。同样，这必须与 CRD 清单中的范围设置相匹配。对于特定用途的客户端，您可能还希望详细控制提供的 HTTP 方法。例如，您可以通过使用一些标签来实现这一点：

```
// +genclient:noVerbs
// +genclient:onlyVerbs=create,delete
// +genclient:skipVerbs=get,list,create,update,patch,delete,watch
// +genclient:method=Create,verb=create,
// result=k8s.io/apimachinery/pkg/apis/meta/v1.Status
```

前三个应该相当容易理解，但最后一个需要一些解释。

此标签上方写的类型将仅用于创建，并不会返回 API 类型本身，而是 `metav1.Status`。对于 CR 来说，这并没有太多意义，但对于用 Go 编写的用户提供的 API 服务器（参见 第八章），这些资源可能存在，并且实际中确实存在。

`// +genclient:method=` 标签的一个常见用例是添加一个用于缩放资源的方法。在 “Scale subresource” 中，我们描述了如何为 CR 启用 */scale* 子资源。以下标签创建相应的客户端方法：

```
// +genclient:method=GetScale,verb=get,subresource=scale,\
//    result=k8s.io/api/autoscaling/v1.Scale
// +genclient:method=UpdateScale,verb=update,subresource=scale,\
//    input=k8s.io/api/autoscaling/v1.Scale,result=k8s.io/api/autoscaling/v1.Scale
```

第一个标签创建了 getter `GetScale`。第二个创建了 setter `UpdateScale`。

###### 注意

所有 CR */scale* 子资源都接收并输出来自 *autoscaling/v1* 组的 `Scale` 类型。在 Kubernetes API 中，出于历史原因，有些资源使用其他类型。

## informer-gen 和 lister-gen

`informer-gen` 和 `lister-gen` 处理 `client-gen` 的 `// +genclient` 标签。没有其他配置可做。每个选择客户端生成的类型都会自动匹配客户端的 Informers 和 Listers（如果通过 *k8s.io/code-generator/generate-group.sh* 脚本调用整个生成器套件）。

Kubernetes 生成器的文档有很大的改进空间，并且肯定会随着时间的推移逐步完善。有关不同生成器的更多信息，通常最好查看 Kubernetes 本身的示例，例如，[k8s.io/api](http://bit.ly/2ZA6dWH) 和 [OpenShift API types](http://bit.ly/2KxpKnc)。这两个存储库都有许多高级用例。

此外，不要犹豫地查看生成器本身。`deepcopy-gen` 在其 [*main.go*](http://bit.ly/2x9HmN4) 文件中有一些可用的文档。`client-gen` 在 [Kubernetes 贡献者文档](http://bit.ly/2WYNlns) 中有一些可用的文档。`informer-gen` 和 `lister-gen` 目前没有进一步的文档，但 *generate-groups.sh* 显示了每个生成器如何被调用的方法。

# 总结

在本章中，我们向您展示了如何为 CR 使用 Kubernetes 代码生成器。现在，我们转向更高级别的抽象工具——即编写自定义控制器和操作员的解决方案，这使您能够专注于业务逻辑。

^(1) Go 工具不会在需要时自动运行生成，也缺乏定义源文件和生成文件之间依赖关系的方法。
