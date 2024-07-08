# 第七章：使用运算符 SDK 中的 Go 运算符

虽然 Helm 和 Ansible 运算符可以快速简单地创建，但它们的功能最终受到这些基础技术的限制。像动态响应应用程序或整个集群中特定变化这样的高级用例需要更灵活的解决方案。

运算符 SDK 提供了这种灵活性，使开发者能够轻松使用 Go 编程语言，包括其外部库生态系统，在他们的运算符中使用。

由于该过程比 Helm 或 Ansible 运算符稍微复杂一些，因此从高层次步骤的摘要开始是有道理的：

1.  创建必要的代码，将其与 Kubernetes 绑定，允许其作为控制器运行运算符。

1.  创建一个或多个 CRD 来建模应用程序的基础业务逻辑，并为用户提供与之交互的 API。

1.  为每个 CRD 创建一个控制器，以处理其资源的生命周期。

1.  构建运算符镜像并创建相关的 Kubernetes 清单以部署运算符及其 RBAC 组件（服务账户、角色等）。

虽然您可以手动编写所有这些部分，但运算符 SDK 提供了命令，可以自动创建大部分支持代码，使您可以专注于实现运算符的实际业务逻辑。

本章使用运算符 SDK 构建项目框架，实现 Go 中的运算符（参见第四章以获取 SDK 安装说明）。我们将探讨需要用自定义应用逻辑编辑的文件，并讨论运算符开发的一些常见实践。一旦运算符准备就绪，我们将在开发模式下运行它以进行测试和调试。

# 初始化运算符

由于运算符是用 Go 编写的，项目框架必须遵循语言约定。特别是，运算符代码必须位于您的 `$GOPATH` 中。有关更多信息，请参阅 [`GOPATH` 文档](https://oreil.ly/2PU_Q)。

SDK 的 `new` 命令创建了运算符所需的基础文件。如果未指定特定的运算符类型，该命令将生成一个基于 Go 的运算符项目：

```
$ OPERATOR_NAME=visitors-operator
$ operator-sdk new $OPERATOR_NAME
INFO[0000] Creating new Go operator 'visitors-operator'.
INFO[0000] Created go.mod
INFO[0000] Created tools.go
INFO[0000] Created cmd/manager/main.go
INFO[0000] Created build/Dockerfile
INFO[0000] Created build/bin/entrypoint
INFO[0000] Created build/bin/user_setup
INFO[0000] Created deploy/service_account.yaml
INFO[0000] Created deploy/role.yaml
INFO[0000] Created deploy/role_binding.yaml
INFO[0000] Created deploy/operator.yaml
INFO[0000] Created pkg/apis/apis.go
INFO[0000] Created pkg/controller/controller.go
INFO[0000] Created version/version.go
INFO[0000] Created .gitignore
INFO[0000] Validating project
[...]  ![1](img/1.png)

```

![1](img/#comarker1-011)

输出已截断以提高可读性。生成过程可能需要几分钟，因为需要下载所有 Go 依赖项。这些依赖项的详细信息将显示在命令输出中。

SDK 创建一个与 `$OPERATOR_NAME` 同名的新目录。生成过程会产生数百个文件，包括生成的文件和供运算符使用的供应商文件。方便的是，大多数文件无需手动编辑。我们将向您展示如何生成满足运算符自定义逻辑所需的文件，详见“自定义资源定义”。

# 运算符范围

您需要首先做出的第一个决定是运算符的范围。有两个选项：

命名空间

限制运算符管理单个命名空间中的资源

Cluster

允许运算符管理整个集群中的资源

默认情况下，SDK 生成的运算符是命名空间范围的。

尽管命名空间范围的运算符通常更可取，但可以将 SDK 生成的运算符更改为集群范围的。执行以下更改以使运算符能够在集群级别运行：

*deploy/operator.yaml*

+   将`WATCH_NAMESPACE`变量的值更改为`""`，表示将监视所有命名空间，而不仅仅是运算符 Pod 所部署的命名空间。

*deploy/role.yaml*

+   将`kind`从`Role`更改为`ClusterRole`，以便在运算符 Pod 的命名空间之外启用权限。

*deploy/role_binding.yaml*

+   将`kind`从`RoleBinding`更改为`ClusterRoleBinding`。

+   在`roleRef`下，将`kind`更改为`ClusterRole`。

+   在`subjects`下，添加键为`namespace`，值为运算符 Pod 所部署的命名空间。

此外，您需要更新生成的 CRD（在下一节中讨论）以指示定义是集群范围的：

+   在 CRD 文件的`spec`部分中，将`scope`字段更改为`Cluster`，而不是默认值`Namespaced`。

+   在 CRD 的*_types.go*文件中，在 CR 的结构体上方添加标签`// +genclient:nonNamespaced`（这将与您用于创建它的`kind`字段具有相同的名称）。这样可以确保将来调用运算符 SDK 刷新 CRD 时不会将值重置为默认值。

例如，对`VisitorsApp`结构体的以下修改表明它是集群范围的：

```
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object 
// VisitorsApp is the Schema for the visitorsapps API // +k8s:openapi-gen=true // +kubebuilder:subresource:status // +genclient:nonNamespaced ![1](img/1.png)
type VisitorsApp struct {
```

![1](img/#co_operators_in_go_with_the_operator_sdk_CO1-1)

标签必须放在资源类型结构体之前。

# 自定义资源定义

在第六章中，我们讨论了创建运算符时 CRD 的角色。您可以使用 SDK 的`add api`命令将新的 CRD 添加到运算符中。该命令从运算符项目的根目录运行，为本书中使用的访客站点示例生成 CRD（使用任意的“example.com”进行演示）：

```
$ `operator-sdk` `add` `api` `--api-version``=``example.com/v1` `--kind``=``VisitorsApp`
INFO[0000] Generating api version example.com/v1 for kind VisitorsApp.
INFO[0000] Created pkg/apis/example/group.go
INFO[0000] Created pkg/apis/example/v1/visitorsapp_types.go
INFO[0000] Created pkg/apis/addtoscheme_example_v1.go
INFO[0000] Created pkg/apis/example/v1/register.go
INFO[0000] Created pkg/apis/example/v1/doc.go
INFO[0000] Created deploy/crds/example_v1_visitorsapp_cr.yaml
INFO[0001] Created deploy/crds/example_v1_visitorsapp_crd.yaml
INFO[0001] Running deepcopy code-generation for Custom Resource group versions:
  [example:[v1], ]
INFO[0001] Code-generation complete.
INFO[0001] Running OpenAPI code-generation for Custom Resource group versions:
  [example:[v1], ]
INFO[0003] Created deploy/crds/example_v1_visitorsapp_crd.yaml
INFO[0003] Code-generation complete.
INFO[0003] API generation complete.

```

该命令生成了多个文件。在以下列表中，请注意`api-version`和 CR 类型名称（`kind`）如何影响生成的名称（文件路径相对于运算符项目的根目录）：

*deploy/crds/example_v1_visitorsapp-cr.yaml*

这是生成类型的示例 CR。它预先填充了适当的`api-version`、`kind`以及资源的名称。您需要填写`spec`部分，使用与您创建的 CRD 相关的值。

*deploy/crds/example_v1_visitorsapp_crd.yaml*

该文件是 CRD 清单的开头。SDK 生成与资源类型名称相关的许多字段（例如复数形式和列表变体），但您需要添加自定义字段，特定于您的资源类型。附录 B 详细介绍了如何完善此文件。

*pkg/apis/example/v1/visitorsapp_types.go*

此文件包含操作员代码库利用的多个结构对象。与许多生成的 Go 文件不同，此文件旨在进行编辑。

`add api` 命令构建适当的框架代码，但在您可以使用资源类型之前，必须定义创建新资源时指定的配置值集合。您还需要添加描述 CR 报告其状态时将使用的字段。您将在定义模板本身以及 Go 对象中添加这些值集合。以下两个部分详细说明每个步骤。

## 定义 Go 类型

在 **_types.go* 文件中（在本例中为 *visitorsapp_types.go*），有两个结构对象需要处理：

+   规格对象（在本示例中为 `VisitorsAppSpec`）必须包括可为此类型资源指定的所有可能配置值。每个配置值由以下内容组成：

    +   变量的名称将在操作员代码中引用（遵循 Go 惯例，为了语言可见性目的以大写字母开头）

    +   用于变量的 Go 类型

    +   变量名称将在 CR 中指定的字段名称（换句话说，用户将编写的 JSON 或 YAML 清单）

+   状态对象（在本示例中为 `VisitorsAppStatus`）必须包括操作员可能设置的所有可能值，以传达 CR 的状态。每个值由以下内容组成：

    +   变量的名称将在操作员代码中引用（遵循 Go 惯例，为了可见性目的以大写字母开头）

    +   用于变量的 Go 类型

    +   作为在获取 `-o yaml` 标志的资源时，将出现在 CR 描述中的字段名称

访问者站点示例支持其访问者应用程序 CR 中的以下数值：

`Size`

要创建的后端副本数量

`Title`

前端网页显示的文本

重要的是要意识到，尽管您在应用程序的不同 Pod 中使用这些值，但将它们包含在单个 CRD 中。从最终用户的角度来看，它们是整个应用程序的属性。操作员的责任是确定如何使用这些值。

每个资源状态中的 VisitorsApp CR 使用以下数值：

`BackendImage`

指示用于部署后端 Pod 的映像和版本

`FrontendImage`

指示用于部署前端 Pod 的映像和版本

*visitorsapp_types.go* 文件的以下片段演示了这些添加：

```
type VisitorsAppSpec struct {
    Size       int32  `json:"size"`
    Title      string `json:"title"`
}

type VisitorsAppStatus struct {
    BackendImage  string `json:"backendImage"`
    FrontendImage string `json:"frontendImage"`
}
```

*visitorsapp_types.go* 文件的其余部分不需要进一步的更改。

在对**_types.go* 文件进行任何更改后，您需要使用 SDK 的`generate`命令（从项目的根目录）更新与这些对象一起工作的任何生成代码：

```
$ `operator-sdk` `generate` `k8s`
INFO[0000] Running deepcopy code-generation for Custom Resource
group versions: [example:[v1], ]
INFO[0000] Code-generation complete.

```

## CRD 清单

类型文件的增加对 Operator 代码非常有用，但不提供给创建资源的最终用户任何洞察力。这些添加是对 CRD 本身进行的。

类似于类型文件，您将在`spec`和`status`部分中对 CRD 进行增加。附录 B 描述了编辑这些部分的过程。

# 运算权限

除了生成 CRD 外，Operator SDK 还创建 Operator 运行所需的 RBAC 资源。默认情况下，生成的角色权限非常宽松，您应在将 Operator 部署到生产环境之前优化其授予的权限。附录 C 涵盖了所有与 RBAC 相关的文件，并讨论了如何将权限范围限定于适用于 Operator 的内容。

# 控制器

CRD 及其相关的 Go 类型文件定义了用户将通过其中进行通信的入站 API。在 Operator Pod 内部，您需要一个控制器来监视 CR 的变化并相应地做出反应。

与添加 CRD 类似，您可以使用 SDK 生成控制器的骨架代码。您将使用先前生成的资源定义的`api-version`和`kind`将控制器范围限定到该类型。以下片段继续介绍访客站点示例：

```
$ `operator-sdk` `add` `controller` `--api-version``=``example.com/v1` `--kind``=``VisitorsApp`
INFO[0000] Generating controller version example.com/v1 for kind VisitorsApp.
INFO[0000] Created pkg/controller/visitorsapp/visitorsapp_controller.go  ![1](img/1.png)
INFO[0000] Created pkg/controller/add_visitorsapp.go
INFO[0000] Controller generation complete.

```

![1](img/#comarker1-022)

注意此文件的名称。它包含实现 Operator 自定义逻辑的 Kubernetes 控制器。

与 CRD 一样，此命令生成多个文件。特别感兴趣的是控制器文件，其根据关联的`kind`进行定位和命名。您无需手动编辑其他生成的文件。

控制器负责“协调”特定资源。单个协调操作的概念与 Kubernetes 遵循的声明模型一致。控制器不会明确处理诸如添加、删除或更新等事件，而是传递资源的当前状态给控制器。由控制器决定要进行的更改集，以更新现实以反映资源描述中所需的状态。有关 Kubernetes 控制器的更多信息，请参阅[官方 Kubernetes 文档](https://oreil.ly/E_hau)。

除了调和逻辑外，控制器还需要建立一个或多个“监视器”。监视器表示当“监视”的资源发生更改时，Kubernetes 应调用此控制器。虽然运算符逻辑的大部分位于控制器的`Reconcile`函数中，但`add`函数建立了将触发调和事件的监视器。SDK 在生成的控制器中添加了两个这样的监视器。

第一个监视器监听控制器监视的主资源的更改。SDK 根据生成控制器时使用的`kind`参数生成此监视器。在大多数情况下，这不需要更改。以下代码片段创建了 VisitorsApp 资源类型的监视器：

```
// Watch for changes to primary resource VisitorsApp
err = c.Watch(&source.Kind{Type: &examplev1.VisitorsApp{}},
              &handler.EnqueueRequestForObject{})
if err != nil {
    return err
}
```

第二个监视器，更确切地说，是一系列监视器，用于监听运算符创建以支持主资源的任何子资源的更改。例如，创建 VisitorsApp 资源会导致创建多个部署和服务对象以支持其功能。控制器为每种这些子类型创建一个监视器，注意仅作用于所有者与主资源相同类型的子资源。例如，以下代码创建了两个监视器，一个用于部署，一个用于服务，其父资源类型为 VisitorsApp：

```
err = c.Watch(&source.Kind{Type: &appsv1.Deployment{}},
              &handler.EnqueueRequestForOwner{
    IsController: true,
    OwnerType:    &examplev1.VisitorsApp{},
})
if err != nil {
    return err
}

err = c.Watch(&source.Kind{Type: &corev1.Service{}},
              &handler.EnqueueRequestForOwner{
    IsController: true,
    OwnerType:    &examplev1.VisitorsApp{},
})
if err != nil {
    return err
}
```

对于此代码片段创建的监视器，有两个感兴趣的区域：

+   构造函数中`Type`的值表示 Kubernetes 监视的子资源类型。每个子资源类型都需要有自己的监视器。

+   每个子资源类型的监视器将`OwnerType`的值设置为主资源类型，作用域限定监视器并导致 Kubernetes 触发父资源的调和。如果没有这个设置，Kubernetes 将对此控制器触发关于*所有*服务和部署更改的调和，而不管它们是否属于运算符。

## 调和函数

`Reconcile`函数，也称为*调和循环*，是运算符逻辑所在的地方。此函数的目的是根据资源请求的期望状态解决系统的实际状态。有关帮助编写此函数的更多信息，请参见下一节。

###### 警告

由于 Kubernetes 在资源的生命周期中多次调用`Reconcile`函数，因此实现必须是幂等的，以防止创建重复的子资源。更多信息请参见“幂等性”。

`Reconcile`函数返回两个对象：一个`ReconcileResult`实例和一个错误（如果有的话）。这些返回值指示 Kubernetes 是否应重新排队请求。换句话说，运算符告诉 Kubernetes 是否应该再次执行调和循环。基于返回值的可能结果是：

`return reconcile.Result{}, nil`

调解过程完成且无错误，并且不需要通过调解循环再次进行。

`return reconcile.Result{}, err`

调解由于错误失败，Kubernetes 应重新排队以再试。

`return reconcile.Result{Requeue: true}, nil`

调解未遇到错误，但 Kubernetes 应重新排队以进行另一个迭代运行。

`return reconcile.Result{RequeueAfter: time.Second*5}, nil`

与前面的结果类似，但这将在重新排队请求之前等待指定的时间。当必须按顺序运行多个步骤但可能需要一些时间才能完成时，这种方法很有用。例如，如果后端服务在启动之前需要运行数据库，则可以延迟重新排队调解以使数据库有时间启动。一旦数据库运行，操作员就不会重新排队调解请求，其余步骤将继续。

# 操作员编写提示

在一本书中不可能涵盖所有操作员的可预见用途和复杂性。单单是应用程序的安装和升级的差异就太多了，这些仅代表操作员成熟模型的前两层。相反，我们将介绍一些通常由操作员执行的基本功能的一般指南，以帮助您入门。

由于基于 Go 的运算符大量使用 Go Kubernetes 库，因此审查[API 文档](https://godoc.org/k8s.io/api)可能很有用。特别是，核心/v1 和 apps/v1 模块经常用于与常见的 Kubernetes 资源交互。

## 检索资源

`Reconcile`函数通常执行的第一步是检索触发调解请求的主资源。Operator SDK 为此生成代码，其示例如下所示，类似于 Visitors Site 示例：

```
// Fetch the VisitorsApp instance instance := &examplev1.VisitorsApp{}
err := r.client.Get(context.TODO(), request.NamespacedName, instance) ![1](img/1.png)![2](img/2.png)

if err != nil {
    if errors.IsNotFound(err) {
        return reconcile.Result{}, nil ![3](img/3.png)
    }
    // Error reading the object - requeue the request.
    return reconcile.Result{}, err
}
```

![1](img/#co_operators_in_go_with_the_operator_sdk_CO2-1)

使用触发调解的资源的值填充先前创建的 VisitorsApp 对象。

![2](img/#co_operators_in_go_with_the_operator_sdk_CO2-2)

变量`r`是调解器对象，`Reconcile`函数在其上调用。它提供了客户端对象，该对象是 Kubernetes API 的经过身份验证的客户端。

![3](img/#co_operators_in_go_with_the_operator_sdk_CO2-3)

当资源被删除时，Kubernetes 仍然调用`Reconcile`函数，在这种情况下，`Get`调用会返回错误。在此示例中，操作员不需要进一步清理已删除的资源，只需返回调解成功即可。我们在“子资源删除”中提供有关处理已删除资源的更多信息。

检索到的实例有两个主要目的：

+   从其`Spec`字段中检索有关资源配置值

+   使用其`Status`字段设置资源的当前状态，并将更新后的信息保存到 Kubernetes 中。

除了`Get`函数外，客户端还提供了一个更新资源值的函数。当更新资源的`Status`字段时，你将使用此函数将更改持久化到 Kubernetes 中。以下代码片段更新了先前检索到的 VisitorsApp 实例状态中的一个字段，并将更改保存回 Kubernetes：

```
instance.Status.BackendImage = "example"
err := r.client.Status().Update(context.TODO(), instance)
```

## 子资源创建

在操作员常见的第一个任务中，是部署必要的资源以使应用程序运行起来。这个操作至关重要，必须是幂等的；对`Reconcile`函数的后续调用应确保资源正在运行，而不是创建重复的资源。

这些子资源通常包括但不限于部署和服务对象。它们的处理方式类似且简单：检查命名空间中资源是否存在，如果不存在，则创建。

以下示例代码片段检查目标命名空间中是否存在部署：

```
found := &appsv1.Deployment{}
findMe := types.NamespacedName{
    Name:      "myDeployment",  ![1](img/1.png)
    Namespace: instance.Namespace,  ![2](img/2.png)
}
err := r.client.Get(context.TODO(), findMe, found)
if err != nil && errors.IsNotFound(err) {
    // Creation logic ![3](img/3.png)
}
```

![1](img/#co_operators_in_go_with_the_operator_sdk_CO3-1)

操作员知道它创建的子资源的名称，或者至少知道如何推断它们（详见“子资源命名”进行更深入的讨论）。在实际用例中，`"myDeployment"`将被替换为操作员在创建部署时使用的相同名称，需要确保相对于命名空间的唯一性。

![2](img/#co_operators_in_go_with_the_operator_sdk_CO3-2)

`instance`变量是关于资源检索的早期片段中设置的，并且指代表示正在协调的主资源的对象。

![3](img/#co_operators_in_go_with_the_operator_sdk_CO3-3)

在这一点上，未找到子资源，并且从 Kubernetes API 中未检索到进一步的错误，因此应执行资源创建逻辑。

操作员通过填充必要的 Kubernetes 对象并使用客户端请求创建资源。请查阅 Kubernetes Go 客户端 API 以获取每种类型资源实例化的规范。你可以在 core/v1 或者 apps/v1 模块中找到许多所需的规范。

例如，以下代码片段创建了访客网站示例应用程序中使用的 MySQL 数据库的部署规范：

```
labels := map[string]string {
    "app":             "visitors",
    "visitorssite_cr": instance.Name,
    "tier":            "mysql",
}
size := int32(1)  ![1](img/1.png)

userSecret := &corev1.EnvVarSource{
    SecretKeyRef: &corev1.SecretKeySelector{
        LocalObjectReference: corev1.LocalObjectReference{Name: mysqlAuthName()},
        Key: "username",
    },
}

passwordSecret := &corev1.EnvVarSource{
    SecretKeyRef: &corev1.SecretKeySelector{
        LocalObjectReference: corev1.LocalObjectReference{Name: mysqlAuthName()},
        Key: "password",
    },
}

dep := &appsv1.Deployment{
    ObjectMeta: metav1.ObjectMeta{
        Name:         "mysql-backend-service", ![2](img/2.png)
        Namespace:    instance.Namespace,
    },
    Spec: appsv1.DeploymentSpec{
        Replicas: &size,
        Selector: &metav1.LabelSelector{
            MatchLabels: labels,
        },
        Template: corev1.PodTemplateSpec{
            ObjectMeta: metav1.ObjectMeta{
                Labels: labels,
            },
            Spec: corev1.PodSpec{
                Containers: []corev1.Container{{
                    Image:  "mysql:5.7",
                    Name:   "visitors-mysql",
                    Ports:  []corev1.ContainerPort{{
                        ContainerPort:    3306,
                        Name:             "mysql",
                    }},
                    Env: []corev1.EnvVar{ ![3](img/3.png)
                        {
                            Name: "MYSQL_ROOT_PASSWORD",
                            Value: "password",
                        },
                        {
                            Name: "MYSQL_DATABASE",
                            Value: "visitors",
                        },
                        {
                            Name: "MYSQL_USER",
                            ValueFrom: userSecret,
                        },
                        {
                            Name: "MYSQL_PASSWORD",
                            ValueFrom: passwordSecret,
                        },
                    },
                }},
            },
        },
    },
}

controllerutil.SetControllerReference(instance, dep, r.scheme) ![4](img/4.png)
```

![1](img/#co_operators_in_go_with_the_operator_sdk_CO4-1)

在许多情况下，操作员将从主资源的规范中读取部署的 Pod 数目。为简单起见，在此示例中，将其硬编码为`1`。

![2](img/#co_operators_in_go_with_the_operator_sdk_CO4-2)

当你尝试查看部署是否存在时，这就是在前面片段中使用的值。

![3](img/#co_operators_in_go_with_the_operator_sdk_CO4-3)

对于此示例，这些是硬编码的值。请注意根据需要生成随机化值。

![4](img/#co_operators_in_go_with_the_operator_sdk_CO4-4)

这可以说是定义中最重要的一行。它建立了主资源（VisitorsApp）与子资源（deployment）之间的父/子关系。Kubernetes 使用此关系执行某些操作，如下一节所示。

deployment 的 Go 表示结构与 YAML 定义紧密相关。再次查阅 API 文档，了解如何使用 Go 对象模型的具体细节。

无论子资源类型是部署、服务等，都要使用客户端创建它：

```
createMe := // Deployment instance from above

// Create the service
err = r.client.Create(context.TODO(), createMe)

if err != nil {
    // Creation failed
    return &reconcile.Result{}, err
} else {
    // Creation was successful
    return nil, nil
}
```

## 子资源删除

在大多数情况下，删除子资源比创建它们简单得多：Kubernetes 会为您完成。如果子资源的所有者类型正确设置为主资源，则在删除父资源时，Kubernetes 垃圾回收将自动清理其所有子资源。

重要的是要理解，当 Kubernetes 删除资源时，仍会调用`Reconcile`函数。仍然执行 Kubernetes 垃圾回收，并且运算符将无法检索到主资源。请参见“检索资源”中检查此情况的代码示例。

但是，在某些情况下，需要特定的清理逻辑。在这种情况下的方法是通过使用*finalizer*阻止主资源的删除。

finalizer 只是资源上的一系列字符串。如果一个或多个 finalizer 存在于资源上，则对象的`metadata.deletionTimestamp`字段将被填充，表示最终用户希望删除该资源。然而，只有当所有 finalizer 都被移除后，Kubernetes 才会执行实际的删除。

使用此结构，您可以阻止资源的垃圾回收，直到运算符有机会执行自己的清理步骤。一旦运算符完成了必要的清理工作，它将删除 finalizer，解除 Kubernetes 执行其正常删除步骤的阻塞。

以下片段演示了使用 finalizer 提供一个窗口，其中运算符可以执行预删除步骤。此代码在检索实例对象之后执行，如“检索资源”中所述：

```
finalizer := "visitors.example.com"

beingDeleted := instance.GetDeletionTimestamp() != nil  ![1](img/1.png)
if beingDeleted {
    if contains(instance.GetFinalizers(), finalizer) {

        // Perform finalization logic. If this fails, leave the finalizer
        // intact and requeue the reconcile request to attempt the clean
        // up again without allowing Kubernetes to actually delete
        // the resource. 
        instance.SetFinalizers(remove(instance.GetFinalizers(), finalizer)) ![2](img/2.png)
        err := r.client.Update(context.TODO(), instance)
        if err != nil {
            return reconcile.Result{}, err
        }
    }
    return reconcile.Result{}, nil
}
```

![1](img/#co_operators_in_go_with_the_operator_sdk_CO5-1)

删除时间戳的存在表明，一个或多个 finalizer 正在阻止所请求的删除。

![2](img/#co_operators_in_go_with_the_operator_sdk_CO5-2)

一旦清理任务完成，运算符会删除 finalizer，因此 Kubernetes 可以继续进行资源清理。

## 子资源命名

虽然最终用户在创建 CR 时提供 CR 的名称，但操作员负责生成它创建的任何子资源的名称。在创建这些名称时，请考虑以下原则：

+   资源名称在给定命名空间内必须是唯一的。

+   子资源名称应动态生成。如果在同一命名空间中存在多个相同类型的资源，则硬编码子资源名称会导致冲突。

+   子资源名称必须可复制和一致。在未来迭代中，操作员可能需要通过协调循环访问资源的子项，并且必须能够可靠地通过名称检索这些资源。

## 幂等性

许多开发人员在编写控制器时面临的最大障碍之一是 Kubernetes 使用*声明式*API 的概念。最终用户不会立即发出 Kubernetes 立即执行的命令。相反，他们请求集群应达到的最终状态。

因此，控制器（及其扩展的操作员）的接口不包括“添加资源”或“更改配置值”等命令。相反，Kubernetes 只是要求控制器协调资源的状态。然后，操作员确定要采取哪些步骤（如果有的话）以确保达到最终状态。

因此，操作员的*幂等性*至关重要。对于未更改的资源的多次协调请求，必须每次产生相同的效果。

以下提示可以帮助您确保操作员中的幂等性：

+   在创建子资源之前，请检查它们是否已存在。请记住，Kubernetes 可能出于各种原因调用协调循环，而不仅仅是在用户首次创建 CR 时。因此，您的控制器不应在每次循环迭代中复制 CR 的子资源。

+   对资源规范（即其配置值）的更改会触发协调循环。因此，仅仅检查预期子资源的存在通常是不够的。操作员还需要验证，在协调时，子资源配置是否与父资源中定义的相匹配。

+   并非每次对资源的更改都会调用协调。可能一次协调中包含多个更改。操作员必须小心确保所有子资源都代表了 CR 的整个状态。

+   即使操作员确定不需要对现有资源进行任何更改，也不意味着它不需要更新 CR 的`Status`字段。根据 CR 状态中捕获的值，可能有必要更新这些值，即使操作员确定不需要对现有资源进行任何更改。

## 操作员的影响

重要的是要意识到您的 Operator 将对集群产生的影响。在大多数情况下，您的 Operator 将创建一个或多个资源。它还需要通过 Kubernetes API 与集群进行通信。如果 Operator 错误处理这些操作，可能会对整个集群的性能产生负面影响。

如何处理这一最佳实践因 Operator 而异。没有一套规则可以确保 Operator 不会过度负担集群。然而，您可以使用以下指南作为分析 Operator 方法的起点：

+   在频繁调用 Kubernetes API 时要小心。确保在重复检查 API 以满足特定状态时使用合理的延迟（以秒为单位，而不是毫秒）。

+   可能时，尽量不要阻塞调解方法长时间。例如，如果在继续之前等待子资源可用，请考虑在延迟后触发另一个调解（有关触发调解循环中后续迭代的更多信息，请参见 “调解函数”）。这种方法允许 Kubernetes 管理其资源，而不是让调解请求长时间等待。

+   如果要部署大量资源，请考虑通过调解循环的多次迭代来限制部署请求。请记住集群上还同时运行其他工作负载。您的 Operator 不应通过同时发出许多创建请求来对集群资源造成过多压力。

# 本地运行 Operator

Operator SDK 提供了在运行中的集群之外运行 Operator 的方法。这有助于通过省去镜像创建和托管步骤来加快开发和测试速度。运行 Operator 的进程可能位于集群之外，但 Kubernetes 将其视为任何其他控制器。

测试 Operator 的高级步骤如下：

1.  *部署 CRD.* 您只需执行一次此操作，除非需要对 CRD 进行进一步更改。在这些情况下，从 Operator 项目根目录再次运行 `kubectl apply` 命令以应用任何更改：

    ```
    $ `kubectl` `apply` `-f` `deploy/crds/*_crd.yaml`

    ```

1.  *以本地模式启动 Operator.* Operator SDK 使用来自 `kubectl` 配置文件的凭据连接到集群并附加 Operator。运行的进程将作为在集群内运行的 Operator pod，并将日志信息写入标准输出：

    ```
    $ `export` `OPERATOR_NAME``=``<``operator-name>`
    $ `operator-sdk` `up` `local` `--namespace` `default`

    ```

    `--namespace` 标志指示 Operator 将显示为运行的命名空间。

1.  *部署示例资源.* SDK 生成了一个示例 CR 和 CRD。它们位于同一目录中，文件名以 *_cr.yaml* 结尾来表示其功能。

    在大多数情况下，您将希望编辑此文件的 `spec` 部分，以提供资源的相关配置值。完成必要的更改后，使用 `kubectl` 部署 CR（从项目根目录）：

    ```
    $ `kubectl` `apply` `-f` `deploy/crds/*_cr.yaml`

    ```

1.  *停止正在运行的运算符进程。* 通过按下 `Ctrl+C` 停止运算符进程。除非运算符将 finalizers 添加到 CR 中，在删除 CR 本身之前，这样做是安全的，因为 Kubernetes 将使用其资源的父/子关系清理任何依赖对象。

###### 注意

此处描述的流程对于开发目的很有用，但对于生产环境，运算符作为镜像交付。有关如何在集群内作为容器构建和部署运算符的更多信息，请参见 附录 A。

# Visitors Site 示例

Visitors Site 运算符的代码库过大而无法包含。您可以在 [本书的 GitHub 存储库](https://github.com/kubernetes-operators-book/chapters/tree/master/ch07/visitors-operator) 中找到完整构建的运算符。

Operator SDK 生成了该存储库中的许多文件。修改以运行 Visitors Site 应用程序的文件包括：

*deploy/crds/*

+   *example_v1_visitorsapp_crd.yaml*

    +   此文件包含了 CRD。

+   *example_v1_visitorsapp_cr.yaml*

    +   这个文件定义了一个带有合理示例数据的 CR。

*pkg/apis/example/v1/visitorsapp_types.go*

+   该文件包含表示 CR 的 Go 对象，包括其 `spec` 和 `status` 字段。

*pkg/controller/visitorsapp/*

+   *backend.go*, *frontend.go*, *mysql.go*

    +   这些文件包含部署 Visitors Site 组件的所有特定信息。这包括运算符维护的部署和服务，以及处理最终用户更改 CR 时更新现有资源的逻辑。

+   *common.go*

    +   此文件包含用于确保部署和服务运行的实用方法，必要时创建它们。

+   *visitorsapp_controller.go*

    +   Operator SDK 最初生成了此文件，然后为 Visitors Site 特定逻辑进行了修改。`Reconcile` 方法包含了大部分更改；通过调用先前描述的文件中的函数，它驱动运算符的整体流程。

# 概要

编写一个运算符需要大量的代码来将其作为控制器与 Kubernetes 集成。Operator SDK 通过生成大部分样板代码来简化开发，让你专注于业务逻辑方面。SDK 还提供了构建和测试运算符的实用工具，大大减少了从构思到运行运算符所需的工作量。

# 资源

+   [Kubernetes CR 文档](https://oreil.ly/IwYGV)

+   [Kubernetes API 文档](https://godoc.org/k8s.io/api)
