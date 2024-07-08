# 第八章：运算符生命周期管理器（Operator Lifecycle Manager，OLM）。

一旦您编写了操作员（Operator），就该将注意力转向其安装和管理。由于部署操作员涉及多个步骤，包括创建部署、添加自定义资源定义以及配置必要的权限，因此需要管理层来促进这一过程。

操作员生命周期管理器（OLM）通过引入交付操作员和在兼容 UI 中可视化所需元数据的打包机制来履行这一角色，其中包括安装说明和 CRD 描述符形式的 API 提示。

OLM 的好处不仅限于安装，还包括 Day 2 运维，例如管理现有操作员的升级、通过版本通道传达操作员稳定性以及将多个操作员托管源聚合到单一接口的能力。

我们通过介绍 OLM 及其接口来开始本章，包括用户将在集群内部与之交互的 CRD 以及它用于操作员的打包格式。接下来，我们将展示 OLM 的操作，使用它连接到 OperatorHub.io 来安装操作员。我们以开发者为重点，探讨编写必要的元数据文件以使操作员可在 OLM 中使用，并在本地集群中进行测试的过程。  

# OLM 自定义资源。

正如您所知，操作员拥有的 CRD 构成了操作员的 API。因此，逐个查看由 OLM 安装的每个 CRD 及其用途是有意义的。

## ClusterServiceVersion（集群服务版本）。

*ClusterServiceVersion*（CSV）是描述操作员的主要元数据资源。每个 CSV 表示操作员的一个版本，并包含以下内容：

+   操作员的通用元数据，包括其名称、版本、描述和图标。

+   描述操作员安装信息，包括创建的部署和所需的权限。

+   操作员拥有的自定义资源定义（CRD）以及操作员依赖的任何 CRD 的引用。

+   CRD 字段上的注解，为用户提供如何正确指定字段值的提示。

当学习 CSV 时，将其概念与传统 Linux 系统进行关联可能很有用。您可以将 CSV 视为类似于 Linux 软件包（如 Red Hat Package Manager（RPM）文件）的东西。与 RPM 文件类似，CSV 包含有关如何安装操作员及其所需依赖项的信息。基于这种类比，您可以将 OLM 视为类似于 yum 或 DNF 的管理工具。

另一个重要的方面是理解 CSV 与其管理的运算符部署资源之间的关系。就像部署描述了它创建的 pod 的“pod 模板”一样，CSV 包含了用于运算符 pod 部署的“部署模板”。这是在 Kubernetes 意义上的正式所有权；如果运算符部署被删除，CSV 将重新创建它以将集群恢复到所需状态，类似于部署将导致删除的 pod 重新创建。

集群服务版本（ClusterServiceVersion）资源通常是从集群服务版本 YAML 文件中填充的。我们将在“编写集群服务版本文件”中提供有关如何编写此文件的更多详细信息。

## CatalogSource

*CatalogSource* 包含访问运算符存储库的信息。OLM 提供了一个名为`packagemanifests`的实用 API，用于查询目录源，该 API 提供了运算符列表以及它们所在的目录。它使用这种资源来填充可用运算符列表。以下是针对默认目录源使用`packagemanifests` API 的示例：

```
$ `kubectl` `-n` `olm` `get` `packagemanifests`
NAME                               CATALOG               AGE
akka-cluster-operator              Community Operators   19s
appsody-operator                   Community Operators   19s
[...]

```

## 订阅

最终用户创建*订阅*来安装，并随后更新，OLM 提供的运算符。订阅是针对*通道*进行的，通道是运算符版本的流，例如“稳定”或“夜间”。

继续之前对 Linux 软件包的类比，订阅等同于安装软件包的命令，例如`yum install`。通过 yum 的安装命令通常会引用软件包的名称而不是特定版本，将最新软件包的确定留给 yum 自己。同样，通过名称和通道订阅运算符让 OLM 根据特定通道中可用的内容解析版本。

用户使用*批准模式*配置订阅。此值设置为`manual`或`automatic`，告诉 OLM 在安装运算符之前是否需要手动用户审查。如果设置为`manual approval`，OLM 兼容的用户界面会向用户呈现 OLM 在运算符安装期间将创建的资源的详细信息。用户可以选择批准或拒绝运算符，OLM 会采取适当的下一步。

## InstallPlan

订阅创建一个*InstallPlan*，描述了 OLM 将创建的用于满足 CSV 资源需求的完整资源列表。对于需要手动批准的订阅，最终用户会在此资源上设置批准，以通知 OLM 安装应该继续。否则，用户无需明确与这些资源交互。

## OperatorGroup

最终用户通过*OperatorGroup*控制运算符的多租户性。这些指定可以由单个运算符访问的命名空间。换句话说，属于 OperatorGroup 的运算符将不会对未在组指示的命名空间中的自定义资源更改作出反应。

尽管您可以使用 OperatorGroups 对一组命名空间进行精细化控制，但它们通常有两种常见用法：

+   将运算符范围限定为单个命名空间

+   允许运算符在所有命名空间全局运行

例如，以下定义创建了一个组，该组将其内的运算符范围限定为单个命名空间`ns-alpha`：

```
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: group-alpha
  namespace: ns-alpha
spec:
  targetNamespaces:
  - ns-alpha
```

完全省略指示符将导致该组覆盖集群中所有命名空间：

```
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: group-alpha
  namespace: ns-alpha ![1](img/1.png)
```

![1](img/#co_operator_lifecycle_manager_CO1-1)

请注意，作为 Kubernetes 资源，OperatorGroup 仍必须驻留在特定命名空间中。但是，缺少`targetNamespaces`指定意味着 OperatorGroup 将覆盖所有命名空间。

###### 注意

此处显示的两个示例涵盖了大多数用例；创建范围超过一个特定命名空间的精细 OperatorGroups 超出了本书的范围。您可以在[OLM 的 GitHub 存储库](https://oreil.ly/ZBAou)中找到更多信息。

# 安装 OLM

在本章的其余部分中，我们将探讨使用和为 OLM 开发。由于大多数 Kubernetes 发行版默认未安装 OLM，第一步是安装运行所需的资源。

###### 警告

OLM 是一个不断发展的项目。因此，请务必查阅其 GitHub 存储库，以找到[当前发布版](https://oreil.ly/It369)的最新安装说明。您可以在 OLM 项目的 GitHub 存储库上找到最新的发布版本。

在当前版本（0.11.0）中，安装执行两项主要任务。

首先，您需要安装 OLM 所需的 CRDs。这些 CRD 作为 API 进入 OLM，并提供配置外部源的能力，这些外部源提供运算符以及用于使这些运算符对用户可用的集群端资源。您可以通过`kubectl apply`命令创建这些 CRD，如下所示：

```
$ `kubectl` `apply` `-f` `\` `https://github.com/operator-framework/operator-lifecycle-manager/releases/``\` `download/0.11.0/crds.yaml`
clusterserviceversions.operators.coreos.com created
installplans.operators.coreos.com created
subscriptions.operators.coreos.com created
catalogsources.operators.coreos.com created
operatorgroups.operators.coreos.com created

```

###### 注意

此处的示例使用了 0.11.0 版本，这是书写时的最新版本；您可以根据阅读时的最新版本更新这些命令。

第二步是创建构成 OLM 本身的所有 Kubernetes 资源。这些资源包括将驱动 OLM 的运算符以及使其正常运行所需的 RBAC 资源（ServiceAccounts、ClusterRoles 等）。

与 CRD 创建一样，您可以通过`kubectl apply`命令执行此步骤：

```
$ `kubectl` `apply` `-f` `\` `https://github.com/operator-framework/operator-lifecycle-manager/``\` `releases/download/0.11.0/olm.yaml`
namespace/olm created
namespace/operators created
system:controller:operator-lifecycle-manager created
serviceaccount/olm-operator-serviceaccount created
clusterrolebinding.rbac.authorization.k8s.io/olm-operator-binding-olm created
deployment.apps/olm-operator created
deployment.apps/catalog-operator created
clusterrole.rbac.authorization.k8s.io/aggregate-olm-edit created
clusterrole.rbac.authorization.k8s.io/aggregate-olm-view created
operatorgroup.operators.coreos.com/global-operators created
operatorgroup.operators.coreos.com/olm-operators created
clusterserviceversion.operators.coreos.com/packageserver created
catalogsource.operators.coreos.com/operatorhubio-catalog created

```

您可以通过查看创建的资源来验证安装是否成功：

```
$ kubectl get ns olm
NAME              STATUS   AGE
olm               Active   43s

$ kubectl get pods -n olm
NAME                                READY   STATUS    RESTARTS   AGE
catalog-operator-7c94984c6c-wpxsv   1/1     Running   0          68s
olm-operator-79fdbcc897-r76ss       1/1     Running   0           68s
olm-operators-qlkh2                 1/1     Running   0           57s
operatorhubio-catalog-9jdd8         1/1     Running   0           57s
packageserver-654686f57d-74skk      1/1     Running   0           39s
packageserver-654686f57d-b8ckz      1/1     Running   0           39s

$ kubectl get crd
NAME                                          CREATED AT
catalogsources.operators.coreos.com           2019-08-07T20:30:42Z
clusterserviceversions.operators.coreos.com   2019-08-07T20:30:42Z
installplans.operators.coreos.com             2019-08-07T20:30:42Z
operatorgroups.operators.coreos.com           2019-08-07T20:30:42Z
subscriptions.operators.coreos.com            2019-08-07T20:30:42Z

```

# 使用 OLM

现在我们已经介绍了围绕 OLM 的基本概念，让我们看看如何使用它来安装运算符。我们将使用 OperatorHub.io 作为运算符的源代码库。我们将在第十章中更详细地介绍 OperatorHub.io，但现在需要知道的重要信息是，它是一个由社区策划的公开可用运算符列表，用于与 OLM 一起使用。与本章前面的 Linux 软件包类比一致，您可以将其视为类似于 RPM 存储库。

安装 OLM 将在 `olm` 命名空间中创建一个默认的目录源。您可以使用 CLI 验证该源是否存在，名称为 `operatorhubio-catalog`：

```
$ `kubectl` `get` `catalogsource` `-n` `olm`
NAME                    NAME                  TYPE   PUBLISHER        AGE
operatorhubio-catalog   Community Operators   grpc   OperatorHub.io   4h20m

```

您可以使用 `describe` 命令找到有关源的更多详细信息：

```
$ `kubectl` `describe` `catalogsource/operatorhubio-catalog` `-n` `olm`
Name:         operatorhubio-catalog
Namespace:    olm
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration...
API Version:  operators.coreos.com/v1alpha1
Kind:         CatalogSource
Metadata:
  Creation Timestamp:  2019-09-23T13:53:39Z
  Generation:          1
  Resource Version:    801
  Self Link:           /apis/operators.coreos.com/v1alpha1/...
  UID:                 45842de1-3b6d-4b1b-bd36-f616dec94c6a
Spec:
  Display Name:  Community Operators  ![1](img/1.png)
  Image:         quay.io/operator-framework/upstream-community-operators:latest
  Publisher:     OperatorHub.io
  Source Type:   grpc
Status:
  Last Sync:  2019-09-23T13:53:54Z
  Registry Service:
    Created At:         2019-09-23T13:53:44Z
    Port:               50051
    Protocol:           grpc
    Service Name:       operatorhubio-catalog
    Service Namespace:  olm
Events:                 <none>

```

![1](img/#comarker1-01)

注意显示名称仅为“Community Operators”，而不指示任何关于 OperatorHub.io 的信息。此值将出现在下一个命令的输出中，当我们查看可能的运算符列表时。

此目录源已配置为读取 OperatorHub.io 托管的所有运算符。您可以使用 `packagemanifest` 实用程序 API 获取发现的运算符列表：

```
$ `kubectl` `get` `packagemanifest` `-n` `olm`
NAME                               CATALOG               AGE
akka-cluster-operator              Community Operators   4h47m
appsody-operator                   Community Operators   4h47m
aqua                               Community Operators   4h47m
atlasmap-operator                  Community Operators   4h47m
[...]  ![1](img/1.png)

```

![1](img/#comarker1-02)

截至撰写时，OperatorHub.io 上有接近 80 个运算符。为简洁起见，我们截取了此命令的输出。

对于此示例，您将安装 etcd 运算符。第一步是定义一个 OperatorGroup 来指定运算符将管理哪些命名空间。您将要使用的 etcd 运算符仅限于单个命名空间（稍后您将看到我们如何确定），因此您将为默认命名空间创建一个组：

```
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: default-og
  namespace: default
spec:
  targetNamespaces:
  - default
```

使用 `kubectl` 的 `apply` 命令创建该组（本示例假定前一片段的 YAML 已保存为名为 *all-og.yaml* 的文件）：

```
$ `kubectl` `apply` `-f` `all-og.yaml`
operatorgroup.operators.coreos.com/default-og created

```

订阅的创建会触发运算符的安装。在进行此操作之前，您需要确定要订阅哪个通道。OLM 除了提供运算符的通道信息外，还提供了丰富的其他详细信息。

您可以使用 `packagemanifest` API 查看此信息：

```
$ `kubectl` `describe` `packagemanifest/etcd` `-n` `olm`
Name:         etcd
Namespace:    olm
Labels:       catalog=operatorhubio-catalog
              catalog-namespace=olm
              provider=CNCF
              provider-url=
Annotations:  <none>
API Version:  packages.operators.coreos.com/v1
Kind:         PackageManifest
Metadata:
  Creation Timestamp:  2019-09-23T13:53:39Z
  Self Link:           /apis/packages.operators.coreos.com/v1/namespaces/...
Spec:
Status:
  Catalog Source:               operatorhubio-catalog
  Catalog Source Display Name:  Community Operators
  Catalog Source Namespace:     olm
  Catalog Source Publisher:     OperatorHub.io
  Channels:
    Current CSV:  etcdoperator.v0.9.4-clusterwide
    Current CSV Desc:
      Annotations:
        Alm - Examples:  [...]   ![1](img/1.png)
[...]  ![2](img/2.png)
    Install Modes:  ![3](img/3.png)
        Type:       OwnNamespace
        Supported:  true ![4](img/4.png)
        Type:       SingleNamespace
        Supported:  true Type:       MultiNamespace
        Supported:  false Type:       AllNamespaces
        Supported:  false ![5](img/5.png)
    Provider:
        Name:       CNCF
    Version:      0.9.4
Name:           singlenamespace-alpha  ![6](img/6.png)
  Default Channel:  singlenamespace-alpha
  Package Name:     etcd
  Provider:
    Name:  CNCF
[...]

```

![1](img/#comarker1-03)

包清单的示例部分包含一系列清单，您可以使用这些清单部署此运算符定义的自定义资源。为简洁起见，我们从此输出中省略了它们。

![2](img/#comarker2)

为了可读性，我们剪裁了文件的大部分内容。我们将在“编写群集服务版本文件”部分讨论这些字段中的许多内容。

![3](img/#comarker3)

安装模式部分描述了最终用户可能部署此运算符的情况。我们稍后将在本章中详细介绍这些内容。

![4](img/#comarker4)

此特定通道提供了一个配置为在部署的同一命名空间中监视的运算符。

![5](img/#comarker5)

类似地，最终用户无法安装此 Operator 来监视集群中的所有命名空间。如果您在包清单数据中查找，将会找到另一个名为 `clusterwide-alpha` 的通道，适用于此目的。

![6](img/#comarker6)

此部分的 `Name` 字段指示订阅引用的通道名称。

由于这个 Operator 来自 OperatorHub.io，直接查看其页面可能会很有帮助。包清单中包含的所有数据都显示在各个 Operator 页面上，但以更易读的方式格式化显示。您可以在 [etcd Operator 页面](https://oreil.ly/1bjkr) 查看这些内容。

一旦确定了通道，最后一步是创建订阅资源本身。以下是一个示例清单：

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: etcd-subscription
  namespace: default ![1](img/1.png)
spec:
  name: etcd ![2](img/2.png)
  source: operatorhubio-catalog ![3](img/3.png)
  sourceNamespace: olm
  channel: singlenamespace-alpha ![4](img/4.png)
```

![1](img/#co_operator_lifecycle_manager_CO2-1)

此清单在默认命名空间中安装了订阅，从而安装了 Operator 部署本身。

![2](img/#co_operator_lifecycle_manager_CO2-2)

通过 `packagemanifest` API 调用找到要安装的 Operator 的名称。

![3](img/#co_operator_lifecycle_manager_CO2-3)

`source` 和 `sourceNamespace` 描述了提供 Operator 的目录源的位置。

![4](img/#co_operator_lifecycle_manager_CO2-4)

OLM 将从 `singlenamespace-alpha` 通道安装 Operators。

与其他资源一样，您可以使用 `kubectl apply` 创建订阅（此命令假定上述订阅 YAML 已保存在名为 *sub.yaml* 的文件中）：

```
$ `kubectl` `apply` `-f` `sub.yaml`
subscription.operators.coreos.com/etcd-subscription created

```

## 探索 Operator

当您创建订阅时，会发生一些事情。在资源层次结构的最高级别上，OLM 在默认命名空间中创建一个 ClusterServiceVersion 资源：

```
$ `kubectl` `get` `csv` `-n` `default`
NAME                  DISPLAY   VERSION   REPLACES              PHASE
etcdoperator.v0.9.4   etcd      0.9.4     etcdoperator.v0.9.2   Succeeded

```

CSV 实际上是订阅安装的内容——就像 RPM 比方中的软件包一样。OLM 执行在 CSV 中定义的 Operator 安装步骤来创建 Operator Pods 本身。此外，OLM 还将存储有关此过程中事件的信息，您可以使用 `describe` 命令查看：

```
$ `kubectl` `describe` `csv/etcdoperator.v0.9.4` `-n` `default`
[...]
Events:
operator-lifecycle-manager  requirements not yet checked
one or more requirements couldn't be found
all requirements found, attempting install
waiting for install components to report healthy
installing: ComponentMissing: missing deployment with name=etcd-operator
installing: ComponentMissing: missing deployment with name=etcd-operator
installing: Waiting: waiting for deployment etcd-operator to become ready:
  Waiting for rollout to finish: 0 of 1 updated replicas are available...
install strategy completed with no errors

```

###### 注意

此处的输出已编辑以适应页面。您的输出将略有不同，并包含更多事件数据。

OLM 负责按照 CSV 中包含的部署模板来创建 Operator Pod 本身。在资源所有权层次结构下进一步查看，您可以看到 OLM 还创建了一个部署资源：

```
$ kubectl get deployment -n default
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
etcd-operator   1/1     1             1           3m42s

```

明确查看部署的详细信息显示了 CSV 与此部署之间的所有权关系：

```
$ `kubectl` `get` `deployment/etcd-operator` `-n` `default` `-o` `yaml`
[...]
ownerReferences:
- apiVersion: operators.coreos.com/v1alpha1
  blockOwnerDeletion: false controller: false kind: ClusterServiceVersion
  name: etcdoperator.v0.9.4
  uid: 564c15d9-ab49-439f-8ea4-8c140f55e641
[...]

```

不出所料，部署根据其资源定义创建了多个 Pod。对于 etcd Operator，CSV 将部署定义为需要三个 Pod：

```
$ kubectl get pods -n default
NAME                            READY   STATUS    RESTARTS   AGE
etcd-operator-c4bc4fb66-zg22g   3/3     Running   0          6m4s

```

总结一下，创建订阅导致以下事件发生：

+   OLM 在与订阅相同的命名空间中创建一个 CSV 资源。此 CSV 包含了部署 Operator 本身的清单，以及其他信息。

+   OLM 使用部署清单为 Operator 创建部署资源。该资源的所有者是 CSV 本身。

+   部署导致为 Operator 本身创建副本集和 pod。

## 删除 Operator

与处理简单部署资源时一样，删除通过 OLM 部署的 Operator 并不那么简单。

部署资源充当 pod 的安装说明。如果通过用户干预或 pod 自身错误删除 pod，则 Kubernetes 检测到部署的期望状态与 pod 的实际数量之间的差异。

类似地，CSV 资源作为 Operator 的安装说明。通常，CSV 指示必须存在部署以执行此计划。如果该部署不存在，OLM 将采取必要步骤，使系统的实际状态与 CSV 的期望状态匹配。

因此，仅仅删除 Operator 的部署资源是不够的。相反，由 OLM 部署的 Operator 将通过删除 CSV 资源来删除：

```
$ kubectl delete csv/etcdoperator.v0.9.4
clusterserviceversion.operators.coreos.com "etcdoperator.v0.9.4" deleted

```

当最初部署时，OLM 负责删除 CSV 创建的资源，包括 Operator 的部署资源。

另外，您需要删除订阅以防止 OLM 在未来安装新的 CSV 版本：

```
$ kubectl delete subscription/etcd-subscription
subscription.operators.coreos.com "etcd-subscription" deleted

```

# OLM Bundle 元数据文件

“OLM 包” 提供了可以安装的 Operator 的详细信息。该包含有关 Operator 所有可用版本的所有必要信息，以便：

+   通过提供用户可以订阅的一个或多个 *通道*，为 Operator 提供灵活的交付结构。

+   部署 Operator 所需的 CRD。

+   指导 OLM 如何创建 Operator 部署。

+   包括每个 CRD spec 字段的额外信息，包括如何在 UI 中呈现这些字段的提示。

OLM 包包括三种类型的文件：自定义资源定义、集群服务版本文件和软件包清单文件。

## 自定义资源定义

由于 Operator 需要其 CRD 来运行，OLM 包将其包括在内。OLM 与 Operator 一起安装 CRD。作为 OLM 包开发者，您无需对 CRD 文件进行任何更改或添加，只需支持 Operator 的已有内容即可。

请记住，只应包括 Operator 拥有的 CRD。任何其他 Operator 提供的依赖 CRD 将由 OLM 的依赖解析自动安装（有关所需 CRD 的概念在“Owned CRDs”中解决）。

###### 注意

每个 CRD 必须在其自己的文件中定义。

## 集群服务版本文件

CSV 文件包含关于 Operator 的大部分元数据，包括：

+   如何部署 Operator

+   Operator 使用的 CRD 列表（包括它拥有的以及其他 Operator 的依赖项）

+   关于 Operator 的元数据，包括描述、标志、成熟级别和相关链接

考虑到此文件的重要角色，我们在下一节中详细介绍如何编写此文件。

## 包清单文件

包清单文件描述了一系列指向特定 Operator 版本的频道。由于频道和它们各自的交付节奏的分解由 Operator 拥有者确定。我们强烈建议频道设置关于稳定性、功能和变更速率的期望。

用户订阅频道。OLM 将使用包清单确定是否在订阅的频道中提供了 Operator 的新版本，并允许用户根据需要采取更新步骤。我们将在“编写包清单文件”中详细介绍此文件。

# 编写集群服务版本文件

每个 Operator 版本都将有自己的集群服务版本文件。CSV 文件是一种标准的 Kubernetes Manifest，类型为 ClusterServiceVersion，这是 OLM 提供的自定义资源之一。

此文件中的资源向 OLM 提供有关特定 Operator 版本的信息，包括安装说明以及用户如何与 Operator 的 CRD 进行交互的额外细节。

## 生成文件骨架

鉴于 CSV 文件中包含的数据量，最简单的起点是使用 Operator SDK 生成一个骨架。SDK 将以 Operator 本身的基本结构为基础构建此骨架，并尽可能填充其余详细信息。这为您提供了一个良好的基础，可以在此基础上完善其余细节。

由于每个 CSV 对应于特定的 Operator 版本，因此版本信息反映在文件名方案中。文件名模式是使用 Operator 名称并附加语义版本号。例如，Visitors Site Operator 的 CSV 文件将命名为类似 *visitors-operator.v1.0.0.yaml* 的东西。

为了使 Operator SDK 能够用特定 Operator 的信息填充骨架 CSV 文件，您必须从 Operator 项目源代码的根目录运行生成命令。这个命令的一般形式如下：

```
$ operator-sdk olm-catalog gen-csv --csv-version *x.y.z*

```

再次强调，由 Operator 开发团队决定他们自己的版本编号策略。为了一致性和一般用户友好性，我们建议 Operator 的发布遵循[语义化版本](https://semver.org)原则。

运行 Visitors Site Operator 的 CSV 生成命令将产生以下输出：

```
$ `operator-sdk` `olm-catalog` `gen-csv` `--csv-version` `1.0.0`
INFO[0000] Generating CSV manifest version 1.0.0
INFO[0000] Fill in the following required fields in file
visitors-operator/1.0.0/visitors-operator.v1.0.0.clusterserviceversion.yaml:
    spec.keywords
    spec.maintainers
    spec.provider
INFO[0000] Created
visitors-operator/1.0.0/visitors-operator.v1.0.0.clusterserviceversion.yaml

```

即使只有基本的 CSV 结构，生成的文件已经相当详细。在高层次上，它包括以下内容：

+   所有 Operator 拥有的 CRD 的引用（换句话说，那些在 Operator 项目中定义的）

+   Operator 部署资源的部分定义

+   Operator 需要的一组 RBAC 规则

+   描述 Operator 将监视的命名空间范围的指示器

+   可以根据需要修改的自定义资源示例（在`metadata.annotations.alm-examples`中找到）

我们将深入探讨每个组件及其应进行的更改类型，在接下来的章节中进行详细讨论。

###### 警告

SDK 将不知道用于 Operator 本身的图像名称。骨架文件包括在部署描述符中的`image: REPLACE_IMAGE`字段。您必须更新此值，以指向 OLM 将部署的 Operator 的托管图像（例如在 Docker Hub 或 Quay.io 上）。

## 元数据

如前所述，`metadata.annotations.alm-examples`字段包含 Operator 拥有的每个 CRD 的示例。SDK 将首先使用在 Operator 项目的*deploy/crds*目录中找到的自定义资源清单来填充此字段。务必查看并填写实际数据，以便最终用户可以根据自己的需求进一步定制。

除了`alm-examples`之外，您可以在清单的`spec`部分找到 Operator 的其余元数据。SDK 生成命令的输出将强调三个特定的必填字段：

关键字

描述 Operator 的类别列表；兼容 UI 用于发现

维护者

代码库维护 Operator 的姓名和电子邮件对列表

提供者

Operator 的发布实体名称

这段来自 etcd Operator 的片段展示了三个必填字段：

```
keywords: ['etcd', 'key value', 'database', 'coreos', 'open source']
maintainers:
- name: etcd Community
  email: etcd-dev@googlegroups.com
provider:
  name: CNCF
```

我们还鼓励您提供以下元数据字段，这些字段可以在 OperatorHub.io 等目录中生成更强大的列表：

显示名称

Operator 的用户友好名称

描述

描述 Operator 功能的字符串；您可以使用 YAML 结构的多行字符串来提供更多的显示信息

版本

Operator 的语义版本，每次发布新的 Operator 镜像时都应递增

替代项

本 CSV 更新的 Operator 版本（如果有）

图标

兼容 UI 使用的 Base64 编码图像

成熟度

包含在此版本中的 Operator 的成熟级别，例如`alpha`、`beta`或`stable`

链接

Operator 的相关链接列表，例如文档、快速入门指南或博客文章

minKubeVersion

Operator 必须部署的 Kubernetes 的最低版本，使用“Major.Minor.Patch”格式（例如，1.13.0）

## 拥有的 CRDs

要安装 Operator，OLM 必须了解其使用的所有 CRD。这些 CRD 分为两种形式：由 Operator 拥有的 CRD 和用作依赖项的 CRD（在 CSV 术语中，这些被称为“必需”CRD；我们将在下一节中介绍这些内容）。

SDK 骨架生成将`spec.customresourcedefinitions`部分添加到 CSV 文件中。它还使用由操作员定义的每个 CRD 填充`owned`部分，包括诸如`kind`、`name`和`version`的标识信息。但是，在 OLM 捆绑包有效之前，您必须手动添加更多字段。

您必须为每个拥有的 CRD 设置以下必需字段：

displayName

自定义资源的用户友好名称

描述

关于自定义资源表示的信息

resources

将由自定义资源创建的 Kubernetes 资源类型列表

`resources` 列表不需要详尽无遗。相反，它应该仅列出用户相关的可见资源。例如，您应该列出用户与之交互的事物，如服务和部署资源，但应省略用户不直接操作的内部 ConfigMap。

每个资源类型只需包含一个实例，无论操作员创建该类型的资源多少个。例如，如果自定义资源创建多个部署，则只需列出部署资源类型一次。

创建一个或多个部署和服务的自定义资源的示例列表如下：

```
resources:
- kind: Service
  version: v1
- kind: Deployment
  version: v1
```

您需要为每个拥有的资源添加两个字段：`specDescriptors` 和 `statusDescriptors`。这些字段提供有关将出现在自定义资源中的 `spec` 和 `status` 字段的附加元数据。兼容的 UI 可以使用此额外信息为用户呈现界面。

对于自定义资源规范中的每个字段，都要向 `specDescriptors` 字段添加一个条目。每个条目应包含以下内容：

displayName

字段的用户友好名称

描述

关于字段表示的信息

路径

对象中字段的点分路径

x-descriptors

关于字段能力的 UI 组件信息

表 8-1 列出了兼容 UI 常见支持的描述符。

表 8-1\. 常用的规范描述符

| 类型 | 描述符字符串 |
| --- | --- |
| 布尔开关 | `urn:alm:descriptor:com.tectonic.ui:booleanSwitch` |
| 复选框 | `urn:alm:descriptor:com.tectonic.ui:checkbox` |
| 端点列表 | `urn:alm:descriptor:com.tectonic.ui:endpointList` |
| 镜像拉取策略 | `urn:alm:descriptor:com.tectonic.ui:imagePullPolicy` |
| 标签 | `urn:alm:descriptor:com.tectonic.ui:label` |
| 命名空间选择器 | `urn:alm:descriptor:com.tectonic.ui:namespaceSelector` |
| 节点亲和性 | `urn:alm:descriptor:com.tectonic.ui:nodeAffinity` |
| 数字 | `urn:alm:descriptor:com.tectonic.ui:number` |
| 密码 | `urn:alm:descriptor:com.tectonic.ui:password` |
| Pod 亲和性 | `urn:alm:descriptor:com.tectonic.ui:podAffinity` |
| Pod 反亲和性 | `urn:alm:descriptor:com.tectonic.ui:podAntiAffinity` |
| 资源需求 | `urn:alm:descriptor:com.tectonic.ui:resourceRequirements` |
| 选择器 | `urn:alm:descriptor:com.tectonic.ui:selector:` |
| 文本 | `urn:alm:descriptor:com.tectonic.ui:text` |
| 更新策略 | `urn:alm:descriptor:com.tectonic.ui:updateStrategy` |

`statusDescriptors` 字段的结构类似，包括需要指定的相同字段。唯一的区别在于有效描述符的集合；这些列在表 8-2 中列出。

表 8-2\. 常用状态描述符

| 类型 | 描述符字符串 |
| --- | --- |
| 条件 | `urn:alm:descriptor:io.kubernetes.conditions` |
| k8s 阶段原因 | `urn:alm:descriptor:io.kubernetes.phase:reason` |
| k8s 阶段 | `urn:alm:descriptor:io.kubernetes.phase` |
| Pod 计数 | `urn:alm:descriptor:com.tectonic.ui:podCount` |
| Pod 状态 | `urn:alm:descriptor:com.tectonic.ui:podStatuses` |
| Prometheus 终端 | `urn:alm:descriptor:prometheusEndpoint` |
| 文本 | `urn:alm:descriptor:text` |
| W3 链接 | `urn:alm:descriptor:org.w3:link` |

例如，以下片段包含了 etcd Operator 的描述符子集：

```
specDescriptors:
- description: The desired number of member Pods for the etcd cluster.
    displayName: Size
    path: size
    x-descriptors:
    - 'urn:alm:descriptor:com.tectonic.ui:podCount'
- description: Limits describes the minimum/maximum amount of compute
               resources required/allowed
    displayName: Resource Requirements
    path: pod.resources
    x-descriptors:
    - 'urn:alm:descriptor:com.tectonic.ui:resourceRequirements'

statusDescriptors:
- description: The status of each of the member Pods for the etcd cluster.
    displayName: Member Status
    path: members
    x-descriptors:
    - 'urn:alm:descriptor:com.tectonic.ui:podStatuses'
- description: The current size of the etcd cluster.
    displayName: Cluster Size
    path: size
- description: The current status of the etcd cluster.
    displayName: Status
    path: phase
    x-descriptors:
    - 'urn:alm:descriptor:io.kubernetes.phase'
- description: Explanation for the current status of the cluster.
    displayName: Status Details
    path: reason
    x-descriptors:
    - 'urn:alm:descriptor:io.kubernetes.phase:reason'
```

## 需要的 CRD

使用但不由 Operator 拥有的自定义资源被指定为*所需*。在安装 Operator 时，OLM 将找到提供所需 CRD 的适当 Operator 并安装它。这允许 Operator 在必要时维持有限的范围，同时利用组合和依赖解析。

CSV 的所需部分是可选的。只有需要使用其他非 Kubernetes 资源的 Operator 需要包括这些。

每个所需 CRD 都使用其：

名称

用于识别所需 CRD 的完整名称

版本

所需的 CRD 版本

种类

Kubernetes 资源类型；在兼容的 UI 中显示给用户。

显示名称

字段的用户友好名称；在兼容的 UI 中显示给用户。

描述

所需 CRD 的使用信息；在兼容的 UI 中显示给用户。

例如，以下指示 EtcdCluster 是不同 Operator 的所需 CRD：

```
required:
- name: etcdclusters.etcd.database.coreos.com
  version: v1beta2
  kind: EtcdCluster
  displayName: etcd Cluster
  description: Represents a cluster of etcd nodes.
```

每个所需 CRD 下 `required` 字段需要一个条目。

## 安装模式

CSV 的安装模式部分告诉 OLM 如何部署 Operator。有四个选项，所有选项都必须在 `installModes` 字段中以自己的标志存在，指示它们是否受支持。当生成 CSV 时，Operator SDK 包含每个选项的默认值集。

支持以下安装模式：

拥有命名空间

Operator 可以部署到选择自己命名空间的 OperatorGroup。

单个命名空间

Operator 可以部署到选择一个命名空间的 OperatorGroup。

多命名空间

Operator 可以部署到选择多个命名空间的 OperatorGroup。

所有命名空间

Operator 可以部署到选择所有命名空间的 OperatorGroup（定义为 `targetNamespace: ""`）。

下面的片段显示了正确构建此字段的方式，以及 SDK 在生成期间设置的默认值：

```
installModes:
- type: OwnNamespace
  supported: true
- type: SingleNamespace
  supported: true
- type: MultiNamespace
  supported: false
- type: AllNamespaces
  supported: true
```

## 版本和更新

忠实于其名称，每个集群服务版本文件代表一个操作员的单个版本。操作员的后续版本将分别具有自己的 CSV 文件。在许多情况下，这可以是前一个版本的副本，并进行适当的更改。

下面描述了操作员版本之间需要进行的一般更改（这不是详尽列表；请务必检查文件的全部内容，以确保不需要进一步的更改）：

+   将新的 CSV 文件名更改为反映操作员的新版本。

+   更新 CSV 文件的 `metadata.name` 字段以及新版本。

+   更新 `spec.version` 字段为新版本。

+   更新 `spec.replaces` 字段以指示新版本升级的先前 CSV 版本。

+   在大多数情况下，新的 CSV 将引用操作员本身的新镜像。请确保根据需要更新 `spec.containers.image` 字段以引用正确的镜像。

+   如果 CRD 发生变化，则可能需要更新 CSV 文件中 CRD 引用的 `specDescriptor` 和 `statusDescriptor` 字段。

尽管这些更改将导致操作员的新版本，但用户在频道中存在该版本之前无法访问该版本。更新 **.package.yaml* 文件以引用适当频道的新 CSV 文件（有关此文件的更多信息，请参见下一节）。

###### 警告

一旦 OLM 发布并使用现有 CSV 文件，请勿修改现有 CSV 文件。请改在新版本的文件中进行更改，并通过频道将其传播给用户。

# 编写包清单文件

与编写集群服务版本文件相比，编写包清单要简单得多。包文件需要三个字段：

包名称

操作员本身的名称；这应该与 CSV 文件中使用的值匹配

频道

用于传送操作员版本的所有频道的列表

默认频道

用户应默认订阅的频道的名称

`channels` 字段中的每个条目由两个项目组成：

名称

频道的名称；这是用户将要订阅的内容

currentCSV

安装通过频道当前已安装的 CSV 文件的完整名称（包括操作员名称但不包括 *.yaml* 后缀）

由操作员团队决定将支持哪些频道的政策留给操作员。

下面的示例通过两个频道分发 Visitors Site 操作员：

```
packageName: visitors-operator
channels:
- name: stable
  currentCSV: visitors-operator.v1.0.0
- name: testing
  currentCSV: visitors-operator.v1.1.0
defaultChannel: stable
```

# 本地运行

一旦编写了必要的捆绑文件，下一步就是构建捆绑并针对本地集群（例如 Minikube 启动的集群）进行测试。在接下来的章节中，我们将描述安装 OLM 到集群、构建 OLM 捆绑和订阅频道以部署操作员的过程。

## 先决条件

本节涵盖了您需要对集群进行的更改，以运行 OLM，并配置它查看您的 bundles 存储库。对于一个集群，您只需要完成这些步骤一次；我们在“构建 OLM Bundle”中讨论了 Operator 的迭代开发和测试。

### 安装 Marketplace Operator

Marketplace Operator 从外部数据存储中导入 Operators。在本章中，您将使用 Quay.io 来托管您的 OLM 包。

###### 注意

尽管其名称如此，Marketplace Operator 并未绑定到特定的 Operators 来源。它只是作为一个导管，从任何兼容的外部存储中获取 Operators。OperatorHub.io 是这样一个站点，我们在第十章中讨论它。

符合 CRDs 代表 Operator API 的概念，安装 Marketplace Operator 会引入两个 CRDs：

+   OperatorSource 资源描述了用于 OLM 包的外部托管注册表。在本例中，我们使用 Quay.io，一个免费的图像托管站点。

+   CatalogSourceConfig 资源连接了 OperatorSource 和 OLM 本身。OperatorSource 会自动创建 CatalogSourceConfig 资源，您无需显式与此类型交互。

###### 警告

与 OLM 类似，Marketplace Operator 是一个不断发展的项目。因此，请务必查阅[其 GitHub 存储库](https://oreil.ly/VNOrU)以获取当前发布的最新安装说明。

由于目前没有 Marketplace Operator 的正式发布，因此通过克隆上游存储库并使用其中的清单来安装它：

```
$ `git` `clone` `https://github.com/operator-framework/operator-marketplace.git`
$ `cd` `operator-marketplace`
$ `kubectl` `apply` `-f` `deploy/upstream/`
namespace/marketplace created
customresourcedefinition.apiextensions.k8s.io/catalogsourceconfigs.....
customresourcedefinition.apiextensions.k8s.io/operatorsources.operators....
serviceaccount/marketplace-operator created
clusterrole.rbac.authorization.k8s.io/marketplace-operator created
role.rbac.authorization.k8s.io/marketplace-operator created
clusterrolebinding.rbac.authorization.k8s.io/marketplace-operator created
rolebinding.rbac.authorization.k8s.io/marketplace-operator created
operatorsource.operators.coreos.com/upstream-community-operators created
deployment.apps/marketplace-operator created

```

您可以通过确保创建了`marketplace`命名空间来验证安装：

```
$ `kubectl` `get` `ns` `marketplace`
NAME          STATUS   AGE
marketplace   Active   4m19s

```

### 安装 Operator Courier

Operator Courier 是一个客户端工具，用于构建并推送 OLM 包到存储库。它还用于验证包文件的内容。

您可以通过 Python 包安装程序`pip`安装 Operator Courier：

```
$ `pip3` `install` `operator-courier`

```

安装完成后，您可以从命令行运行 Operator Courier：

```
$ `operator-courier`
usage: operator-courier <command> [<args>]

These are the commands you can use:
    verify      Create a bundle and test it for correctness.
    push        Create a bundle, test it, and push it to an app registry.
    nest        Take a flat to-be-bundled directory and version nest it.
    flatten     Create a flat directory from versioned operator bundle yaml
                files.

```

### 检索一个 Quay token

Quay.io 是一个免费的容器镜像托管站点。我们将使用 Quay.io 来托管 OLM 包以供 Operator Marketplace 使用。

新用户可以通过[网站](https://quay.io/)注册免费的 Quay.io 帐户。

为了让 Operator Courier 能够将 OLM 包推送到您的 Quay.io 帐户，您需要一个认证令牌。虽然令牌可通过 Web UI 访问，但您也可以使用以下脚本从命令行检索它，按指示替换您的用户名和密码：

```
USERNAME=<quay.io username>
PASSWORD=<quay.io password>
URL=https://quay.io/cnr/api/v1/users/login

TOKEN_JSON=$(curl -s -H "Content-Type: application/json" -XPOST $URL -d \
'{"user":{"username":"'"${USERNAME}"'","password": "'"${PASSWORD}"'"}}')

echo `echo $TOKEN_JSON | awk '{split($0,a,"\""); print a[4]}'`
```

本书的 GitHub 存储库提供了此脚本的交互版本，位于[这里](https://github.com/kubernetes-operators-book/chapters/blob/master/ch08/get-quay-token)。

以后在将 bundle 推送到 Quay.io 时，您将使用此令牌，因此请将其保存在可以访问的地方。脚本的输出提供了一个命令将其保存为环境变量。

### 创建 OperatorSource

OperatorSource 资源定义了用于托管 Operator 包的外部数据存储。在本例中，您将定义一个 OperatorSource 来指向您的 Quay.io 账户，这将提供对其托管的 OLM 包的访问。

以下是一个示例 OperatorSource 清单；您应该将其中的两个`<QUAY_USERNAME>`替换为您的 Quay.io 用户名：

```
apiVersion: operators.coreos.com/v1
kind: OperatorSource
metadata:
  name: <QUAY_USERNAME>-operators  ![1](img/1.png)
  namespace: marketplace
spec:
  type: appregistry
  endpoint: https://quay.io/cnr
  registryNamespace: <QUAY_USERNAME>
```

![1](img/#co_operator_lifecycle_manager_CO3-1)

在这里使用您的用户名并不是硬性要求；这只是确保 OperatorSource 名称唯一的一种简单方式。

编写了 OperatorSource 清单后，使用以下命令创建资源（假设清单文件名为*operator-source.yaml*）：

```
$ `kubectl` `apply` `-f` `operator-source.yaml`

```

要验证 OperatorSource 是否正确部署，请在`marketplace`命名空间中查找所有已知的 OperatorSources 列表：

```
$ `kubectl` `get` `opsrc` `-n` `marketplace`
NAME            TYPE         ENDPOINT             REGISTRY  STATUS
jdob-operators  appregistry  https://quay.io/cnr  jdob      Failed ![1](img/1.png)

```

![1](img/#comarker1-04)

如果在创建源时终点没有包，则状态将为`Failed`。现在您可以忽略这一点；一旦上传了包，稍后您将刷新此列表。

###### 注意

此处显示的输出已经截断以提高可读性；您的结果可能略有不同。

当首次创建 OperatorSource 时，如果在用户的 Quay.io 应用程序列表中找不到 OLM 包，则可能会失败。在稍后的步骤中，您将创建和部署这些包，之后 OperatorSource 将正确启动。我们将此步骤包含为先决条件，因为您只需执行一次；在相同的 Quay.io 命名空间中更新 OLM 包或创建新的 OLM 包时，您将重用 OperatorSource 资源。

此外，OperatorSource 的创建会导致 CatalogSource 的创建。对于这个资源不需要进一步的操作，但您可以通过检查`marketplace`命名空间来确认其存在：

```
$ `kubectl` `get` `catalogsource` `-n` `marketplace`
NAME            NAME     TYPE   PUBLISHER   AGE
jdob-operators           grpc               6m5s
[...]

```

## 构建 OLM 包

安装了初始先决条件后，大部分时间将花在构建和测试循环上。本节涵盖了在 Quay.io 上构建和托管 OLM 包所需的步骤。

### 执行代码检查

使用 Operator Courier 的`verify`命令验证 OLM 包：

```
$ `operator-courier` `verify` `$OLM_FILES_DIRECTORY`

```

### 将包推送到 Quay.io

当元数据文件通过验证并准备好进行测试时，Operator Courier 将 OLM 包上传到您的 Quay.io 账户。在使用`push`命令时有一些必需的参数（和一些可选参数）：

```
$ `operator-courier` `push`
usage: operator-courier [-h] [--validation-output VALIDATION_OUTPUT]
source_dir namespace repository release token

```

这里是 Visitors Site Operator 的推送示例：

```
OPERATOR_DIR=visitors-olm/
QUAY_NAMESPACE=jdob
PACKAGE_NAME=visitors-operator
PACKAGE_VERSION=1.0.0
QUAY_TOKEN=***** ![1](img/1.png)
$ `operator-courier` `push` `"``$OPERATOR_DIR``"` `"``$QUAY_NAMESPACE``"` `\` `"``$PACKAGE_NAME``"` `"``$PACKAGE_VERSION``"` `"``$QUAY_TOKEN``"`

```

![1](img/#comarker1-05)

`QUAY_TOKEN`是完整的令牌，包括“basic”前缀。您可以使用本节早期介绍的脚本来设置此变量。

###### 警告

默认情况下，以这种方式推送到 Quay.io 的包被标记为私有。转到[*https://quay.io/application/*](https://quay.io/application/)的图像，并将其标记为公共，以便集群可以访问。

现在 Operator 捆绑包已准备好测试。对于后续版本，请根据 CSV 文件的新版本更新`PACKAGE_VERSION`变量（有关更多信息，请参阅“版本控制和更新”）并推送新的捆绑包。

### 重新启动 OperatorSource

OperatorSource 在启动时读取配置的 Quay.io 帐户中的 Operator 列表。上传新 Operator 或 CSV 文件的新版本后，您需要重新启动 OperatorSource Pod 以应用更改。

Pod 的名称以 OperatorSource 的相同名称开头。使用前一节中的示例 OperatorSource，将“jdob”作为 Quay.io 用户名，以下演示如何重新启动 OperatorSource：

```
$ kubectl get pods -n marketplace
NAME                                      READY   STATUS    RESTARTS   AGE
jdob-operators-5969c68d68-vfff6           1/1     Running   0          34s
marketplace-operator-bb555bb7f-sxj7d      1/1     Running   0           102m
upstream-community-operators-588bf67cfc   1/1     Running   0           101m

$ kubectl delete pod jdob-operators-5969c68d68-vfff6 -n marketplace
pod "jdob-operators-5969c68d68-vfff6" deleted

$ kubectl get pods -n marketplace
NAME                                      READY   STATUS    RESTARTS   AGE
jdob-operators-5969c68d68-6w8tm           1/1     Running   0           12s  ![1](img/1.png)
marketplace-operator-bb555bb7f-sxj7d      1/1     Running   0           102m
upstream-community-operators-588bf67cfc   1/1     Running   0           102m

```

![1](img/#comarker1-06)

新启动的 Pod 名称后缀与原始 Pod 不同，证实已创建了新的 Pod。

在任何时候，您都可以查询 OperatorSource 以查看其已知 Operator 的列表：

```
$ `OP_SRC_NAME``=``jdob-operators`
$ `kubectl` `get` `opsrc` `$OP_SRC_NAME` `\` `-o``=``custom-columns``=``NAME:.metadata.name,PACKAGES:.status.packages` `\` `-n` `marketplace`
NAME             PACKAGES
jdob-operators   visitors-operator

```

## 通过 OLM 安装 Operator

在配置 Marketplace Operator 以检索您的捆绑包后，通过创建订阅到其支持的频道之一来测试它。OLM 会响应订阅并安装相应的 Operator。

### 创建 OperatorGroup

您将需要一个 OperatorGroup 来指定 Operator 应该监视哪些命名空间。它必须存在于您希望部署 Operator 的命名空间中。为了在测试时简化，此处定义的示例 OperatorGroup 将 Operator 部署到现有的`marketplace`命名空间中：

```
apiVersion: operators.coreos.com/v1alpha2
kind: OperatorGroup
metadata:
  name: book-operatorgroup
  namespace: marketplace
spec:
  targetNamespaces:
  - marketplace
```

与其他 Kubernetes 资源一样，使用`kubectl`的`apply`命令创建 OperatorGroup：

```
$ `kubectl` `apply` `-f` `operator-group.yaml`
operatorgroup.operators.coreos.com/book-operatorgroup created

```

### 创建订阅

通过选择 Operator 和其一个频道，订阅将前面的步骤连接在一起。OLM 使用此信息来启动相应的 Operator Pod。

以下示例为访客站点操作员创建了一个新的稳定频道订阅：

```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: book-sub
  namespace: marketplace  ![1](img/1.png)
spec:
  channel: stable  ![2](img/2.png)
  name: visitors-operator
  source: jdob-operators  ![3](img/3.png)
  sourceNamespace: marketplace  ![4](img/4.png)
```

![1](img/#co_operator_lifecycle_manager_CO4-1)

指示将在其中创建订阅的命名空间。

![2](img/#co_operator_lifecycle_manager_CO4-2)

选择包清单中定义的一个频道。

![3](img/#co_operator_lifecycle_manager_CO4-3)

标识要查看其相应 Operator 和频道的 OperatorSource。

![4](img/#co_operator_lifecycle_manager_CO4-4)

指定 OperatorSource 的命名空间。

使用`apply`命令创建订阅：

```
$ `kubectl` `apply` `-f` `subscription.yaml`
subscription.operators.coreos.com/book-sub created

```

OLM 将收到新订阅的通知，并在`marketplace`命名空间中启动 Operator Pod：

```
$ kubectl get pods -n marketplace
NAME                                           READY   STATUS    RESTARTS   AGE
jdob-operators-5969c68d68-6w8tm                1/1     Running   0           143m
visitors-operator-86cb966f59-l5bkg             1/1     Running   0           12s

```

###### 注意

我们已为了易读性截断此处的输出；您的结果可能略有不同。

## 测试正在运行的 Operator

一旦 OLM 启动了 Operator，您可以通过创建相同类型的自定义资源来测试它。有关测试正在运行的 Operator 的更多信息，请参阅第六章和第七章。

# 访客站点操作员示例

您可以在[书籍的 GitHub 存储库](https://github.com/kubernetes-operators-book/chapters/tree/master/ch08)中找到访客站点运营商的 OLM 捆绑文件。

有两个值得注意的目录：

*bundle*

此目录包含实际的 OLM 捆绑文件，包括 CSV、CRD 和软件包文件。您可以使用本章概述的过程来构建和部署访客站点运营商。

*testing*

此目录包含了从 OLM 部署运营商所需的额外资源。这些资源包括 OperatorSource、OperatorGroup、订阅和一个用于测试运营商的示例自定义资源。

欢迎读者通过 GitHub 的问题选项卡提交有关这些文件的反馈、问题和疑问。

# 摘要

如同任何软件，管理安装和升级对于运营商至关重要。运营商生命周期管理器（Operator Lifecycle Manager，OLM）扮演了这一角色，为您提供了一个机制来发现运营商、处理更新并确保稳定性。

# 资源

+   [OLM 安装](https://oreil.ly/cu1IP)

+   [OLM 存储库](https://oreil.ly/1IN19)

+   [市场运营商存储库](https://oreil.ly/VVvFM)

+   [运营商快递存储库](https://oreil.ly/d6XdP)
