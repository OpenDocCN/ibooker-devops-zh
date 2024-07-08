# 第十四章：Kubernetes 的基于角色的访问控制

此时，您几乎会遇到的每个 Kubernetes 集群都已启用基于角色的访问控制（RBAC）。所以您很可能之前就遇到过 RBAC。也许最初您无法访问您的集群，直到使用某些神奇的命令添加了一个 RoleBinding 来映射一个用户到一个角色。即使您可能已经接触过 RBAC，您可能还没有很好地理解 Kubernetes 中的 RBAC，包括其用途和如何使用它。

基于角色的访问控制提供了一种机制，用于限制对 Kubernetes API 的访问和操作，以确保只有授权的用户能够访问。RBAC 是加固您部署应用程序的 Kubernetes 集群访问权限的关键组成部分，（可能更重要的是）防止一个人在错误的命名空间误以为正在销毁他们的测试集群时，意外地关闭生产环境。

###### 注意

虽然 RBAC 在限制对 Kubernetes API 的访问方面非常有用，但重要的是要记住，任何能在 Kubernetes 集群内运行任意代码的人都可以有效地获得整个集群的 root 权限。有一些方法可以使此类攻击变得更加困难和昂贵，正确的 RBAC 设置就是这种防御的一部分。但如果您专注于敌对的多租户安全性，仅仅 RBAC 本身是足以保护您的。您必须隔离运行在集群中的 Pods，以提供有效的多租户安全性。通常，这是通过使用隔离容器或容器沙箱来完成的。

在我们深入了解 Kubernetes 中的 RBAC 的详细信息之前，了解 RBAC 作为一个概念以及身份验证和授权的高层理解是非常有价值的。

每个对 Kubernetes 的请求首先需要进行*认证*。认证提供了发出请求的调用者的身份。可以简单地说该请求未经身份验证，或者它可以深度集成到可插拔的身份验证提供程序（例如 Azure Active Directory）中，在该第三方系统中建立一个身份。有趣的是，Kubernetes 并没有内置的身份存储，而是专注于将其他身份源集成到自身中。

一旦用户经过身份验证，授权阶段确定他们是否被授权执行该请求。授权是用户的身份、资源（实际上是 HTTP 路径）以及用户试图执行的动作的组合。如果特定用户被授权在该资源上执行该操作，则允许该请求继续进行。否则，将返回 HTTP 403 错误。让我们深入了解这个过程。

# 基于角色的访问控制

要在 Kubernetes 中正确管理访问权限，理解身份、角色和角色绑定如何交互以控制谁可以使用哪些资源是至关重要的。刚开始时，RBAC 可能看起来像是一个难以理解的挑战，由一系列相互连接的抽象概念组成；但一旦理解了，您就可以确信自己能够有效地管理集群访问权限。

## Kubernetes 中的身份

每个对 Kubernetes 的请求都与某个身份相关联。即使是没有身份的请求也与`system:unauthenticated`组相关联。Kubernetes 区分用户身份和服务账户身份。服务账户由 Kubernetes 自身创建和管理，通常与集群内运行的组件相关联。用户账户则是与集群的实际用户相关联的所有其他账户，通常包括在集群外运行的持续交付服务等自动化服务。

Kubernetes 为身份验证提供者使用通用接口。每个提供者提供一个用户名，以及可选的用户所属组集合。Kubernetes 支持多种身份验证提供者，包括：

+   HTTP 基本身份验证（已大部分废弃）

+   x509 客户端证书

+   主机上的静态令牌文件

+   云身份验证提供者，例如 Azure Active Directory 和 AWS 身份和访问管理（IAM）

+   身份验证 Webhook

尽管大多数托管的 Kubernetes 安装为您配置了身份验证，但如果您正在部署自己的身份验证，则需要适当地配置 Kubernetes API 服务器上的标志。

在您的集群中，应始终为不同的应用程序使用不同的身份。例如，您应该为生产前端使用一个身份，为生产后端使用另一个身份，并且所有生产身份应该与开发身份不同。您还应为不同的集群使用不同的身份。所有这些身份应该是不与用户共享的机器身份。您可以使用 Kubernetes 服务账户来实现这一点，或者您可以使用由您的身份系统提供的 Pod 身份提供者；例如，Azure Active Directory 提供了一个[Pod 的开源身份提供者](https://oreil.ly/YLymu)，其他流行的身份提供者也提供类似的解决方案。

## 理解 Kubernetes 中的角色和角色绑定

身份仅是 Kubernetes 授权的起点。一旦 Kubernetes 知道请求的身份，它就需要确定是否授权该用户执行该请求。为了实现这一点，它使用角色和角色绑定。

*角色*是一组抽象的能力。例如，`appdev`角色可能代表创建 Pod 和服务的能力。*角色绑定*是将角色分配给一个或多个身份的过程。因此，将`appdev`角色绑定到用户身份`alice`表示 Alice 具有创建 Pod 和服务的能力。

## Kubernetes 中的角色和角色绑定

在 Kubernetes 中，有两对相关资源表示角色和角色绑定。一对作用域是命名空间级别（Role 和 RoleBinding），而另一对作用域是集群级别（ClusterRole 和 ClusterRoleBinding）。

让我们首先检查 Role 和 RoleBinding。Role 资源是命名空间化的，表示该单个命名空间中的功能。你不能将命名空间角色用于非命名空间资源（例如 CustomResourceDefinitions），并且将 RoleBinding 绑定到角色只在包含 Role 和 RoleBinding 的 Kubernetes 命名空间中提供授权。

作为一个具体的例子，这里有一个简单的角色，赋予一个身份创建和修改 Pods 和 Services 的能力：

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-and-services
rules:
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
```

要将此角色绑定到用户 `alice`，我们需要创建一个如下所示的 RoleBinding。这个角色绑定还将组 `mydevs` 绑定到相同的角色：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: default
  name: pods-and-services
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: alice
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: mydevs
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-and-services
```

有时候你需要创建一个适用于整个集群的角色，或者你想要限制对集群级资源的访问。为了实现这一点，你可以使用 ClusterRole 和 ClusterRoleBinding 资源。它们在很大程度上与它们的命名空间对等物相同，但是作用域是集群级的。

### Kubernetes 角色的动词

角色定义了对资源（例如 Pods）以及描述可以在该资源上执行的动作的动词。这些动词大致对应于 HTTP 方法。Kubernetes RBAC 中常用的动词列在 Table 14-1 中。

Table 14-1\. Kubernetes RBAC 常见动词

| Verb | HTTP 方法 | 描述 |
| --- | --- | --- |
| `create` | `POST` | 创建新资源。 |
| `delete` | `DELETE` | 删除现有资源。 |
| `get` | `GET` | 获取资源。 |
| `list` | `GET` | 列出资源集合。 |
| `patch` | `PATCH` | 通过部分更改修改现有资源。 |
| `update` | `PUT` | 通过完整对象修改现有资源。 |
| `watch` | `GET` | 监视资源的流式更新。 |
| `proxy` | `GET` | 通过流式 WebSocket 代理连接资源。 |

### 使用内建角色

设计自己的角色可能会很复杂且耗时。Kubernetes 有大量用于已知系统身份（例如调度程序）的内建集群角色，需要已知的一组功能。你可以通过运行以下命令查看这些角色：

```
$ kubectl get clusterroles
```

虽然大多数内建角色是为系统实用程序设计的，但其中四个是为通用最终用户设计的：

+   `cluster-admin` 角色提供对整个集群的完全访问权限。

+   `admin` 角色提供对完整命名空间的完全访问权限。

+   `edit` 角色允许最终用户在命名空间中修改资源。

+   `view` 角色允许对命名空间进行只读访问。

大多数集群已经设置了大量的 ClusterRole 绑定，你可以使用 `kubectl get clusterrolebindings` 查看这些绑定。

### 内建角色的自动协调

当 Kubernetes API 服务器启动时，它会自动安装一些默认的 ClusterRoles，这些 ClusterRoles 是在 API 服务器代码中定义的。这意味着，如果您修改任何内置的集群角色，这些修改是暂时的。每当 API 服务器重新启动（例如进行升级）时，您的更改将被覆盖。

为了防止这种情况发生，在进行其他任何修改之前，您需要向内置的 ClusterRole 资源添加 `rbac.authorization.kubernetes.io/autoupdate` 注解，并将其值设置为 `false`。如果将此注解设置为 `false`，则 API 服务器将不会覆盖已修改的 ClusterRole 资源。

###### 警告

默认情况下，Kubernetes API 服务器安装一个集群角色，允许 `system:unauthenticated` 用户访问 API 服务器的 API 发现端点。对于任何暴露于敌对环境（例如公共互联网）的集群来说，这是一个不好的想法，并且已经至少发生过一个严重的安全漏洞通过这种暴露。如果您在公共互联网或其他敌对环境上运行 Kubernetes 服务，您应确保在您的 API 服务器上设置 `--anonymous-auth=false` 标志。

# 管理 RBAC 的技术

管理集群的 RBAC 可能会很复杂和令人沮丧。可能更令人担忧的是，错误配置的 RBAC 可能会导致安全问题。幸运的是，有几种工具和技术可以使 RBAC 管理变得更加容易。

## 使用 can-i 进行授权测试

第一个有用的工具是 `kubectl` 的 `auth can-i` 命令。此工具用于测试特定用户是否可以执行特定操作。您可以在配置集群时使用 `can-i` 验证配置设置，或者要求用户在提交错误或 bug 报告时使用该工具验证其访问权限。

在其最简单的用法中，`can-i` 命令接受一个动词和一个资源。例如，此命令将指示当前 `kubectl` 用户是否被授权创建 Pods：

```
$ kubectl auth can-i create pods
```

您还可以使用 `--subresource` 命令行标志测试子资源，如日志或端口转发：

```
$ kubectl auth can-i get pods --subresource=logs
```

## 在源代码控制中管理 RBAC

与 Kubernetes 中的所有资源一样，RBAC 资源使用 YAML 建模。鉴于这种基于文本的表示形式，将这些资源存储在版本控制中是有意义的，这允许进行问责、审计和回滚。

`kubectl` 命令行工具提供了一个 `reconcile` 命令，类似于 `kubectl apply`，它将与集群的当前状态协调一组角色和角色绑定。您可以运行：

```
$ kubectl auth reconcile -f some-rbac-config.yaml
```

如果您希望在应用更改之前查看它们，可以在命令中添加 `--dry-run` 标志以输出但不应用更改。

# 高级主题

一旦您掌握了基于角色的访问控制的基础知识，管理 Kubernetes 集群的访问就相对容易了。但是，当管理大量用户或角色时，您可以使用其他高级功能来扩展 RBAC 的管理能力。

## 聚合 ClusterRoles

有时您希望能够定义其他角色的组合角色。一种选择是简单地将一个 ClusterRole 的所有规则克隆到另一个 ClusterRole 中，但这很复杂且容易出错，因为对一个 ClusterRole 的更改不会自动反映在另一个 ClusterRole 中。相反，Kubernetes RBAC 支持使用*聚合规则*来将多个角色组合成一个新角色。此新角色结合了所有聚合角色的所有功能，并且任何对任何组成子角色的更改都将自动传播回聚合角色中。

与 Kubernetes 中的其他聚合或分组类似，要聚合的 ClusterRoles 是使用标签选择器指定的。在这种特定情况下，ClusterRole 资源中的`aggregationRule`字段包含一个`clusterRoleSelector`字段，后者是一个标签选择器。所有与此选择器匹配的 ClusterRole 资源都动态聚合到聚合 ClusterRole 资源的`rules`数组中。

管理 ClusterRole 资源的最佳实践是创建多个细粒度的集群角色，然后将它们聚合以形成更高级别或更广泛的集群角色。这就是内置集群角色的定义方式。例如，您可以看到内置的`edit`角色如下所示：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: edit
  ...
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-edit: "true"
...
```

这意味着`edit`角色被定义为所有具有标签`rbac.authorization.k8s.io/aggregate-to-edit`设置为`true`的 ClusterRole 对象的集合。

## 使用组进行绑定

在管理不同组织中大量人员且具有相似集群访问权限时，通常最佳做法是使用组来管理定义访问权限的角色，而不是单独向特定身份添加绑定。当将组绑定到角色或 ClusterRole 时，属于该组的任何成员都可以访问该角色定义的资源和动作。因此，要使任何个人获得访问组角色的权限，需要将该个人添加到组中。

使用组是管理规模化访问的首选策略，原因有几点。首先，在任何大型组织中，对集群的访问是以某人所在团队为单位定义的，而不是他们的具体身份。例如，属于前端运营团队的人需要访问与前端相关的资源并具有编辑权限，而他们可能只需要对与后端相关的资源具有查看/读取权限。授予组权限使特定团队与其能力之间的关联清晰明了。当向个人授予角色时，很难清楚地了解每个团队所需的适当（即最小）权限，特别是当一个人可能属于多个团队时。

将角色绑定到组而不是个人的额外好处是简单性和一致性。当有人加入或离开团队时，只需简单地将其添加到或从组中移除，而不是必须删除多个不同的角色绑定。如果您不得不为其身份删除多个角色绑定，可能会导致权限过多或过少，从而造成不必要的访问或阻止其执行必要操作。此外，由于只需维护单一组角色绑定集，因此无需大量工作来确保所有团队成员拥有相同且一致的权限集。

###### 注意

许多云服务提供商支持将其身份和访问管理平台集成到 Kubernetes RBAC 中，以便可以与 Kubernetes RBAC 一起使用这些平台的用户和组。

许多组系统支持“即时”访问（JIT），即人们仅在响应事件时（例如半夜的警报页面）临时添加到组中，而不是保持持久访问权限。这意味着您可以审计任何特定时间谁有访问权限，并确保即使是受损身份也不能访问您的生产基础设施。

最后，在许多情况下，这些同样的组用于管理对其他资源（从设施到文档和机器登录）的访问权限。因此，将这些相同的组用于 Kubernetes 访问控制大大简化了管理工作。

要将组绑定到 ClusterRole，使用 `subject` 绑定中的 Group 类型：

```
...
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: my-great-groups-name
...
```

在 Kubernetes 中，组由身份验证提供程序提供。Kubernetes 内部没有强烈的组概念，只是一个身份可以是一个或多个组的一部分，并且这些组可以通过绑定与 Role 或 ClusterRole 关联。

# 摘要

当您从一个小团队和小集群开始时，每个团队成员拥有等效的集群访问权限就足够了。但随着团队的增长和产品变得更加关键，限制对集群部分区域的访问变得至关重要。在设计良好的集群中，访问权限被限制在最少的人员和能力集上，以便高效地管理集群中的应用程序。

理解 Kubernetes 如何实现 RBAC 及其如何用于控制集群访问权限对于开发人员和集群管理员都非常重要。与构建测试基础设施一样，最佳实践是尽早设置 RBAC。从正确的基础开始要比后期尝试改装要容易得多。希望本章提供的信息为您在集群中添加 RBAC 提供了必要的基础。
