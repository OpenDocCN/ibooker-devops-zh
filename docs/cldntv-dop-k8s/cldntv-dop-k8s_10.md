# 第八章：运行容器

> 如果您有一个棘手的问题无法回答，可以先处理一个简单问题无法回答的问题。
> 
> Max Tegmark

在之前的章节中，我们主要关注了 Kubernetes 的操作方面：如何获取您的集群，如何维护它们以及如何管理您的集群资源。现在让我们转向 Kubernetes 中最基本的对象：*容器*。我们将深入探讨容器在技术层面上的工作原理，它们与 Pods 的关系以及如何将容器镜像部署到 Kubernetes。

本章中，我们还将涵盖容器安全的重要主题，以及如何利用 Kubernetes 中的安全功能按照最佳实践安全地部署应用程序。最后，我们将讨论如何在 Pod 上挂载磁盘卷，使容器能够共享和持久化数据。

# 容器和 Pods

我们已经在第二章介绍了 Pods，并讨论了部署如何使用 ReplicaSets 来维护一组副本 Pods，但我们并没有详细研究过 Pods 本身。Pod 是 Kubernetes 中的调度单位。一个 Pod 对象表示一个容器或一组容器，Kubernetes 中的所有运行都通过 Pod 来实现：

> 一个 Pod 代表在同一执行环境中运行的应用程序容器和卷的集合。Pods 而不是容器是 Kubernetes 集群中最小的可部署工件。这意味着 Pod 中的所有容器始终部署在同一台机器上。
> 
> Kelsey Hightower 等，《Kubernetes 权威指南》

到目前为止，在本书中，“Pod”和“容器”这两个术语基本可以互换使用：演示应用程序 Pod 中只有一个容器。然而，在更复杂的应用程序中，Pod 很可能包含两个或更多个容器。因此，让我们看看这是如何工作的，以及何时以及为什么您可能希望将容器组合在 Pods 中。

## 什么是容器？

在询问为什么要在 Pod 中拥有多个容器之前，让我们花点时间重新审视容器到底是什么。

您从“容器的到来”了解到，容器是一个标准化的软件包，其中包含软件及其依赖项、配置、数据等：即使是运行所需的一切。然而，它究竟是如何工作的呢？

在 Linux 和大多数其他操作系统中，所有在机器上运行的东西都是通过*进程*来实现的。进程代表运行应用程序的二进制代码和内存状态，如 Chrome、`top`或 Visual Studio Code。所有进程存在于同一个全局命名空间：它们可以相互看到和交互，它们共享 CPU、内存和文件系统等资源池。（Linux 命名空间有点像 Kubernetes 命名空间，尽管从技术上讲并非完全相同。）

从操作系统的角度来看，容器表示存在于自己命名空间中的隔离进程（或一组进程）。容器内部的进程看不到外部进程，反之亦然。容器无法访问属于另一个容器的资源，或者容器外部的进程。容器边界就像是一个环形栅栏，阻止进程胡作非为并占用彼此的资源。

就容器内部进程而言，它好像在自己的机器上运行，并且完全访问所有资源，且没有其他进程在运行。如果在容器内运行几个命令，您就可以看到这一点：

```
`kubectl run busybox --image busybox:1.28 --rm -it --restart=Never /bin/sh`
If you don't see a command prompt, try pressing enter. / # `ps ax`
PID   USER     TIME  COMMAND
 1 root      0:00 /bin/sh 8 root      0:00 ps ax 
/ # `hostname`
busybox
```

通常，`ps ax` 命令将列出机器上运行的所有进程，通常有很多进程（在典型的 Linux 服务器上可能有几百个）。但在这里只显示了两个进程：`/bin/sh` 和 `ps ax`。因此，容器内可见的进程只有实际在容器中运行的进程。

类似地，`hostname` 命令通常会显示主机名称，但在这里返回的是 `busybox`：实际上，这是容器的名称。因此，对于 `busybox` 容器来说，它看起来好像在一个名为 `busybox` 的机器上运行，并且它拥有整个机器的控制权。这对于同一台机器上运行的每个容器都是成立的。

###### 小贴士

一个有趣的练习是自己创建一个容器，而不依赖像 Docker 这样的容器运行时。Liz Rice 在[“What is a Container, Really?”](https://oreil.ly/KRMP7)的精彩演讲中展示了如何在 Go 程序中从头开始做到这一点。

## Kubernetes 中的容器运行时

正如我们在第三章中提到的，Docker 并不是运行容器的唯一方式。事实上，2020 年 12 月，维护者们[宣布](https://oreil.ly/D8jVd) Kubernetes 中的 Docker 运行时将被弃用，并用使用[容器运行时接口（CRI）](https://oreil.ly/K5qJw)的替代方案取而代之。这并不意味着将来 Kubernetes 中将不再支持 Docker 容器，也不意味着 Docker 将不再是与 Kubernetes 外的容器交互的有用工具。只要容器符合[开放容器倡议（OCI）](https://oreil.ly/OXs0C)定义的标准，它就应该能在 Kubernetes 中运行，而使用 Docker 构建的容器确实符合这些标准。您可以在 Kubernetes 博客上的[“Dockershim Removal FAQ”](https://oreil.ly/PMTXH)中详细了解此变更及其影响。

## 容器中应该放什么内容？

技术上没有理由限制您在一个容器中运行多少进程：您可以在同一个容器中运行完整的 Linux 发行版，包括多个正在运行的应用程序、网络服务等等。这也是为什么有时容器被称为*轻量级虚拟机*。但这并不是使用容器的最佳方式，因为这样您无法获得资源隔离的好处。

如果进程不需要彼此知道，则它们无需在同一个容器中运行。一个容器的一个良好的经验法则是它应该*只做一件事*。例如，我们的演示应用程序容器监听一个网络端口，并向连接到它的任何人发送字符串`Hello, 世界`。这是一个简单的、自包含的服务：它不依赖于任何其他程序或服务，反过来也没有程序依赖它。这是一个非常适合拥有自己容器的理想候选者。

容器还有一个*入口点*：在容器启动时运行的命令。通常情况下，这会导致创建一个单独的进程来运行命令，尽管某些应用程序经常启动几个子进程作为辅助程序或工作程序。如果要在容器中启动多个独立的进程，您需要编写一个包装脚本作为入口点，然后启动您想要的进程。

###### 提示

每个容器应只运行一个主要进程。如果在一个容器中运行大量不相关的进程，则没有充分利用容器的强大功能，您应考虑将应用程序拆分为多个相互通信的容器。

## 什么应该放在一个 Pod 中？

现在您知道容器是什么，就能明白将它们组合在 Pod 中的好处所在。Pod 代表一组需要相互通信和共享数据的容器；它们需要一起调度，需要一起启动和停止，并且需要运行在同一物理机器上。

典型例子是将数据存储在本地缓存中的应用程序，比如[Memcached](https://memcached.org/about)。您需要运行两个进程：您的应用程序和处理存储和检索数据的`memcached`服务器进程。虽然您可以将这两个进程运行在一个单独的容器内，但这是不必要的——它们只需要通过网络套接字进行通信。最好将它们拆分成两个独立的容器，每个容器只需关心构建和运行自己的进程。

您可以使用公共的 Memcached 容器镜像，[可从 Docker Hub 获取](https://hub.docker.com/_/memcached)，它已经设置为可以作为 Pod 的一部分与另一个容器一起工作。

因此，您创建一个 Pod，其中包含两个容器：Memcached 和您的应用程序。应用程序可以通过网络连接与 Memcached 通信，因为这两个容器位于同一个 Pod 中，所以连接始终是本地的：这两个容器始终运行在同一节点上。

同样地，想象一个 web 应用程序，其中包含一个 web 服务器容器，比如 NGINX，以及一个生成 HTML 网页、文件和图片等的博客应用程序。博客容器将数据写入磁盘，由于 Pod 中的容器可以共享磁盘卷，数据也可以供 NGINX 容器通过 HTTP 服务。你可以在 [Kubernetes 文档网站](https://oreil.ly/Jpueo) 上找到这样一个例子。

> 总的来说，在设计 Pod 时应该问自己的正确问题是，“如果这些容器落在不同的机器上，它们能否正常工作？” 如果答案是“否”，那么 Pod 是容器的正确分组方式。如果答案是“是”，那么多个 Pod 可能是正确的解决方案。
> 
> Kelsey Hightower 等人，《Kubernetes 上手指南》

Pod 中的容器应该共同工作以执行一个任务。如果你只需要一个容器来完成这个任务，那很好：使用一个容器。如果你需要两个或三个，那也可以。如果你有更多，你可能需要考虑这些容器是否可以分成单独的 Pod。

# 容器清单

我们已经概述了容器的内容、容器中应包含的内容，以及何时应将容器组合在 Pod 中。那么，我们如何在 Kubernetes 中实际运行容器呢？

当你创建你的第一个 Deployment 时，在“部署清单”中，它包含一个 `template.spec` 部分，指定要运行的容器（在那个例子中只有一个容器）：

```
spec:
  containers:
  `-` `name``:` `demo`
    `image``:` `cloudnatived/demo:hello`
    `ports``:`
    `-` `containerPort``:` `8888`
```

下面是具有两个容器部署的 `template.spec` 部分的示例：

```
spec:
  containers:
  `-` `name``:` `container1`
    `image``:` `example/container1`
  `-` `name``:` `container2`
    `image``:` `example/container2`
```

每个容器规范中唯一需要的字段是 `name` 和 `image`：一个容器必须有一个名称，这样其他资源可以引用它，并且你必须告诉 Kubernetes 在容器中运行什么镜像。

## 镜像标识符

到目前为止，你在本书中已经使用了一些不同的容器镜像标识符；例如，`cloudnatived/demo:hello`、`alpine` 和 `busybox:1.28`。

实际上，镜像标识符有四个不同的部分：*注册表主机名*、*仓库命名空间*、*镜像仓库* 和 *标签*。除了镜像名称外，其他都是可选的。一个使用所有这些部分的镜像标识符看起来像这样：

`docker.io/cloudnatived/demo:hello`

+   在这个例子中，注册表主机名是 `docker.io`；实际上，这是 Docker 镜像的默认设置，所以我们不需要指定它。然而，如果你的镜像存储在另一个注册表中，你需要给出其主机名。例如，Google Container Registry 的镜像以 `gcr.io` 为前缀。

+   仓库命名空间是 `cloudnatived`：这是我们（你好！）。如果不指定仓库命名空间，则使用默认命名空间（称为 `library`）。这是一组 [官方镜像](https://oreil.ly/nCHJn)，由 Docker, Inc. 批准和维护。流行的官方镜像包括操作系统基础镜像（`alpine`、`ubuntu`、`debian`、`centos`）、语言环境（`golang`、`python`、`ruby`、`php`、`java`）以及广泛使用的软件（`mongo`、`mysql`、`nginx`、`redis`）。

+   镜像仓库是 `demo`，它标识了注册表和命名空间内的特定容器镜像（另见 “Container Digests”）。

+   标签是 `hello`。标签标识了同一镜像的不同版本。

你可以根据需要给容器打上任何标签：一些常见的选择包括：

+   语义化版本标签，比如 `v1.3.0`。这通常指的是应用程序的版本。

+   Git SHA 标签，比如 `5ba6bfd...`。这标识了构建容器所用源代码库中特定的提交（参见 “Git SHA Tags”）。

+   它所代表的环境，如 `staging` 或 `production`。

你可以给一个给定的镜像添加任意多个标签。

## 最新标签

如果在拉取镜像时未指定标签，默认的 Docker 镜像标签是 `latest`。例如，当你运行一个未指定标签的 `alpine` 镜像时，你将得到 `alpine:latest`。

`latest` 标签是在没有指定标签的情况下构建或推送镜像时添加的默认标签。它并不一定标识最近的镜像，只是最近未显式标记的镜像。这使得 `latest` 作为标识符 [相当无用](https://oreil.ly/cVp7N)。

这就是为什么在部署生产容器到 Kubernetes 时，始终使用特定标签非常重要。当你只运行一个临时容器进行故障排除或实验时，例如 `alpine` 容器，可以省略标签并获取最新镜像。但对于真正的应用程序，你需要确保如果明天部署 Pod，你将得到与今天部署时相同的容器镜像：

> 在生产环境中部署容器时应避免使用 `latest` 标签，因为难以追踪镜像的版本，也更难以正确回滚。
> 
> [Kubernetes 文档](https://oreil.ly/ZmTBp)

## 容器摘要

正如我们所见，`latest` 标签并不总是如你所想象的那样，即使是语义化版本或者 Git SHA 标签也不能唯一永久地标识特定的容器镜像。如果维护者决定推送一个具有相同标签的不同镜像，在下次部署时，你将获得那个更新的镜像。技术上来说，一个标签是 *不确定的*。

有时希望具有*确定性*部署：换句话说，保证部署始终引用你指定的确切容器镜像。你可以通过容器的*摘要*来实现这一点：镜像内容的加密哈希，不可变地标识该镜像。

镜像可以有多个标签，但只有一个摘要。这意味着如果你的容器清单指定了镜像摘要，你可以保证确定性部署。带有摘要的镜像标识看起来像这样：

`cloudnatived/demo@sha256:aeae1e551a6cbd60bcfd56c3b4ffec732c45b8012b7cb758c6c4a34...`

## 基础镜像标签

在 Dockerfile 中引用基础镜像时，如果未指定标签，将会获取`latest`，就像在运行容器时一样。如果有一天你的构建停止工作，并发现你正在使用的`latest`镜像现在指向了引入某些破坏性变更的不同版本的镜像，这可能会让人困惑。

出于这个原因，你可能希望在 Dockerfile 中的`FROM`行中使用更具体的基础镜像标签。但你应该使用哪个标签？或者应该使用确切的摘要吗？这在很大程度上取决于你的开发情况和偏好。

让我们以[Docker Hub 上的官方 Python 镜像](https://oreil.ly/VOII3)为例。你可以选择使用`python:3`、`python:3.9`、`python:3.9.7`或许多其他版本标签的变体，还可以选择不同的基础操作系统，如 Windows、Alpine 和 Debian。

使用较为通用的标签，比如`python:3`，的优势在于，每次构建新镜像时，你会自动获取任何更新和安全补丁，以及 Python 3 的最新小版本。缺点是，有时这些更新可能会导致问题，比如系统包重命名或移除。你的应用在 Python 3.9 上可能运行良好，但如果构建新镜像后没有意识到基础镜像`python:3`实际上从 3.9 版本移动到了 3.10 版本，可能会因为新的 3.10 版本引入的变更而开始失败。

如果你使用更具体的标签，比如`python:3.9.7`，你的基础镜像不太可能意外更改。然而，你需要注意并手动更新你的 Dockerfile，以便在有重要的安全和错误修复时拉取这些更新。你可能更喜欢这种开发风格，因为你可以更好地控制你的构建过程，但定期检查基础镜像的更新是很重要的，以免滞后，因为它们将缺少维护者推送的安全修复。

你使用的镜像标签很大程度上取决于你的团队偏好、发布节奏和开发风格。你应该权衡你选择的标签系统的利弊，并定期检查，以确保你有一个可以提供合理可靠的构建并按可持续的速度进行常规更新的流程。

## 端口

在我们的演示应用程序中，您已经看到了`ports`字段在“服务资源”中的使用。它指定应用程序将监听的网络端口号，并且可以与[服务](https://oreil.ly/N3EvV)匹配，用于将请求路由到容器。

## 资源请求和限制

我们已经在第五章详细介绍了容器的资源请求和限制，所以在这里简要回顾一下就够了。

每个容器可以作为其规范的一部分提供以下一个或多个：

+   `resources.requests.cpu`

+   `resources.requests.memory`

+   `resources.limits.cpu`

+   `resources.limits.memory`

尽管在单个容器上指定了请求和限制，但我们通常根据 Pod 的总资源请求和限制来讨论。Pod 的资源请求是该 Pod 中所有容器的资源请求之和，等等。

## 镜像拉取策略

在容器可以在节点上运行之前，必须从适当的容器注册表中*拉取*或下载镜像。容器上的`imagePullPolicy`字段控制 Kubernetes 完成此操作的频率。它可以取三个值之一：`Always`、`IfNotPresent`或`Never`：

+   `Always`将在每次容器启动时拉取镜像。假设您指定了一个标签 —— 这是您应该做的（参见“最新标签”）——那么这可能是不必要的，并且可能会浪费时间和带宽。

+   `IfNotPresent`是默认值，适合大多数情况。如果镜像尚未在节点上存在，将下载它。之后，除非更改镜像规范，否则每次容器启动时都会使用保存的镜像，并且 Kubernetes 不会尝试重新下载它。

+   `Never`根本不会更新镜像。使用此策略时，Kubernetes 永远不会从注册表中获取镜像：如果节点上已经存在，将使用它，但如果不存在，则容器将无法启动。您不太可能希望这样做。

如果遇到奇怪的问题（例如，推送新容器镜像后 Pod 没有更新），请检查您的镜像拉取策略。

## 环境变量

环境变量是在运行时向容器传递信息的常见但有限的方式。常见是因为所有 Linux 可执行文件都可以访问环境变量，即使是在容器出现之前编写的程序也可以使用它们的环境进行配置。有限是因为环境变量只能是字符串值：没有数组，没有键值对，总之没有结构化数据。进程环境的总大小也被限制为 32 KiB，因此不能在环境中传递大数据文件。

要设置环境变量，请在容器的`env`字段中列出它：

```
containers:
- name: demo
  image: cloudnatived/demo:hello
  env:
  - name: GREETING
    value: "Hello from the environment"
```

如果容器镜像本身指定了环境变量（例如在 Dockerfile 中设置），那么 Kubernetes 的`env`设置将覆盖它们。这对于修改容器的默认配置非常有用。

###### 小贴士

向容器传递配置数据的更灵活的方法是使用 Kubernetes 的 ConfigMap 或 Secret 对象：有关详细信息，请参阅第十章。

# 容器安全性

您可能已经注意到在 “什么是容器？” 中，当我们用 `ps ax` 命令查看容器中的进程列表时，所有进程都以 root 用户身份运行。在 Linux 和其他类 Unix 操作系统中，`root` 是超级用户，拥有读取任何数据、修改任何文件以及执行系统上任何操作的权限。

在完整的 Linux 系统上，一些进程需要以 `root` 身份运行（例如管理所有其他进程的 `init` 进程），但在容器中通常情况并非如此。

确实，当不需要时以 root 用户身份运行进程是一个坏主意。这违反了 [*最小权限原则*](https://oreil.ly/Q5h79)。这一原则指出，程序只能访问它实际需要执行其工作的信息和资源。

程序会有 bug ——这是任何写过程序的人都明白的事实。一些 bug 允许恶意用户劫持程序去执行不应该做的事情，比如读取秘密数据或执行任意代码。为了减少这种风险，以最小可能的权限运行容器非常重要。

这始于不允许它们以 `root` 身份运行，而是分配给它们一个普通用户：一个没有特殊特权的用户，例如读取其他用户文件的特权。

> 就像你不应该在服务器上以 root 运行任何东西一样，在服务器上的容器中也不应该以 root 运行任何东西。运行在其他地方创建的二进制文件需要极大的信任，同样适用于容器中的二进制文件。
> 
> [马克·坎贝尔](https://oreil.ly/RlmNm)

攻击者也可以利用容器运行时的 bug “逃离” 容器，并在主机上获取与容器中相同的权限。

## 以非 root 用户身份运行容器

下面是一个容器规范的示例，告诉 Kubernetes 以特定用户身份运行容器：

```
containers:
- name: demo
  image: cloudnatived/demo:hello
  securityContext:
    `runAsUser``:` `1000`
```

`runAsUser` 的值是 *UID*（数字用户标识符）。在许多 Linux 系统上，UID 1000 被分配给系统上创建的第一个非 root 用户，因此通常选择 1000 或更高的值作为容器 UID 是安全的。无论容器中是否存在具有该 UID 的 Unix 用户，甚至容器中是否存在操作系统，这种方式都同样有效，即使是在空白容器中也是如此。

如果指定了 `runAsUser` UID，它会覆盖容器镜像中配置的任何用户。如果没有指定 `runAsUser`，但容器指定了一个用户，Kubernetes 将以该用户身份运行。如果在清单或镜像中都没有指定用户，容器将以 `root` 用户运行（正如我们所见，这是一个坏主意）。

为了最大的安全性，您应该为每个容器选择不同的 UID。这样，如果某个容器被某种方式妥协，或者意外覆盖数据，它只有访问自己数据的权限，而不是其他容器的权限。

另一方面，如果你希望两个或更多的容器能够通过挂载卷等方式访问相同的数据，你应该为它们分配相同的 UID。

## 阻止 Root 容器

为了防止这种情况发生，Kubernetes 允许您阻止容器以 root 用户身份运行。

使用`runAsNonRoot: true`设置将实现这一点：

```
containers:
- name: demo
  image: cloudnatived/demo:hello
  securityContext:
    `runAsNonRoot``:` `true`
```

当 Kubernetes 运行该容器时，它将检查容器是否希望以 root 用户身份运行。如果是这样，它将拒绝启动。这将保护您免受忘记在容器中设置非 root 用户或运行配置为以 root 用户身份运行的第三方容器的影响。

如果发生这种情况，您将看到 Pod 状态显示为`CreateContainerConfigError`，当您使用`kubectl describe`查看 Pod 时，您将看到此错误：

```
Error: container has runAsNonRoot and image will run as root
```

# 最佳实践

使用`runAsNonRoot: true`设置以非 root 用户身份运行容器，并阻止 root 容器的运行。

## 设置只读文件系统

另一个有用的安全上下文设置是`readOnlyRootFilesystem`，它将阻止容器向自己的文件系统写入。例如，可以想象一个容器利用 Docker 或 Kubernetes 中的错误，将其文件系统的写入影响到主机节点上的文件。如果其文件系统是只读的，这种情况就不会发生；容器将收到 I/O 错误：

```
containers:
- name: demo
  image: cloudnatived/demo:hello
  securityContext:
    `readOnlyRootFilesystem``:` `true`
```

许多容器不需要向自己的文件系统写入任何内容，因此这个设置不会对它们产生干扰。始终设置`readOnlyRootFilesystem`是[良好的实践](https://oreil.ly/JGiKP)，除非容器确实需要写入文件。

## 禁用特权升级

通常，Linux 二进制文件以执行它们的用户拥有的权限运行。但是也有例外情况：使用`setuid`机制的二进制文件可以临时获取拥有该二进制文件的用户（通常为`root`）的权限。

这在容器中是一个潜在的问题，因为即使容器以常规用户（例如 UID 1000）身份运行，如果它包含一个`setuid`二进制文件，该二进制文件默认可以通过设置获得 root 权限。

要防止这种情况发生，请将容器安全策略中的`allowPrivilegeEscalation`字段设置为`false`：

```
containers:
- name: demo
  image: cloudnatived/demo:hello
  securityContext:
    `allowPrivilegeEscalation``:` `false`
```

现代 Linux 程序不需要`setuid`；它们可以使用称为 *capabilities* 的更灵活和细粒度的权限机制来实现相同的功能。

## 能力

传统上，Unix 程序具有两个特权级别：*normal* 和 *superuser*。普通程序的权限不超过运行它们的用户，而超级用户程序可以执行任何操作，绕过所有内核安全检查。

Linux 的能力机制通过定义程序可以执行的各种具体操作来改进此问题：加载内核模块、执行直接网络 I/O 操作、访问系统设备等等。需要特定权限的程序可以被授予，但不能超出其它权限。

例如，监听端口 80 的 Web 服务器通常需要以 `root` 用户身份运行才能实现此功能；低于 1024 的端口号被视为特权 *系统* 端口。相反，程序可以被授予 `NET_BIND_SERVICE` 能力，允许其绑定到任何端口，但不给予其它特殊权限。

Docker 容器的默认能力集相当宽松。这是一个权衡安全性与可用性的实用决策：默认情况下不给容器设置任何能力会要求操作者在许多容器上设置能力才能运行。

另一方面，最小权限原则表明容器不应具有其不需要的任何能力。Kubernetes 安全上下文允许您从默认设置中删除任何能力，并根据需要添加能力，就像这个例子展示的那样：

```
containers:
- name: demo
  image: cloudnatived/demo:hello
  securityContext:
    `capabilities``:`
      `drop``:` `[``"``CHOWN``"``,` `"``NET_RAW``"``,` `"``SETPCAP``"``]`
      `add``:` `[``"``NET_ADMIN``"``]`
```

容器将删除 `CHOWN`、`NET_RAW` 和 `SETPCAP` 能力，并添加 `NET_ADMIN` 能力。

Docker [文档](https://oreil.ly/TOz3a)列出了默认情况下容器设置的所有能力，并可以根据需要添加。

为了最大化安全性，您应该为每个容器放弃所有能力，并仅在需要时添加特定的能力：

```
containers:
- name: demo
  image: cloudnatived/demo:hello
  securityContext:
    `capabilities``:`
      `drop``:` `[``"``all``"``]`
      `add``:` `[``"``NET_BIND_SERVICE``"``]`
```

容量机制在容器内部的进程能够做的事情上设置了硬限制，即使它们以 root 用户身份运行。一旦在容器级别放弃了某项能力，即使是以最高权限运行的恶意进程也无法重新获得该能力。

## Pod 安全上下文

我们已经在单个容器级别上覆盖了安全上下文设置，但您也可以在 Pod 级别上设置其中的一些：

```
apiVersion: v1
kind: Pod
...
spec:
  `securityContext``:`
    `runAsUser``:` `1000`
    `runAsNonRoot``:` `false`
    `allowPrivilegeEscalation``:` `false`
```

这些设置将适用于 Pod 中的所有容器，除非容器在其自己的安全上下文中覆盖了特定设置。

## Pod 服务账户

Pod 使用命名空间的默认服务账户权限运行，除非您另有指定（参见“应用程序和部署”）。如果出于某些原因需要授予额外的权限（例如查看其他命名空间中的 Pod），请为应用程序创建专用服务账户，将其绑定到所需的角色，并配置 Pod 使用新的服务账户。

要实现这一点，请在 Pod 规范的 `serviceAccountName` 字段中设置服务账户的名称：

```
apiVersion: v1
kind: Pod
...
spec:
  `serviceAccountName``:` `deploy-tool`
```

# 卷

正如您可能记得的那样，每个容器都有自己的文件系统，只能由该容器访问，并且是 *临时的*：当容器重新启动时，不属于容器镜像的任何数据都将丢失。

通常情况下，这是可以的；例如，演示应用程序是一个无状态服务器，因此不需要持久存储。它也不需要与任何其他容器共享文件。

然而，更复杂的应用可能需要既能与同一 Pod 中的其他容器共享数据，又能在重启后保持数据持久性。Kubernetes Volume 对象可以提供这两者。

您可以将许多不同类型的 Volume 附加到 Pod 上。无论基础存储介质如何，挂载到 Pod 上的 Volume 对所有 Pod 中的容器都是可访问的。需要通过共享文件进行通信的容器可以使用各种类型的 Volume。我们将在以下部分讨论一些更重要的类型。

## emptyDir Volumes

最简单的 Volume 类型是 `emptyDir`。这是一种临时存储，初始为空——因此得名——并将其数据存储在节点上（可以是内存中或节点的磁盘上）。它只在 Pod 在该节点上运行时存在。

当您想要为容器提供一些额外的存储空间，但数据永久保存或在容器移动到另一个节点时移动并不关键时，`emptyDir` 是有用的。一些示例包括缓存下载文件或生成内容，或者用于数据处理作业的临时工作空间。

同样地，如果您只想在 Pod 中的容器之间共享文件，但不需要长时间保留数据，那么 `emptyDir` Volume 是理想的选择。

这是一个创建 `emptyDir` Volume 并将其挂载到容器上的 Pod 示例：

```
apiVersion: v1
kind: Pod
...
spec:
  volumes:
  - name: cache-volume
    emptyDir: {}
  containers:
  - name: demo
    image: cloudnatived/demo:hello
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
```

首先，在 Pod 规范的 `volumes` 部分，我们创建一个名为 `cache-volume` 的 `emptyDir` Volume：

```
volumes:
- `name``:` `cache-volume`
  `emptyDir``:` `{``}`
```

现在 `cache-volume` Volume 可供 Pod 中的任何容器挂载和使用。为此，我们在 `demo` 容器的 `volumeMounts` 部分列出它：

```
name: demo
image: cloudnatived/demo:hello
volumeMounts:
- `mountPath``:` `/cache`
  `name``:` `cache-volume`
```

容器不必采取任何特殊操作即可使用新存储：任何写入路径 `/cache` 的内容将写入 Volume，并对挂载同一 Volume 的其他容器可见。所有挂载该 Volume 的容器都可以对其读写。

###### 提示

在写入共享 Volume 时要小心。Kubernetes 不强制执行磁盘写入的任何锁定。如果两个容器同时尝试写入同一个文件，可能会导致数据损坏。为了避免这种情况，可以实现自己的写锁定机制，或使用支持锁定的 Volume 类型，如 `nfs` 或 `glusterfs`。

## 持久化 Volumes

尽管临时的 `emptyDir` Volume 适合缓存和临时文件共享，但某些应用需要存储持久数据；例如，任何类型的数据库。总的来说，我们不建议您试图在 Kubernetes 中运行数据库。通常情况下，最好使用云服务：例如，大多数云提供商都有管理解决方案，如 MySQL 和 PostgreSQL 的关系型数据库，以及键值（NoSQL）存储。

正如我们在“Kubernetes 不是万能药”中看到的，Kubernetes 最擅长管理无状态应用，这意味着没有持久化数据。存储持久化数据会显著复杂化应用程序的 Kubernetes 配置。您需要确保您的持久化存储可靠、高性能、安全且备份。

如果您需要在 Kubernetes 中使用持久卷，那么持久卷资源（PersistentVolume）正是您要找的。我们不会在这里详细介绍它，因为具体细节通常特定于您的云提供商；您可以在 Kubernetes 的[文档](https://oreil.ly/FQvRN)中了解更多关于持久卷的信息。

在 Kubernetes 中最灵活使用持久卷的方式是创建一个 PersistentVolumeClaim 对象。这代表了对特定类型和大小持久卷的请求；例如，一个 10 GiB 大小的高速读写存储卷。

然后，Pod 可以将此持久卷索取作为一个卷，容器可以挂载和使用它：

```
volumes:
- name: data-volume
  persistentVolumeClaim:
    claimName: data-pvc
```

您可以在集群中创建一组持久卷（PersistentVolumes），供 Pod 按此方式索取。或者，您可以设置[*动态配置*](https://oreil.ly/4VNTz)：当像这样的持久卷索取时，将自动配置并连接适当大小的存储到 Pod。

我们将在“有状态副本集”中更详细地讨论这个问题。

# 重启策略（Restart Policies）

正如我们在“运行容器进行故障排除”中看到的，当 Pod 退出时，Kubernetes 会始终重新启动它。因此，默认的重启策略是`Always`，但您可以将其更改为`OnFailure`（仅在容器以非零状态退出时重启）或`Never`：

```
apiVersion: v1
kind: Pod
...
spec:
  `restartPolicy``:` `OnFailure`
```

如果您希望运行一个 Pod 直到完成并退出，而不是重新启动，您可以使用 Job 资源来实现这一点（参见“作业”）。

# 镜像拉取密钥（Image Pull Secrets）

如果容器注册表中没有指定的镜像，Kubernetes 将从该注册表下载。但是，如果您使用的是私有注册表呢？您如何提供给 Kubernetes 认证到注册表的凭据？

Pod 上的`imagePullSecrets`字段允许您进行配置。首先，您需要将注册表凭据存储在一个 Secret 对象中（有关详细信息，请参阅“Kubernetes Secrets”）。现在，您可以告诉 Kubernetes 在拉取 Pod 中的任何容器时使用此 Secret。例如，如果您的 Secret 命名为`registry-creds`：

```
apiVersion: v1
kind: Pod
...
spec:
  `imagePullSecrets``:`
  `-` `name``:` `registry-creds`
```

注册表凭据数据的确切格式在 Kubernetes 的[文档](https://oreil.ly/AfdOF)中有描述。

您还可以将`imagePullSecrets`附加到服务账户上（参见“Pod 服务账户”）。使用此服务账户创建的任何 Pod 都将自动具有附加的注册表凭据。

# 初始容器（Init Containers）

如果你发现自己需要在运行主应用程序之前运行容器的情况下，可以使用一个[初始化容器](https://oreil.ly/Obo5B)。

初始化容器在 Pod 规范中定义，工作方式基本与常规容器相同，但它们不使用存活探针或就绪探针。相反，初始化容器必须在 Pod 中的其他容器启动之前成功运行并退出：

```
apiVersion: v1
kind: Pod
...
spec:
  containers:
  - name: demo
    image: cloudnatived/demo:hello
  initContainers:
  - name: init-demo
    image: busybox
    command: [*`sh`*, *`-c`*, *`echo` `Hello` `from` `initContainer`*]
```

这些可以在启动应用程序之前进行预检查，或者运行任何必要的引导脚本来准备你的应用程序。一个常见的初始化容器用例是从外部 Secret 存储中获取 Secrets，并在启动前将它们挂载到你的应用程序中。只需确保你的初始化容器是幂等的，并且如果决定使用它们，则可以安全地重试。

# 总结

要理解 Kubernetes，首先需要理解容器。在本章中，我们概述了容器的基本概念，它们如何在 Pod 中协同工作，以及可用于控制容器在 Kubernetes 中运行方式的选项。

基本要求：

+   Linux 容器在内核级别是一组隔离的进程集合，具有环形资源。从容器内部看，它看起来就像容器拥有一个专属的 Linux 机器。

+   容器不是虚拟机。每个容器应该运行一个主要进程。

+   一个 Pod 通常包含一个运行主要应用程序的容器，以及可选的*辅助*容器来支持它。

+   容器镜像规格可以包括注册表主机名、仓库命名空间、镜像仓库和标签；例如，`docker.io/cloudnatived/demo:hello`。只有镜像名称是必需的。

+   为了可重现的部署，始终为容器镜像指定一个标签。否则，你将得到任意的`latest`版本。

+   容器中的程序不应该以 root 用户身份运行。相反，分配给它们一个普通用户。

+   你可以在容器上设置`runAsNonRoot: true`字段，以阻止任何想要作为`root`运行的容器。

+   容器的其他有用的安全设置包括`readOnlyRootFilesystem: true`和`allowPrivilegeEscalation: false`。

+   Linux 权限提供了一种细粒度的权限控制机制，但是容器的默认权限可能过于宽松。你可以通过取消所有容器的权限，然后根据需要授予特定的权限来锁定你的 Pods。

+   相同 Pod 中的容器可以通过读取和写入挂载的 Volume 来共享数据。最简单的 Volume 类型是`emptyDir`，它在开始时是空的，并且只在 Pod 运行时保留其内容。

+   另一方面，持久卷会根据需要保留其内容。Pods 可以使用持久卷声明动态配置新的持久卷。

+   初始化容器在 Pod 中启动应用程序之前进行初始设置非常有用。
