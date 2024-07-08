# 第三章：GitOps：Git 作为事实的来源

在本章中，我们将专注于 GitOps，这是 Azure Arc 可用 Kubernetes 的重要组成部分。GitOps 可能是一个新概念，因此我们将花时间定义和解析它是什么，它是如何工作的，以及您如何在 Azure Arc 可用 Kubernetes 中使用它。

# 什么是 GitOps？

GitOps 是 Weaveworks 在 2017 年 8 月的一篇[博客文章](https://oreil.ly/eYIzF)中首次提出的术语。它是云原生应用程序的操作模型模式，将应用程序和声明性基础设施代码存储在 Git 中作为事实的来源。它用于自动化持续交付。Weaveworks 在其 Kubernetes 环境中使用了 GitOps 操作模型模式。

在 Git 中，存在描述系统状态的代码，包括应用程序、配置、仪表板、监控等。在 GitOps 中，还有一种软件（运算符），它确保云原生生活环境的状态与 Git 中描述的所需状态相匹配。在撰写本文时，大多数运算符（如 Flux 和 ArgoCD）都设计用于与 Kubernetes 配合使用；但 GitOps 并不限于 Kubernetes。例如，有一个名为 Kubestack 的[Terraform](https://oreil.ly/CzEP9) GitOps 框架。您计划使用 Flux 或 ArgoCD 运算符的 Git 存储库可以包含 YAML 格式的 Kubernetes 清单。这些清单应描述有效的 Kubernetes 对象，如命名空间、ConfigMaps、Secrets、Deployments、Pods、Services、Ingress、DaemonSets 等等。Git 存储库还可以包含应用程序的 Helm Charts。

GitOps 的原则包括：

声明性配置

系统状态以声明方式描述。

版本控制的、不可变的存储

Git 是事实的来源；所需的系统状态在 Git 中进行版本控制。

自动化交付

Git 是由自主代理执行的操作（`创建`，`更改`，`删除`）的单一位置。

自主代理

软件运算符强制执行所需状态并警示漂移。

闭环

自动化交付已批准的系统状态更改。

深入了解 GitOps，建议从 [“GitOps：通过拉取请求进行操作”](https://oreil.ly/NR62h) 博客文章开始。

# Azure Arc 如何利用 GitOps

Azure Arc 使用 GitOps 来覆盖两个方面：首先是 Kubernetes 的配置，其次是应用程序的部署到 Kubernetes。配置可以包括诸如命名空间、ConfigMaps、Secrets、Ingress Controllers、Ingress 等对象。应用程序可以是诸如部署、Pods、服务和 Helm Charts 等内容。

Azure Arc 利用 Flux，一个开源的 GitOps 运算符。Flux 由 Weaveworks 构建，并且是 CNCF Sandbox 项目的一部分。

在 Azure Arc 启用的 Kubernetes 中，GitOps 是通过运行在 Kubernetes 集群上的 Flux 操作员（作为 Pod）来连接 Kubernetes 集群和 Git 仓库的一种方式。此 Flux 操作员用于同步 Kubernetes 集群配置和 Git 仓库中的期望状态配置，目标是通过 `create`、`change` 和 `delete` 操作使两者保持一致。

这种同步是通过 Flux 操作员的拉取方式实现的，Flux 将持续轮询 Git 仓库，查找任何新内容，并在 Kubernetes 集群中进行创建或更新。Flux 操作员还具有持续轮询容器注册表或 Helm 仓库的能力，以获取 Kubernetes 集群中所需的添加或更改。

Azure Arc 启用的 Kubernetes 集群和 Git 仓库连接存储在 ARM 中作为名为 *Microsoft.KubernetesConfiguration/sourceControlConfigurations* 的扩展资源。此内容存储在 Azure Cosmos DB 数据库中，静态加密。我们可以通过 Azure 门户或 Azure CLI 配置此设置。`sourceControlConfiguration` 属性包括：

+   配置名称

+   操作员实例名称

+   操作员命名空间

+   仓库 URL

+   操作员范围（命名空间/集群）

+   操作员类型

+   操作员参数

+   Helm（启用/禁用）

对于每个 Azure Arc 启用的 Kubernetes 集群，您可以拥有多个 `sourceControlConfigurations`，这些配置可以针对命名空间进行范围设置。这有助于处理多环境和多租户场景。

Azure Arc 启用的 Kubernetes 还可以使用 GitOps 管理大量 Kubernetes 集群的配置。通过使用 Azure 策略，在 Kubernetes 集群加入 Azure Arc 后立即应用所需的 `sourceControlConfigurations` 来实现这一目标。如果您希望跨多个 Kubernetes 集群应用相同的配置（如服务网格、监控代理、入口控制器等），则可以使用此功能。
