# 第三章：操作员在 Kubernetes 接口中

操作员扩展了 Kubernetes 的两个关键概念：*资源* 和 *控制器*。Kubernetes API 包括一种机制，即 CRD，用于定义新资源。本章探讨了操作员建立在其中的 Kubernetes 对象，以添加到集群中的新功能。它将帮助您了解操作员如何适应 Kubernetes 架构，并解释为何将应用程序变成 Kubernetes 本地应用程序是有价值的。

# 标准缩放：ReplicaSet 资源

查看标准资源 ReplicaSet，可以了解到资源如何构成 Kubernetes 核心的应用管理数据库。与 Kubernetes API 中的任何其他资源一样，[ReplicaSet](https://oreil.ly/nW3ui) 是 API 对象的集合。 ReplicaSet 主要收集形成应用程序运行副本列表的 Pod 对象。另一个对象类型的规范定义了集群上应保持的这些副本数量。第三个对象规范指向创建新 Pod 的模板，当运行的 Pod 较少时。ReplicaSet 中还收集了更多对象，但这三种类型定义了集群上运行的可扩展 Pod 集的基本状态。在此，我们可以看到来自 第一章 的 `staticweb` ReplicaSet 的这三个关键部分（`Selector`、`Replicas` 和 `Pod Template` 字段）：

```
$ kubectl describe replicaset/staticweb-69ccd6d6c
Name:           staticweb-69ccd6d6c
Namespace:      default
Selector:       pod-template-hash=69ccd6d6c,run=staticweb
Labels:         pod-template-hash=69ccd6d6c
                run=staticweb
Controlled By:  Deployment/staticweb
Replicas:       1 current / 1 desired
Pods Status:    1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  pod-template-hash=69ccd6d6c
           run=staticweb
  Containers:
   staticweb:
    Image:        nginx

```

标准 Kubernetes 控制平面组件 ReplicaSet 控制器管理 ReplicaSets 及其属于它们的 Pod。 ReplicaSet 控制器创建 ReplicaSets 并持续监视它们。当运行的 Pod 数量与 `Replicas` 字段中期望的数量不匹配时，ReplicaSet 控制器启动或停止 Pod，以使实际状态与期望状态匹配。

ReplicaSet 控制器采取的操作是有意通用且应用程序无关的。它根据 Pod 模板启动新副本，或删除多余的 Pod。它不应知道每个可能在 Kubernetes 集群上运行的应用程序的启动和关闭顺序的具体细节。

操作员是应用程序特定的 CR 和自定义控制器的组合，它确实了解有关启动、缩放、恢复和管理其应用程序的所有细节。操作员的 *操作数* 是我们称之为应用程序、服务或操作员管理的任何资源。

# 自定义资源

CR 作为 Kubernetes API 的扩展，包含一个或多个字段，就像本地资源一样，但不是默认的 Kubernetes 部署的一部分。CR 包含结构化数据，API 服务器提供了一个机制，通过使用 `kubectl` 或另一个 API 客户端，读取和设置其字段，就像操作本地资源字段一样。用户通过提供 CR 定义在运行的集群上定义 CR。CRD 类似于 CR 的架构，定义了 CR 的字段及其字段包含的值的类型。

## CR 还是 ConfigMap？

Kubernetes 提供了一个标准资源，[*ConfigMap*](https://oreil.ly/ba0uh)，用于使配置数据对应用程序可用。ConfigMaps 似乎与 CRs 的可能用途重叠，但这两个抽象针对不同的情况。

ConfigMaps 最适合为在集群上的 pod 中运行的程序提供配置——想象一下应用程序的配置文件，比如 *httpd.conf* 或 MySQL 的 *mysql.cnf*。应用程序通常希望从其 pod 中读取这种配置，作为文件或环境变量值的形式，而不是从 Kubernetes API 中读取。

Kubernetes 提供了 CRs 来表示 API 中的新对象集合。CRs 由标准 Kubernetes 客户端（如 `kubectl`）创建和访问，并遵循 Kubernetes 的约定，如资源 `.spec` 和 `.status`。在其最有用的情况下，CRs 被自定义控制器代码监视，进而创建、更新或删除其他集群对象，甚至是集群外的任意资源。

# 自定义控制器

CRs 是 Kubernetes API 数据库中的条目。它们可以通过常见的 `kubectl` 命令创建、访问、更新和删除——但单独的 CR 只是一组数据。要为在集群上运行的特定应用程序提供声明性 API，您还需要捕获管理该应用程序流程的活动代码。

我们已经看过一个标准的 Kubernetes 控制器，即 ReplicaSet 控制器。要创建一个操作员，为应用程序的主动管理提供 API，您需要构建控制器模式的实例来控制您的应用程序。此自定义控制器检查并维护应用程序的期望状态，以 CR 中表示。每个操作员都有一个或多个自定义控制器，实现其特定于应用程序的管理逻辑。

# 操作员范围

Kubernetes 集群被划分为*命名空间*。命名空间是集群对象和资源名称的边界。在单个命名空间内名称必须唯一，但在不同命名空间之间可以重复。这使得多个用户或团队可以共享单个集群变得更容易。可以对每个命名空间应用资源限制和访问控制。一个操作员（Operator）可以被限制在一个命名空间中，或者可以在整个集群中维护其操作数。

###### 注意

有关 Kubernetes 命名空间的详细信息，请参阅 [Kubernetes 命名空间文档](https://oreil.ly/k4Okf)。

## 命名空间范围

通常，将您的操作员限制在单个命名空间内是有意义的，并且对于多个团队使用的集群更加灵活。限定在命名空间中的操作员可以独立升级，这允许某些便利设施。例如，您可以在测试命名空间中测试升级，或者从不同的命名空间为兼容性提供旧的 API 或应用程序版本。

## 集群范围的操作员（Cluster-Scoped Operators）

有些情况下，希望运算符能够监视和管理整个集群中的应用程序或服务。例如，管理服务网格（如 [Istio](https://oreil.ly/jM5q2)）或为应用程序端点发放 TLS 证书（如 [`cert-manager`](https://oreil.ly/QT8tE)）的运算符，在监视并操作集群范围的状态时可能效果最佳。

默认情况下，本书中使用的 Operator SDK 创建部署和授权模板，将运算符限制在单个命名空间内。可以将大多数运算符更改为在集群范围内运行。这需要在运算符的清单中进行更改，以指定它应监视集群中的所有命名空间，并且应在 ClusterRole 和 ClusterRoleBinding 的授权对象的权力下运行，而不是命名空间角色和 RoleBinding。

# 授权

在 Kubernetes 中，授权——通过 API 在集群上执行操作的权力——由几种可用的访问控制系统之一定义。基于角色的访问控制（RBAC）是这些中首选且最紧密集成的。RBAC 根据系统用户执行的*角色*来调控对系统资源的访问。角色是一组能够在特定 API 资源上执行某些操作的能力，如*创建*、*读取*、*更新*或*删除*。角色描述的能力通过 RoleBinding 赋予或绑定给用户。

## 服务账户

在 Kubernetes 中，常规的人类用户账户不由集群管理，也没有描述它们的 API 资源。在集群上标识你的用户来自外部提供者，可以是任何东西，从文本文件中的用户列表到通过你的 Google 账户代理认证的 OpenID Connect（OIDC）提供者。

###### 注意

查看[“Kubernetes 中的用户”](https://oreil.ly/WmdTq)文档，了解更多关于 Kubernetes 服务账户的信息。

另一方面，服务账户由 Kubernetes 管理，并且可以通过 Kubernetes API 创建和操作。服务账户是一种特殊类型的集群用户，用于授权程序而不是人员。运算符是使用 Kubernetes API 的程序，大多数运算符应该从服务账户派生其访问权限。在部署运算符时创建服务账户是标准步骤。服务账户标识运算符，账户的角色表示授予运算符的权限。

## 角色

Kubernetes RBAC 默认拒绝权限，因此角色定义授予的权利。一个常见的 Kubernetes 角色的“Hello World”示例如下 YAML 摘录：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]  ![1](img/1.png)
  verbs: ["get", "watch", "list"]  ![2](img/2.png)
```

![1](img/#co_operators_at_the_kubernetes_interface_CO1-1)

此角色授予的权限仅在 Pod 上有效。

![2](img/#co_operators_at_the_kubernetes_interface_CO1-2)

此列表允许在允许资源上执行特定操作。包含对只读访问 Pod 的动词可供绑定了该角色的账户使用。

## RoleBindings

RoleBinding 将角色绑定到一个或多个用户列表。这些用户被授予绑定中引用角色定义的权限。RoleBinding 只能引用其自身命名空间中的角色。当部署限制在命名空间内的操作员时，RoleBinding 将适当的角色绑定到标识操作员的服务账户上。

## ClusterRoles 和 ClusterRoleBindings

正如前文讨论的，大多数操作员限制在一个命名空间内。角色（Roles）和角色绑定（RoleBindings）也受限于命名空间。ClusterRoles 和 ClusterRoleBindings 则是它们的集群范围等价物。标准的命名空间角色绑定只能引用其命名空间内的角色，或整个集群定义的 ClusterRoles。当角色绑定引用 ClusterRole 时，ClusterRole 声明的规则仅适用于绑定所在命名空间中指定的资源。通过这种方式，一组通用角色可以定义为 ClusterRoles，但可以在特定命名空间中重复使用并授予用户。

ClusterRoleBinding 授予用户在整个集群中所有命名空间的能力。负责集群范围职责的操作员通常将 ClusterRole 绑定到操作员服务账户上，通过 ClusterRoleBinding 实现。

# 概要

操作员是 Kubernetes 的扩展。我们已经概述了用于构建知道如何管理其负责应用程序的操作员的 Kubernetes 组件。因为操作员建立在核心 Kubernetes 概念之上，它们可以使应用程序显著地“原生于 Kubernetes”。了解其环境的这类应用程序能够利用平台的设计模式和现有功能，以提高可靠性和减少依赖性。因为操作员礼貌地扩展 Kubernetes，它们甚至可以管理平台本身的部分和流程，正如在 Red Hat 的 OpenShift Kubernetes 发行版中所见。
