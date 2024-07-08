# 第二章：使用 Helm

Helm 提供了一个命令行工具，名为 `helm`，可以使用所有与 Helm charts 相关的主要功能。在本章中，我们将探索 `helm` 客户端的主要功能。在此过程中，我们将了解 Helm 如何与 Kubernetes 交互。

我们将首先看看如何安装和配置 Helm，并逐步学习 Helm 的主要命令组。然后，我们将介绍如何查找和学习软件包，以及如何安装、升级和删除它们。

# 安装和配置 Helm 客户端

Helm 提供了一个能够执行所有主要 Helm 任务的单一命令行客户端。这个客户端名为 `helm`。虽然有许多其他工具可以处理 Helm charts，但这是由 Helm 核心维护者维护的官方通用工具，也是本章和下一章的主题。

`helm` 客户端是用一种名为 Go 的编程语言编写的。与 Python、JavaScript 或 Ruby 不同，Go 是一种编译语言。一旦 Go 程序编译完成，您就不需要任何 Go 工具来运行或处理二进制文件。

因此，我们将首先介绍下载和安装静态二进制文件的过程，然后简要介绍如果您希望的话，可以获取并从 Go 源代码编译的过程。

## 安装预编译二进制文件

每次 Helm 维护者发布新版本的 `helm` 时，项目都会提供针对多个常见操作系统和架构的新签名二进制版本的构建。截至撰写本文时，Helm 的预构建版本适用于从 64 位 Intel/AMD 到 ARM、s390 和 PPC 的 Linux、Windows 和 macOS 等各种架构。这意味着您可以在从树莓派到超级计算机的任何设备上运行 Helm。

Helm 发行版的详尽列表位于[发布页面](https://oreil.ly/L_My5)。发布页面将显示按时间顺序排列的发行版列表，最新发行版位于顶部。

### Helm 版本号说明

直到 2020 年 11 月，仍在积极维护两个不同的 Helm 主要版本。当前稳定的 Helm 主要版本是版本 3。当您访问 Helm 下载页面时，您可能会看到可以下载的两个版本。由于版本按时间顺序列出，甚至可能会出现 Helm 2 的版本比最新的 Helm 3 版本更新的情况。您应该使用 Helm 3。

Helm 遵循称为[语义化版本](https://semver.org)（SemVer）的版本控制约定。在语义化版本中，版本号传达了关于发布版中可期待内容的含义。由于 Helm 遵循此规范，用户可以仅仔细阅读版本号即可对发行版期望有所了解。

在其核心，语义版本具有三个数字组件和一个可选的*稳定性标记*（用于 alpha、beta 和候选发布版）。以下是一些示例：

+   `v1.0.0`

+   `v3.3.2`

+   `v2.4.22-alpha.2`

让我们首先讨论数值组件。

我们经常将这种格式概括为 `X.Y.Z`，其中 `X` 是 *主版本*，`Y` 是 *次版本*，`Z` 是 *补丁发布*：

+   主要的版本号往往不经常增加。它表示 Helm 发生了重大变化，并且其中一些变化可能会破坏与旧版本的兼容性。Helm 2 和 Helm 3 之间的差别很大，需要进行版本迁移的工作。

+   次要版本号表示功能添加。例如 3.2.0 和 3.3.0 之间的区别可能是增加了一些小的新功能。但是，版本之间没有 *重大变更*。（有一个例外：安全修复可能需要进行重大变更，但我们会明确宣布这种情况。）

+   补丁版本号表示在此版本与上一个版本之间仅进行了向后兼容的错误修复。建议始终保持在最新的补丁版本。

当您看到一个带有稳定性标记的发布版本，例如 `alpha.1`，`beta.4` 或 `rc.2`，附加到发布版本号时，这意味着该版本被视为预发布版本，尚未准备好用于主流生产环境。特别是在主要或次要更新之前，Helm 经常发布 *候选版本*。这给社区一个机会，在正式发布之前就稳定性、兼容性和新功能提供反馈。

有了这个理解，我们准备进行实际安装。

### 下载二进制文件

从仓库安装 Helm 的最简单方法是直接访问 [发布页面](https://oreil.ly/L_My5)，并下载最新的 Helm 3 版本。

在 Windows 上，下载文件是一个 ZIP 存档，包含一个 *README.md* 文本文件，一个 *LICENSE* 文本文件和 *helm.exe*。

在 macOS 和 Linux 上，下载将以一个可以用 `tar -zxf` 命令解压的 gzip 压缩的 tar 存档（`.tar.gz`）形式存在。与 Windows 版本类似，它将包含一个 *README.md* 文本文件，一个 *LICENSE* 文本文件和 `helm` 二进制文件。如果您正在使用 Windows Subsystem for Linux（WSL），您应该将 Linux AMD64 版本安装到您的 WSL 实例中。

无论您使用哪个操作系统，二进制文件是运行 Helm 所需的唯一文件，您可以将其放在系统上任何您喜欢的位置。它应该被预先标记为可执行文件，但在类 UNIX 环境中，您偶尔可能需要运行命令 `chmod helm +x` 来设置 Helm 为可执行文件。

###### 注意

当使用像 Homebrew（macOS）、Snap（Linux）或 Chocolatey（Windows）这样的软件包管理器安装时，`helm` 将安装在标准位置，并立即通过命令行对您可用。

安装完毕 `helm` 后，您应该能够运行 `helm help` 命令并查看 Helm 帮助文本。

### 使用获取脚本进行安装

在 macOS 和 Linux 上，您可能更喜欢运行一个 shell 脚本，它会确定要安装的 Helm 版本，并自动完成安装。

通过这种方式安装的常规命令序列如下：

```
$ curl -fsSL -o get_helm.sh \
https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

上述命令会获取 `get_helm.sh` 脚本的最新版本，然后使用它来查找并安装 Helm 3 的最新版本。

对于自动安装 Helm 的系统，例如持续集成（CI）系统，如果始终拥有最新的 Helm 版本很重要，我们建议使用这种方法。

## 构建源代码的指南

除非您已经熟悉 Go 开发，否则从源代码构建 Helm 可能会是一项艰巨的任务。您需要一个版本的 `make` 命令。因为 `Makefile` 风格的构建脚本没有遵循单一标准，不是所有版本的 `make` 都能用于构建 Helm。GNU Make 是在 Linux 和 Mac 上最常用的，也是 Helm 核心开发人员最常用的，因此是一个安全的选择。您还需要 `gcc` 编译器和完整的 Go 工具链。

除此之外，Helm 还需要几个辅助工具。幸运的是，当您第一次运行 `make` 时，它将尝试安装任何缺失的附加工具。

虽然它们并非绝对必需，但您可能还需要 `git` 工具和 `kubectl` 命令。`git` 工具允许您直接使用 Helm 源代码仓库，而不是下载源代码包。当然，`kubectl` 是用于与您的 Kubernetes 集群交互的。虽然这不是 *构建* Helm 所必需的，但在检查 Helm 是否按您希望的方式运行时，这无疑是必需的。

一旦安装和配置好工具，您只需切换到包含 Helm 源代码的文件夹（即包含 *README.md* 和 Makefile 文件的目录），然后运行 `make build`。第一次运行此命令时，至少需要几分钟。构建系统必须获取大量依赖项，包括 Kubernetes 的大部分源代码，并对其进行编译。

###### 提示

编译 Helm 初次可能会令人望而却步，尤其是对于不熟悉 Go 编程语言的人来说。 Kubernetes 是一个复杂的平台，因此 Helm 的源代码庞大且难以构建。计划至少花上一两个小时准备一个新的环境来安装 Helm。

要验证 Helm 是否正常运行（特别是如果您修改了源代码），可以运行 `make test`。这将构建代码，运行各种检查器和 linter，并运行 Helm 的单元测试。如果您计划向 Helm 的核心维护者提交任何更改请求，在查看您请求的更改之前，此命令 *必须* 通过。

当 Helm 编译完成时，它将位于源代码旁边的一个名为 *bin/* 的子目录中。它不会自动添加到可执行路径中，因此要执行您刚刚构建的版本，您可能需要指定相对或确切的路径（例如，*./bin/helm* 或 *$GOPATH/src/helm.sh/helm/bin/helm*）。

如果命令 `helm version` 正确执行，您可以放心地确认您已正确编译 Helm。

从这里，您可以按照详细的[开发人员指南](https://oreil.ly/X9-Ii)了解更多信息。和往常一样，如果遇到问题，可以在 Kubernetes Slack 服务器的 `helm-users` 频道中寻求帮助。

## 使用 Kubernetes 集群

Helm 与 Kubernetes API 服务器直接交互。因此，Helm 需要能够连接到 Kubernetes 集群。Helm 试图通过读取 `kubectl`（主要的 Kubernetes 命令行客户端）使用的相同配置文件来自动执行此操作。

Helm 将尝试通过读取环境变量 *$KUBECONFIG* 来查找这些信息。如果未设置，它将在与 `kubectl` 相同的默认位置查找（例如，在 UNIX、Linux 和 macOS 上为 *$HOME/.kube/config*）。

您还可以通过环境变量（`HELM_KUBECONTEXT`）和命令行标志（`--kube-context`）覆盖这些设置。可以通过运行 `helm help` 查看环境变量和标志的列表。

Helm 的维护者建议使用 `kubectl` 管理您的 Kubernetes 凭据，并让 Helm 仅仅自动检测这些设置。如果您尚未安装 `kubectl`，开始的最佳方法是参考[官方 Kubernetes 安装文档](https://oreil.ly/pHZIh)。

## 开始使用 Helm

无论您是从源代码构建 Helm 还是使用上述方法之一进行安装，在这一点上您应该在系统上拥有 `helm` 命令可用。从这里开始，我们将假设可以使用 `helm` 命令执行 Helm（与之前部分讨论的完整或相对路径相反）。

接下来，我们将看看 Helm 最常见的启动工作流程：

1.  添加一个图表存储库。

1.  查找要安装的图表。

1.  安装 Helm 图表。

1.  查看已安装的内容列表。

1.  升级您的安装。

1.  删除安装。

然后，在下一章中，我们将深入探讨 Helm 的一些附加功能，并通过这样做了解更多有关 Helm 如何工作的信息。

# 添加图表存储库。

图表存储库是一个独立的主题，在第七章中我们将详细讨论它们。但任何使用 Helm 的人都必须了解有关图表存储库的一些基础知识。

Helm 图表是可以安装到您的 Kubernetes 集群中的*独立包*。在图表开发过程中，您通常只需使用存储在本地文件系统上的一个图表来工作。

但是当涉及到共享图表时，Helm 描述了一种标准格式，用于索引和共享有关 Helm 图表的信息。一个 Helm *图表仓库* 简单地是一组文件，通过网络可访问，符合 Helm 规范以索引包。

###### 注意

Helm 3 引入了一个实验性功能，用于将 Helm 图表存储在不同类型的仓库中：Open Container Initiative (OCI) 注册表（有时被称为 Docker 注册表）。在这个后端，一个 Helm 图表可以与 Docker 镜像并存储。虽然这个功能目前支持不广泛，但它可能成为 Helm 包存储的未来。这在 第七章 中更详细地讨论。

互联网上有许多——也许成千上万——图表仓库。找到流行的仓库最简单的方法是使用你的网络浏览器导航到 [Artifact Hub](https://artifacthub.io)。在那里，你将找到成千上万的 Helm 图表，每个都托管在适当的仓库上。

要开始，我们将安装流行的 [Drupal 内容管理系统](https://www.drupal.org)。这是一个很好的示例图表，因为它涵盖了许多 Kubernetes 类型，包括 `Deployment`、`Service`、`Ingress` 和 `ConfigMap`。

Helm 2 默认安装了一个 Helm 仓库。`stable` 图表仓库曾一度是生产准备好的 Helm 图表的官方来源。但我们意识到将图表集中到一个仓库对少数维护者来说过于繁重，并且对图表贡献者来说是令人沮丧的。

在 Helm 3 中，没有默认仓库。鼓励用户使用 Artifact Hub 找到他们需要的内容，然后添加他们偏好的仓库。

Drupal 的 Helm 图表位于最完善的图表仓库之一：Bitnami 的官方 Helm 图表中。你可以查看 Artifact Hub 上的 [Drupal 图表的条目](https://oreil.ly/baxxf) 以获取更多信息。

###### 注意

小部分 Bitnami 开发人员是设计 Helm 仓库系统核心贡献者之一。他们为图表开发建立了 Helm 的最佳实践，并编写了许多被广泛使用的图表。

添加 Helm 图表可以使用 `helm repo add` 命令完成。几个 Helm 仓库命令被分组在 `helm repo` 命令组下：

```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
```

`helm repo add` 命令将添加一个名为 `bitnami` 的仓库，指向网址 [*https://charts.bitnami.com/bitnami*](https://charts.bitnami.com/bitnami)。

现在，我们可以通过运行第二个 `repo` 命令来验证 Bitnami 仓库是否存在。

```
$ helm repo list
NAME    URL
bitnami https://charts.bitnami.com/bitnami
```

这个命令显示了所有安装的 Helm 仓库。现在，我们只看到刚刚添加的 Bitnami 仓库。

一旦我们添加了一个仓库，其索引将被本地缓存，直到我们下次更新它（参见 第七章）。现在我们能做的一件重要的事情是搜索该仓库。

# 搜索图表仓库

尽管我们知道，在 Artifact Hub 上查看过 Drupal 图表存在于此存储库中，但仍然有必要从命令行搜索它。通常情况下，搜索不仅是查找可以安装的图表的有用方式，还可以查找可用的版本。

首先，让我们搜索 Drupal 图表：

```
$ helm search repo drupal
NAME            CHART VERSION   APP VERSION     DESCRIPTION
bitnami/drupal  7.0.0           9.0.0           One of the most versatile open...
```

我们简单搜索了术语*drupal*。Helm 不仅会搜索包名称，还会搜索标签和描述等其他字段。因此，我们可以搜索*content*，看到 Drupal 因为它是内容管理系统而被列出：

```
$ helm search repo content
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
bitnami/drupal          7.0.0           9.0.0           One of the most versa...
bitnami/harbor          6.0.1           2.0.0           Harbor is an an open...
bitnami/joomla          7.1.18          3.9.19          PHP content managemen...
bitnami/mongodb         7.14.6          4.2.8           NoSQL document-orient...
bitnami/mongodb-sharded 1.4.2           4.2.8           NoSQL document-orient...
```

虽然 Drupal 是第一个结果，但请注意还有许多其他图表在描述文本中包含*content*一词。

默认情况下，Helm 尝试安装图表的最新稳定版本，但您可以覆盖此行为并安装特定版本的图表。因此，查看图表的摘要信息以及确切的图表版本通常非常有用：

```
$ helm search repo drupal --versions
NAME            CHART VERSION   APP VERSION     DESCRIPTION
bitnami/drupal  7.0.0           9.0.0           One of the most versatile op...
bitnami/drupal  6.2.22          8.9.0           One of the most versatile op...
bitnami/drupal  6.2.21          8.8.6           One of the most versatile op...
bitnami/drupal  6.2.20          8.8.5           One of the most versatile op...
bitnami/drupal  6.2.19          8.8.5           One of the most versatile op...
...
```

Drupal 图表有几十个版本。前面的示例已经被截断，仅显示了前几个顶级版本。

# 图表和应用版本

*图表版本*是 Helm 图表的版本。*应用程序版本*是打包在图表中的应用程序的版本。Helm 使用图表版本来做出版本决策，例如哪个包是最新的。正如我们在前面的示例中看到的，多个图表版本可能包含相同的应用程序版本。

# 安装一个包

在接下来的章节中，我们将深入探讨 Helm 中包安装的工作原理。不过，在本节中，我们将查看安装 Helm 图表的基本机制。

在 Helm 中，至少需要两个信息来安装图表：安装名称和您想安装的图表。

请记住，在上一章中，我们区分了*安装*和*图表*。这在安装和升级过程中是一个重要的区别。在操作系统包管理器中，我们可能请求安装某个软件包。但在操作系统上几乎不会需要多次安装完全相同的软件包。Kubernetes 集群则不同。在 Kubernetes 中说“我想为应用程序 A 安装一个 MySQL 数据库，并为应用程序 B 安装第二个 MySQL 数据库”是完全有道理的。即使两个数据库是完全相同版本并且具有相同的配置，为了适当地管理我们的应用程序，我们可能希望运行两个实例。

因此，Helm 需要一种方法来区分同一个图表的不同实例。因此，一个图表的*安装*是该图表的特定实例。一个图表可能有多个安装。当我们运行`helm install`命令时，我们需要给出一个安装名称以及图表名称。因此，最基本的安装命令看起来像这样：

```
$ helm install mysite bitnami/drupal
NAME: mysite
LAST DEPLOYED: Sun Jun 14 14:46:51 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
*******************************************************************
*** PLEASE BE PATIENT: Drupal may take a few minutes to install ***
*******************************************************************

1\. Get the Drupal URL:

  You should be able to access your new Drupal installation through

  http://drupal.local/

2\. Login with the following credentials

  echo Username: user
  echo Password: $(kubectl get secret --namespace default mysite-drupal...
```

之前将创建一个`bitnami/drupal`图表的实例，并将此实例命名为`mysite`。

当安装命令运行时，它将返回大量信息，包括有关如何开始使用 Drupal 的用户界面说明。

###### 注意

在以后的`helm install`示例中，出于简洁起见，我们将省略返回的输出。但是，在使用 Helm 时，您将看到每个安装的输出。在下一章中，我们还将看到如何使用`helm get`命令再次查看该输出。

此时，集群中现在有一个名为`mysite`的实例。如果我们尝试重新运行前面的命令，我们将不会得到第二个实例。相反，我们会因为名称`mysite`已被使用而收到错误消息：

```
$ helm install mysite bitnami/drupal
Error: cannot re-use a name that is still in use
```

还需要进一步澄清一点。在 Helm 2 中，实例名称是集群范围的。您只能在*每个集群*中有一个名为`mysite`的实例。在 Helm 3 中，命名已更改。现在实例名称限定在 Kubernetes 命名空间内。只要它们分别位于不同的命名空间中，我们可以安装两个名为`mysite`的实例。

例如，在 Helm 3 中，以下操作是完全合法的，尽管在 Helm 2 中会生成致命错误：

```
$ kubectl create ns first
$ kubectl create ns second
$ helm install --namespace first mysite bitnami/drupal
$ helm install --namespace second mysite bitnami/drupal
```

这将在`first`命名空间中安装一个名为`mysite`的 Drupal 站点，并在`second`命名空间中安装一个配置相同的实例，也命名为`mysite`。起初可能会感到困惑，但当我们将命名空间视为名称的*前缀*时，情况就变得更清晰了。在这种情况下，我们有一个名为“first mysite”的站点和另一个名为“second mysite”的站点。

# 在 Helm 中始终使用命名空间标志

在处理命名空间和 Helm 时，您可以使用`--namespace`或`-n`标志来指定所需的命名空间。

## 安装时的配置

在前面的示例中，我们以几种不同的方式安装了相同的图表。在所有情况下，它们的配置是相同的。虽然默认配置有时很好，但更常见的是我们想将自己的配置传递给图表。

许多图表将允许您提供配置值。如果我们查看[Drupal 的 Artifact Hub 页面](https://oreil.ly/baxxf)，我们将看到一长串可配置参数。例如，我们可以通过设置`drupalUsername`值来配置 Drupal 管理员帐户的用户名。

###### 注意

在下一章中，我们将学习如何使用`helm`命令获取此信息。

有几种方法可以告诉 Helm 您想要配置哪些值。最好的方法是创建一个包含所有配置覆盖的 YAML 文件。例如，我们可以创建一个设置`drupalUsername`和`drupalEmail`值的文件：

```
drupalUsername: admin
drupalEmail: admin@example.com
```

现在我们有一个文件（通常命名为*values.yaml*），其中包含所有配置信息。由于它在一个文件中，因此很容易复制相同的安装过程。您还可以将此文件检入到版本控制系统中，以跟踪值随时间的变化。Helm 核心维护者认为将配置值存储在 YAML 文件中是一个良好的实践。但请记住，如果配置文件包含敏感信息（如密码或身份验证令牌），则应采取措施确保这些信息不会泄露。

`helm install`和`helm upgrade`都提供了一个`--values`标志，该标志指向一个带有值覆盖的 YAML 文件：

```
$ helm install mysite bitnami/drupal --values values.yaml
NAME: mysite
LAST DEPLOYED: Sun Jun 14 14:56:15 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
*******************************************************************
*** PLEASE BE PATIENT: Drupal may take a few minutes to install ***
*******************************************************************

1\. Get the Drupal URL:

  You should be able to access your new Drupal installation through

  http://drupal.local/

2\. Login with the following credentials

  echo Username: admin
  echo Password: $(kubectl get secret --namespace default mysite-drupal -o js...
```

请注意，在前面的输出中，`Username`现在是`admin`而不是`user`。Helm 的一个很好的特性是，甚至可以使用您提供的值更新帮助文本。

###### 注意

您可以多次指定`--values`标志。有些人使用此功能在一个文件中进行“常见”覆盖，而在另一个文件中进行特定的覆盖。

还有第二个标志可以用来向安装或升级添加单个参数。`--set`标志接受一个或多个直接值。它们不需要存储在 YAML 文件中：

```
$ helm install mysite bitnami/drupal --set drupalUsername=admin
```

这只设置了一个参数，`drupalUsername`。此标志使用简单的`key=value`格式。

配置参数可以是结构化的。也就是说，配置文件可以有多个部分。例如，Drupal 图表具有特定于 MariaDB 数据库的配置。这些参数都被分组到`mariadb`部分中。在我们之前的示例基础上，我们可以像这样覆盖 MariaDB 数据库名称：

```
drupalUsername: admin
drupalEmail: admin@example.com
mariadb:
  db:
    name: "my-database"
```

当使用`--set`标志时，子部分会更加复杂。您需要使用点分符号表示法：`--set mariadb.db.name=my-database`。当设置多个值时，这可能会变得冗长。

通常，Helm 核心维护者建议将配置存储在*values.yaml*文件中（请注意，文件名不一定需要是“values”），只有在绝对必要时才使用`--set`。这样，您可以在操作期间轻松访问使用的值（并可以随时间跟踪这些值），同时保持 Helm 命令的简洁。使用文件还意味着您不必像在命令行设置时那样转义很多字符。

在继续升级之前，我们将快速查看 Helm 命令中最有帮助的一个功能。

# 列出您的安装

我们已经看到，Helm 可以将许多内容安装到同一个集群中，甚至可以在同一个图表的多个实例之间进行安装。而且，在集群上有多个用户时，不同的人可能会将内容安装到同一个命名空间上。

`helm list`命令是一个简单的工具，帮助您查看安装并了解这些安装情况：

```
$ helm list
NAME    NAMESPACE  REVISION  UPDATED       STATUS    CHART           APP VERSION
mysite  default    1         2020-06-14... deployed  drupal-7.0.0    9.0.0
```

此命令将为您提供大量有用的信息，包括发布的名称和命名空间，当前修订号（在第一章中讨论，并在下一节中深入讨论），上次更新时间，安装状态以及图表和应用程序的版本。

与其他命令一样，`helm list`会考虑命名空间。默认情况下，Helm 使用 Kubernetes 配置文件设置的命名空间作为默认值。通常情况下，这个命名空间被命名为`default`。之前，我们将 Drupal 实例安装到`first`命名空间中。我们可以通过`helm list --namespace first`来查看。

当列出所有发布时，一个有用的标志是`--all-namespaces`标志，它将查询您具有权限的所有 Kubernetes 命名空间，并返回找到的所有发布：

```
$ helm list --all-namespaces
NAME    NAMESPACE  REVISION  UPDATED        STATUS     CHART         APP VERSION
mysite  default    1         2020-06-14...  deployed   drupal-7.0.0  9.0.0
mysite  first      1         2020-06-14...  deployed   drupal-7.0.0  9.0.0
mysite  second     1         2020-06-14...  deployed   drupal-7.0.0  9.0.0
```

# 升级安装

当我们谈论 Helm 中的升级时，我们指的是升级安装，而不是图表本身。*安装*是您集群中图表的特定实例。当您运行`helm install`时，它创建该安装。要修改该安装，请使用`helm upgrade`。

在当前上下文中，这是一个重要的区分，因为在 Helm 中升级安装可以包含两种不同类型的更改：

+   您可以升级图表的*版本*

+   您可以升级*安装的配置*

这两者并不是互斥的；您可以同时执行两者。但这确实引入了一个 Helm 用户在讨论其系统时引用的新术语：*发布*是安装的特定配置和图表版本的组合。

当我们首次安装图表时，我们创建了安装的初始发布。让我们称其为发布 1。当我们执行升级时，我们将创建相同安装的新发布：发布 2。当我们再次升级时，我们将创建发布 3。（在下一章中，我们将看到如何通过回滚也创建发布。）

然后，在升级过程中，我们可以使用新配置创建一个发布，使用新的图表版本，或者两者兼而有之。

例如，假设我们安装 Drupal 图表时，关闭了`ingress`。（这将有效阻止外部集群流量进入 Drupal 实例。）

请注意，我们在此处使用`--set`标志来保持示例的紧凑性，但在常规情况下建议使用*values.yaml*文件：

```
$ helm install mysite bitnami/drupal --set ingress.enabled=false
```

关闭`ingress`功能后，我们可以开始设置我们喜欢的站点。然后，当我们准备好时，我们可以创建一个启用`ingress`功能的新发布：

```
$ helm upgrade mysite bitnami/drupal --set ingress.enabled=true
```

在这种情况下，我们正在运行一个仅更改配置的`upgrade`。

在后台，Helm 将加载图表，生成该图表中的所有 Kubernetes 对象，然后查看这些对象与已安装图表版本的区别。它只会发送需要更改的 Kubernetes 对象。换句话说，Helm 将尝试仅修改最小必要的内容。

前面的示例只会更改 `ingress` 配置。数据库或者运行 Drupal 的 Web 服务器都不会发生任何变化。因此，不会重新启动或删除并重新创建任何内容。这有时会让新的 Helm 用户感到困惑，但这是有意设计的。Kubernetes 的哲学是以最简洁的方式进行更改，Helm 也致力于遵循这一哲学。

偶尔，您可能希望强制重启其中一个服务。这并不是您需要使用 Helm 的事情。您可以直接使用 `kubectl` 来重新启动。就像操作系统的软件包管理器不用于重新启动程序一样。同样，您不需要使用 Helm 来重新启动 Web 服务器或数据库。

当图表的新版本发布时，您可能希望升级现有的安装以使用新的图表版本。大多数情况下，Helm 力求简化这个过程：

```
$ helm repo update ![1](img/1.png)
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "bitnami" chart repository
Update Complete. ⎈ Happy Helming!⎈

$ helm upgrade mysite bitnami/drupal ![2](img/2.png)
```

![1](img/#co_using_helm_CO1-1)

从图表仓库获取最新的包。

![2](img/#co_using_helm_CO1-2)

将 `mysite` 发布升级到使用 `bitnami/drupal` 的最新版本。

正如您所见，Helm 的默认策略是尝试使用图表的最新版本。如果您希望保留特定版本的图表，可以显式声明：

```
$ helm upgrade mysite bitnami/drupal --version 6.2.22
```

在这种情况下，即使发布了更新版本，也只会安装 `bitnami/drupal` 的 `6.2.22` 版本。

## 配置数值和升级

了解 Helm 的安装和升级最重要的一点是，每次发布都会全新应用配置数据。以下是一个快速说明：

```
$ helm install mysite bitnami/drupal --values values.yaml ![1](img/1.png)
$ helm upgrade mysite bitnami/drupal ![2](img/2.png)
```

![1](img/#co_using_helm_CO2-1)

使用配置文件进行安装。

![2](img/#co_using_helm_CO2-2)

不使用配置文件进行升级。

这一对操作的结果是什么？安装将使用 *values.yaml* 中提供的所有配置数据，但升级则不会。因此，某些设置可能会被改回其默认值。这通常不是您想要的结果。

# 检查数值

在下一章中，我们将介绍 `helm get` 命令。您可以使用 `helm get values mysite` 查看上次 `helm install` 或 `helm upgrade` 操作使用的值。

Helm 核心维护者建议您在每次安装和升级时提供一致的配置。要将相同的配置应用于两个发布，需要在每个操作中提供这些值：

```
$ helm install mysite bitnami/drupal --values values.yaml ![1](img/1.png)
$ helm upgrade mysite bitnami/drupal --values values.yaml ![2](img/2.png)
```

![1](img/#co_using_helm_CO3-1)

使用配置文件进行安装。

![2](img/#co_using_helm_CO3-2)

使用相同的配置文件进行升级。

我们建议将配置存储在 *values.yaml* 文件中，这样可以轻松复制这种模式。想象一下，如果您使用 `--set` 设置三到四个配置参数，这些命令将会变得多么繁琐！对于每个发布，您都需要记住要设置哪些内容。

虽然我们强烈建议使用此处讨论的模式，并每次指定`--values`，但也有一个升级快捷方式可供使用，它将只重用您发送的上一组值：

```
$ helm upgrade mysite bitnami/drupal --reuse-values
```

`--reuse-values`标志将告诉 Helm 重新加载上次设置的服务器端副本的值，然后使用这些值生成升级。如果您总是*只*重用相同的值，这种方法是可以接受的。然而，Helm 的维护者强烈建议不要尝试将`--reuse-values`与额外的`--set`或`--values`选项混合使用。这样做会使故障排除变得复杂，并且很快会导致不可维护的安装，其中没有人确切知道如何设置某些配置参数。虽然 Helm 保留了一些状态信息，但它不是配置管理工具。建议用户使用自己的工具管理配置，并在每次调用中显式传递该配置给 Helm。

到目前为止，我们已经学会了如何安装、列出和升级安装。在本章的最后一节中，我们将删除一个安装。

# 卸载安装

要删除 Helm 安装，请使用`helm uninstall`命令：

```
$ helm uninstall mysite
```

请注意，此命令不需要图表名称（`bitnami/drupal`）或任何配置文件。它只需要安装名称。在本节中，我们将看看删除如何工作，并简要介绍 Helm 2 和 Helm 3 之间的一个重大变化。

与`install`、`list`和`upgrade`一样，您可以提供`--namespace`标志来指定您希望从*特定命名空间*删除安装：

```
$ helm uninstall mysite --namespace first
```

前述内容将删除我们在本章早期在`first`命名空间中创建的站点。请注意，没有删除多个应用程序的命令。您必须卸载*特定*的安装。

删除可能需要一些时间。较大的应用程序可能需要几分钟，甚至更长时间，因为 Kubernetes 清理所有资源。在此期间，您将无法使用相同的名称重新安装。

## Helm 如何存储发布信息

Helm 3 中的一个重大变化是如何删除有关安装的 Helm 自身数据。本节简要描述了安装是如何被跟踪的，然后总结了解释了为什么 Helm 在 2 和 3 版本之间发生了变化。

当我们首次使用 Helm 安装图表（例如使用`helm install mysite bitnami/drupal`），我们创建了 Drupal 应用程序实例，并且还创建了一个包含发布信息的特殊记录。默认情况下，Helm 将这些记录存储为 Kubernetes `Secret`（尽管还支持其他存储后端）。

我们可以使用`kubectl get secret`来查看这些记录：

```
$ kubectl get secret
NAME                           TYPE                                  DATA   AGE
default-token-vjhx2            kubernetes.io/service-account-token   3      58m
mysite-drupal                  Opaque                                1      13m
mysite-mariadb                 Opaque                                2      13m
sh.helm.release.v1.mysite.v1   helm.sh/release.v1                    1      13m
sh.helm.release.v1.mysite.v2   helm.sh/release.v1                    1      13m
sh.helm.release.v1.mysite.v3   helm.sh/release.v1                    1      7m53s
sh.helm.release.v1.mysite.v4   helm.sh/release.v1                    1      5m30s
```

我们可以在底部看到多个发布记录，每个修订版本都有一个。正如您所看到的，我们通过运行`install`和`upgrade`操作创建了`mysite`的四个修订版本。

在下一章中，我们将看到如何利用这些扩展记录来回滚到安装的先前版本。但我们现在指出这一点是为了说明`helm uninstall`的工作原理。

当我们运行命令`helm uninstall mysite`时，它将加载`mysite`安装的最新发布记录。从该记录中，它将组装一个应从 Kubernetes 中删除的对象列表。然后 Helm 将在返回并删除这四个发布记录之前删除所有这些对象：

```
$ helm uninstall mysite
release "mysite" uninstalled
```

`helm list`命令将不再显示`mysite`：

```
$ helm list
NAME    NAMESPACE       REVISION        UPDATED STATUS  CHART   APP VERSION
```

现在我们没有任何安装。如果我们重新运行`kubectl get secrets`命令，我们还将看到`mysite`的所有记录都已被清除：

```
$ kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-vjhx2   kubernetes.io/service-account-token   3      65m
```

正如我们从这个输出中看到的，Drupal 图表创建的两个`Secret`不仅被删除了，还删除了四个发布记录。

在下一章中，我们将看到`helm rollback`命令。前面的解释应该让你了解为什么默认情况下无法回滚卸载。不过，可以删除应用程序但保留发布记录：

```
$ helm uninstall --keep-history
```

在 Helm 2 中，默认情况下保留了历史记录。在 Helm 3 中，更改为默认删除历史记录。不同的组织偏好不同的策略，但核心维护者发现，当大多数人卸载时，他们希望销毁所有安装的痕迹。

# 结论

在本章中，我们涵盖了安装和使用 Helm 的基础知识。在探讨了流行的 Helm 安装和配置方法之后，我们添加了一个图表仓库，并学习了如何搜索图表。然后，我们安装了、列出了、升级了，并最终卸载了`bitnami/drupal`图表。

在这个过程中，我们掌握了一些重要的概念。我们学习了安装和发布。我们初步了解了图表仓库，在第七章中将详细介绍。在本章的结尾，我们稍微了解了 Helm 如何存储关于我们安装的信息。

在下一章中，我们将回顾`helm`命令，学习 Helm 工具的其他功能。
