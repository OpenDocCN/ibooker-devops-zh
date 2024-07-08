# 附录 A. 图表 API 版本

本附录涵盖了图表 API 版本 2 和 1（传统）之间的区别。

每个图表的*Chart.yaml*文件中指定了图表 API 版本，并且被 Helm 用来确定如何解析图表以及提供哪些功能集。

对于新图表，通常应使用 API 版本 2。然而，许多公开可用的图表是在 API 版本 2 生成之前创建的，并且使用 1，即传统的 API 版本。在这里，我们将详细介绍这两个 API 版本及其不同之处。

# API 版本 2

Chart API 版本 2 是在 Helm 3 中引入的当前 API 版本。这是使用`helm create`创建新图表时使用的默认 API 版本。

使用 API 版本 2 的图表保证受 Helm 3 支持，但不一定受 Helm 2 支持。如果您只计划支持 Helm 3 及以上版本，则建议仅使用此 API 版本。

## *Chart.yaml*文件

以下是使用 API 版本 2 的图表的*Chart.yaml*文件示例：

```
apiVersion: v2 ![1](img/1.png)
name: lemon
version: 1.2.3
type: application
description: When life gives you lemons, do the DevOps
appVersion: 2.0.0
home: https://example.com
icon: https://example.com/img/lemon.png
sources:
  - https://github.com/myorg/mychart
keywords:
  - fruit
  - citrus
maintainers:
  - name: Carly Jenkins
    email: carly@mail.cj.example.com
    url: https://cj.example.com
  - name: William James Spode
    email: william.j@mail.wjs.example.com
    url: https://wjs.example.com
deprecated: false
annotations:
  sour: 1
kubeVersion: ">=1.14.0"
dependencies:
  - name: redis
    version: ~10.5.7
    repository: https://kubernetes-charts.storage.example.com/
    condition: useCache,redis.enabled
  - name: postgresql
    version: 8.6.4
    repository: @myrepo
    tags:
      - database
      - backend
```

![1](img/#co_chart_api_versions_CO1-1)

表示图表 API 版本 2 的字段

此文件中的顶级字段将在以下子节中详细描述。

### 字段：apiVersion

*必需*

此图表的 API 版本。

此字段应始终设置为`v2`。

### 字段：名称

*必需*

图表的名称。

在大多数情况下，这应与您的应用程序名称相对应（即`lemon`）。如果您的应用程序分为多个可安装组件，则将此名称后缀为组件的描述是常见的做法；例如，`lemon-frontend`、`lemon-backend`等。

图表名称必须由小写字母、数字和破折号（`-`）组成。

### 字段：版本

*必需*

图表的当前版本，严格使用语义化版本 2 格式化。

### 字段：类型

*必需*

指定图表类型，可以是以下两种类型之一：

`应用程序`

典型的可安装图表

`库`

包含常见定义的不可安装图表，意味着作为依赖图表包含。

此字段是 API 版本 2 独有的。在 API 版本 1 中，所有图表都被视为`应用程序`图表。有关`库`图表的更多信息，请参阅第六章。

### 字段：描述

图表的简单、一句话描述。

### 字段：appVersion

表示图表代表的应用程序的版本。

此字段应与您正在部署的*软件*版本匹配，而不是图表本身。例如，如果您正在创建一个新的内部图表来部署自定义配置的 Nginx 1.18.0，则`appVersion`字段将是`1.18.0`，而`version`字段则可能更像是`0.1.0`（初始版本）。

### 字段：主页

图表和/或应用程序主页的绝对 URL。

### 字段：图标

图表的图标可以使用作为此图表图标的图像的绝对 URL。

这个字段通常被像 Artifact Hub 这样的服务用来显示可供下载的图表的适当图像。

### 字段：sources

一个或多个图表源代码的绝对 URL（如果可用）。

### 字段：keywords

一个图表所代表的关键字或主题列表。

这些字段被像 Artifact Hub 这样的服务用来按类别分组图表或进一步增强搜索功能。

### 字段：maintainers

一组用于维护图表的人员的姓名/电子邮件/URL 组合列表。

### 字段：deprecated

图表是否已被弃用。

这个字段被像 Artifact Hub 这样的服务用来决定何时移除图表列表。

### 字段：annotations

额外映射的图表信息，由 Helm 未解释，并提供给其他应用程序检查。

注意：这个字段*与 Kubernetes 注解没有任何有意义的联系*；但是，根据您如何决定使用这个字段，您可以选择将其定义为 Kubernetes 特定的注解。

### 字段：kubeVersion

一个 SemVer 约束，指定图表正确安装所需的最低 Kubernetes 版本。

一些图表可能使用的 Kubernetes 资源类型和 API 组仅在特定版本的 Kubernetes 上可用。如果操作员尝试在与目标集群的 `kubeVersion` 不兼容的情况下安装图表，则在任何 Kubernetes 资源被配置之前将发生错误。

### 字段：dependencies

一个图表的依赖列表。

当您运行 `helm dependency update` 时，这里列出的图表依赖关系将被协商并适当地放置到 *charts/* 子目录中。

至少，`dependencies` 块下的每个条目应包含一个 `name` 子字段，以及一个 `repository` 或一个 `alias` 子字段。`repository` 应是指向有效图表仓库（服务 */index.yaml*）的绝对 URL。`alias` 应该是字符 “@” 后跟以前添加的图表仓库的名称（例如，*`@myrepo`*）。

关于如何使用图表依赖关系的更多信息，请参阅 “图表依赖关系”。

## Chart.lock 文件

当一个图表在 *Chart.yaml* 的 `dependencies` 字段下列出依赖关系时，会生成一个名为 *Chart.lock* 的特殊文件，并在每次运行 `helm dependency update` 命令时更新。当一个图表包含 *Chart.lock* 文件时，操作员可以运行 `helm dependency build` 来生成 *charts/* 目录，而无需重新协商依赖关系。

这里是一个基于 *Chart.yaml* 示例中指定的依赖关系生成的 *Chart.lock* 文件的示例：

```
dependencies:
- name: redis
  repository: https://kubernetes-charts.storage.example.com/
  version: 10.5.7
- name: postgresql
  repository: https://charts.example.com/
  version: 8.6.4
digest: sha256:529608876e9f959460d0521eee3f3d7be67a298a4c9385049914f44bd75ac9a9
generated: "2020-07-17T11:10:34.023896-05:00"
```

类似 `conditions` 和 `tags` 的动态字段被剥离，这个文件简单地包含了在更新过程中为每个依赖项解析的 `repository`、`name` 和 `version`，以及一个 `digest`（SHA-256）和一个 `generated` 时间戳。

注意，PostgreSQL 依赖项的`alias: "@myrepo"`设置已转换为`repository: https://charts.example.com/`。这意味着在更新依赖项之前，通过以下命令添加了一个图表仓库：

```
$ helm repo add myrepo https://charts.example.com/
```

# API 版本 1（遗留）

图表 API 版本 1 是最初的 API 版本，也是 Helm 2 所认可的唯一版本。*Chart.yaml* 中的`apiVersion`字段首次在 Helm 3 中引入，Helm 2 不认可该字段。在 Helm 2 中，默认假定所有图表都遵循 API 版本 1。在 Helm 3 中，`apiVersion` 是严格必需的。

使用 API 版本 1 的图表确保同时受到 Helm 2 和 Helm 3 的支持，但可能无法支持 Helm 3 未来可能提供的某些功能。

## Chart.yaml 文件

用于使用 API 版本 1 的图表的*Chart.yaml*文件格式几乎与使用 API 版本 2 的图表相同，有一些显著差异。

以下是使用 API 版本 1 的图表的*Chart.yaml*文件示例：

```
apiVersion: v1 ![1](img/1.png)
name: lemon
version: 1.2.3
description: When life gives you lemons, do the DevOps
appVersion: 2.0.0
home: https://example.com
icon: https://example.com/img/lemon.png
sources:
  - https://github.com/myorg/mychart
keywords:
  - fruit
  - citrus
maintainers:
  - name: Carly Jenkins
    email: carly@mail.cj.example.com
    url: https://cj.example.com
  - name: William James Spode
    email: william.j@mail.wjs.example.com
    url: https://wjs.example.com
deprecated: false
annotations:
  sour: 1
kubeVersion: ">=1.14.0"
tillerVersion: ">=2.12.0"
engine: gotpl
```

![1](img/#co_chart_api_versions_CO2-1)

表示图表 API 版本 1 的字段

### 与 v2 的差异

与 API 版本 2 的*Chart.yaml*示例相比，有一些细微差异：

+   `apiVersion`字段设置为`v1`（注：在 Helm 2 中，此字段不是严格必需的）。

+   `type`字段缺失。在 API 版本 1 中不存在库图表的概念。

+   `dependencies`字段缺失。在 API 版本 1 中，图表依赖项在名为*requirements.yaml*的专用文件中指定（稍后在本节中描述）。

+   存在两个额外字段：`tillerVersion` 和 `engine`。

###### 注意

在许多方面，这两个图表 API 版本实质上可以被视为 Helm 2 图表（`v1`）与 Helm 3 图表（`v2`）的对比。特别是自从 Helm 3 发布时引入了图表 API 版本 2 以来。

这些版本没有被命名为`v2`和`v3`（标示 Helm 版本）是因为图表的 API 版本独立于 Helm CLI 的 API 版本。

例如，如果 Helm 4 发布，可能仍然使用图表 API 版本 2。同样地，如果出于某些原因确定图表 API 版本 2 不足，可能会在另一个重大 Helm 发布之前引入新的图表 API 版本 3。

### 字段：tillerVersion（遗留）

指定安装图表所需的 Tiller 版本的 SemVer 约束。

Tiller 是仅在 Helm 2 中使用的遗留 Helm 服务器端组件。在使用 Helm 3 时，此字段完全被忽略。

### 字段：engine（遗留）

要使用的模板引擎的名称。默认为*gotpl*。

## *requirements.yaml* 文件（遗留）

在 API 版本 1 中，还有一个名为*requirements.yaml*的额外文件，指定图表的依赖关系。该文件的格式与 API 版本 2 中定义的`dependencies`字段完全相同。

这里是一个独立的*requirements.yaml*文件示例：

```
dependencies:
  - name: redis
    version: ~10.5.7
    repository: https://kubernetes-charts.storage.example.com/
    condition: redis.enabled
  - name: postgresql
    version: 8.6.4
    repository: "@myrepo"
    tags:
      - database
      - backend
```

关于每个子字段的详细描述，请参阅标题为“字段：dependencies”的子节，位于 API 版本`v2`下。在 API 版本`v2`中，此文件的内容直接在*Chart.yaml*中定义。

## *requirements.lock*文件（遗留）

在 API 版本 1 中，图表依赖锁定文件的名称为*requirements.lock*。此文件在格式和用途上与 API 版本 2 中描述的*Chart.lock*文件完全相同，只是名称不同。有关更多信息，请参阅标题为“Chart.lock 文件”的子节，位于 API 版本 2 下。
