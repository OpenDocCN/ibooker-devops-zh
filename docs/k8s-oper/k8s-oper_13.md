# 附录 C. 基于角色的访问控制（RBAC）

当操作员 SDK 生成操作员项目（无论是基于 Helm、Ansible 还是 Go 的操作员）时，它会创建多个清单文件来部署操作员。其中许多文件授予部署操作员在其生命周期中执行各种任务所需的权限。

操作员 SDK 生成了三个与操作员权限相关的文件：

*deploy/service_account.yaml*

Kubernetes 不提供作为用户进行身份验证的方法，而是提供了一种程序化的身份验证方法，即*服务帐户*。服务帐户在操作员 pod 对 Kubernetes API 发出请求时充当身份。此文件仅定义服务帐户本身，您无需手动编辑它。有关服务帐户的更多信息，请参阅[Kubernetes 文档](https://oreil.ly/8oXS-)。

*deploy/role.yaml*

此文件为服务帐户创建和配置了一个*角色*。角色在与集群 API 交互时决定了服务帐户拥有的权限。出于安全原因，操作员 SDK 生成此文件时授予了极其广泛的权限，因此您在将操作员部署到生产环境之前需要进行编辑。在下一节中，我们将详细解释如何在此文件中精细调整默认权限。

*deploy/role_binding.yaml*

此文件创建了一个*角色绑定*，将服务帐户映射到角色。您无需对生成的文件进行任何更改。

# 精细调整角色

在其最基本的层面上，角色将资源类型映射到用户或服务帐户可以对这些类型资源执行的操作（在角色资源术语中称为“动词”）。例如，以下角色为部署授予了查看权限（但不包括创建或删除权限）：

```
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

由于操作员 SDK 不知道操作员需要与集群交互的程度，因此默认角色允许在多种 Kubernetes 资源类型上执行所有操作。以下片段摘自由 SDK 生成的操作员项目，通配符`*`允许对给定资源执行所有操作：

```
...
- apiGroups:
  - ""
  resources:
  - pods
  - services
  - endpoints
  - persistentvolumeclaims
  - events
  - configmaps
  - secrets
  verbs:
  - '*'
- apiGroups:
  - apps
  resources:
  - deployments
  - daemonsets
  - replicasets
  - statefulsets
  verbs:
  - '*'
...
```

毫不奇怪，将如此开放和广泛的权限授予服务帐户被视为一种不良实践。您应根据操作员的范围和行为对所做的特定更改进行限制。一般来说，您应尽可能限制访问权限，同时确保操作员可以正常运行。

例如，以下角色片段提供了访客站点操作员所需的最小功能：

```
...
- apiGroups:
  - ""
  resources:
  - pods
  - services
  - secrets
  verbs:
  - create
  - list
  - get
- apiGroups:
  - apps
  resources:
  - deployments
  verbs:
  - create
  - get
  - update
...
```

关于配置 Kubernetes 角色的详细信息超出了本书的范围。您可以在[Kubernetes RBAC 文档](https://oreil.ly/osBC3)中找到更多信息。
