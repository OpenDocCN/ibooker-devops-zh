# 第十章：使用 Linkerd 观察您的平台

处理微服务应用程序的一个挑战是监控它们。即使在单一语言中，处理多个开发团队，理解哪些工作负载正在通信并从这些通信中提取有用的指标可能是一个巨大的挑战。每个开发人员、语言和框架都会优先考虑不同的细节，组织需要一种统一的方式来查看所有这些不同的服务。

*可观察性*指的是通过外部观察来理解系统的能力。一个应用程序可以更或少可观察，因此当我们谈论 Linkerd 中的可观察性时，我们指的是它如何影响您应用程序的可观察性。在本章中，我们将看看 Linkerd 通过为所有应用程序提供标准指标来增加可观察性，让您看到微服务之间的关系，并允许拦截和分析应用程序间的通信。

# 为什么我们需要这个？

与应用安全性类似，微服务对平台工程师提出了新的挑战。动态扩展组件的能力，按需创建和更新服务，以及动态配置基础设施，增加了理解应用程序健康状况的难度。当您的组织为应用程序开发人员构建平台时，重要的是让团队能够轻松做出正确的决策。

# Linkerd 如何帮助？

Linkerd 有助于将可观察性融入您的平台。当您将工作负载添加到网格中时，它会自动显示有关该工作负载行为的重要信息。这意味着当我们将 Linkerd 添加到我们的平台时，我们可以使所有应用团队在可观察性方面做出正确的决策。如果允许您的应用程序加入网格，您可以自动显示有关您的应用程序性能、健康状况和关系数据的标准信息。如果进一步构建服务配置文件，您可以保存和共享有关应用程序内个别路由的关键信息。

当我们通过`linkerd` CLI 来探索如何使用 Linkerd 观察您的应用程序时，我们将覆盖的所有内容也可以通过 Linkerd Viz 仪表板显示。我们将在本章末尾介绍仪表板。

正如我们在第一章中提到的，有三个黄金指标已被反复证明对理解微服务应用的运行情况至关重要：流量、成功率和延迟（参见图 1-8）。

在微服务应用中，每个工作负载都能够提供这些指标非常关键：仅凭这些黄金指标，您应该能够了解特定工作负载的表现如何，以及系统中哪些领域需要特别关注或优化。

Linkerd 代理会自动从每个工作负载和请求中收集详细的指标，并通过 Prometheus 提供这些信息，以便您可以使用各种广泛可用的工具在您的组织内部显示这些信息。

# Linkerd 中的可观察性

我们将使用 booksapp 和 emojivoto 应用程序演示 Linkerd 中的可观察性。这两个应用程序故意包含各种故障：我们将使用 Linkerd 的可观察性工具精确定位故障的具体位置。（解决这些问题将留给读者作为练习！）

## 设置您的集群

您需要一个已安装了 Linkerd 和 Linkerd Viz 的集群（如果需要有关设置此类集群的复习，请参阅第三章）。我们将从克隆[booksapp 示例应用](https://oreil.ly/LJou0)和[emojivoto 示例应用](https://oreil.ly/0n5Gd)仓库开始，如示例 10-1 所示，因为我们需要这些仓库来正确配置这些示例应用。

##### 示例 10-1\. 克隆仓库

```
# Clone the booksapp repo
$ git clone https://github.com/BuoyantIO/booksapp.git

# Clone the emojivoto repo
$ git clone https://github.com/BuoyantIO/emojivoto.git
```

接下来，我们可以在集群中启动和运行这些应用程序，如示例 10-2 所示。

##### 示例 10-2\. 设置我们的应用程序

```
# Install booksapp
$ kubectl create ns booksapp && \
  curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/booksapp.yml \
  | linkerd inject - | kubectl -n booksapp apply -f -

# Install emojivoto
$ curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/emojivoto.yml \
  | linkerd inject - | kubectl apply -f -

# Check that booksapp is ready
$ linkerd check --proxy --namespace booksapp

# Check that emojivoto is ready
$ linkerd check --proxy --namespace emojivoto
```

一旦我们的检查结果返回正常，我们可以开始使用 `linkerd viz` 命令查看我们的应用程序，如示例 10-3 所示。请注意，Linkerd Viz 开始显示数据可能需要一分钟左右的时间，因为它需要收集足够的数据以生成统计信息。

##### 示例 10-3\. 收集应用程序指标

```
# View namespace metrics
$ linkerd viz stat ns

# View deployment metrics
$ linkerd viz stat deploy -n emojivoto
$ linkerd viz stat deploy -n booksapp

# View Pod metrics
$ linkerd viz stat pod -n emojivoto
$ linkerd viz stat pod -n booksapp
```

仅从这些基本查询中，您立即可以看到 emojivoto 和 booksapp 应用程序都存在可靠性问题。在接下来的章节中，我们将深入探讨我们的应用程序，以确定问题的根源。

## Tap

Linkerd Viz Tap 允许授权用户收集应用程序之间流动的请求的元数据。它会显示有关请求头、URI、响应代码等的详细信息，以便您在调试时按需访问这些数据，如示例 10-4 所示。Tap 还提供了方便的工具，用于验证您的应用间连接的 TLS 状态。

##### 示例 10-4\. 查看 Tap 数据

```
# Tap the emojivoto web frontend
$ linkerd viz tap deploy/web -n emojivoto
```

`linkerd viz tap` 命令会持续运行，直到收到终止信号。它会显示代理传输的实时数据，这些数据将详细描述去往和来自 Web 部署的各个请求。每一行将显示源和目标详情、TLS 状态、任何状态信息以及其他可用的元数据。

# 安装 Tap

Tap 已集成到 Linkerd Viz 扩展中，因此它将在执行 `linkerd viz install` 命令时自动安装。但是，如果您安装 Viz 前已运行任何工作负载，您需要在 Tap 可用之前重新启动这些工作负载。

Tap 数据是一个强大的诊断工具，可以提供关于您的应用程序如何彼此通信的详细见解。如果在查看 Linkerd Viz 仪表板中的工作负载时启用了 Tap，它将自动显示请求的摘要。确保在本章后期查看 Viz 仪表板时尝试查看 emojivoto 工作负载的 Tap 数据。

## 服务配置文件

Linkerd *服务配置文件*，由 ServiceProfile 资源体现，允许您为网格提供有关如何使用给定工作负载的详细信息。在其最基本的层面上，ServiceProfile 定义了工作负载允许哪些路由。一旦定义了路由，您可以配置每个路由的指标、超时和重试，以及哪些 HTTP 状态将被视为失败。

# ServiceProfile 和 HTTPRoute

Linkerd 项目正在过渡到完全采用[网关 API](https://oreil.ly/Onjs9)。随着项目朝着这个目标迈进，你会看到一些 Linkerd 自定义资源，包括 ServiceProfile，开始被弃用。

在 Linkerd 2.13 和 2.14 中，ServiceProfile 和 HTTPRoute 常常具有互斥的功能，这使得在开始在集群中使用这些资源之前，仔细审查 [ServiceProfile 文档](https://oreil.ly/zJk_j) 尤为重要。

您可以通过多种方式构建 ServiceProfiles。最灵活的方法是手工编写它们，但是 Linkerd CLI 提供了几种不同的自动生成方法，正如您将在以下部分看到的那样。

### 配置 emojivoto 的路由

emojivoto 应用程序有三个工作负载：

+   `emoji` 和 `voting` 工作负载使用 gRPC 进行通信，其 gRPC 消息在 protobuf 文件中定义。

+   `web` 工作负载使用 HTTP 与 web 浏览器交互。

我们将从 `emoji` 和 `voting` 开始，因为它们有 *protobuf* 文件。Protobuf 文件作为我们 API 的指南，并且可以被 Linkerd CLI 消耗，以自动创建 ServiceProfiles，如 示例 10-5 所示。

##### 示例 10-5\. 从 protobuf 文件创建 ServiceProfiles

```
# Begin by checking for any existing routes.
$ linkerd viz routes -n emojivoto deploy

# The output will show every workload in the emojivoto
# namespace with a default route. We will now work to
# create application-specific routes for emoji and
# voting.

# Create a ServiceProfile object.
$ linkerd profile --proto emojivoto/proto/Emoji.proto emoji-svc -n emojivoto

# This creates, but doesn't apply, the ServiceProfile
# for the emoji service. Take a minute to review the
# profile object so you understand the basic structure.
# We'll be using these ServiceProfiles again in the
# next chapter.

# Create and apply ServiceProfiles for emoji and voting.
$ linkerd profile --proto emojivoto/proto/Emoji.proto emoji-svc -n emojivoto |
  kubectl apply -f -

$ linkerd profile --proto emojivoto/proto/Voting.proto voting-svc -n emojivoto |
  kubectl apply -f -

# Now you can view the updated route data in your environment to see
# your deployed applications. You may need to wait a minute
# for data to populate.
$ linkerd viz routes deploy/emoji -n emojivoto
$ linkerd viz routes deploy/voting -n emojivoto

# Each app will show and store details about which routes have
# been accessed.
```

# 存储 Linkerd Viz 指标

一旦为您的应用程序创建了 ServiceProfiles，Linkerd 的 Viz 扩展将在 Prometheus 中存储这些数据。将 Linkerd 投入生产的一个非常重要的部分是计划如何长期管理这些 Prometheus 数据。Linkerd Viz 随附的 Prometheus 组件*不*足以进行长期数据收集：它将数据存储在内存中，并且每次重新启动时都会*丢失*数据。

通过为 `emoji` 和 `voting` 创建路由，我们已经覆盖了应用程序的三分之二。剩下的是 `web` 组件。虽然我们知道它必须使用 HTTP，因为我们通过浏览器与它通信，但不幸的是，这个组件的作者实际上并没有编写任何关于 API 结构的文档。这让我们不得不试图在没有关于 API 结构的任何信息的情况下构建 ServiceProfile。

幸运的是，我们可以使用 Linkerd 的 Tap 功能来实现这一点，就像 示例 10-6 中展示的那样。

##### 示例 10-6\. 使用 Tap 创建 ServiceProfiles

```
# Create a new ServiceProfile with Tap.
$ linkerd viz profile -n emojivoto web-svc --tap deploy/web --tap-duration 10s |
  kubectl apply -f -

# After you run that command, you should expect to see a
# 10-second pause as Linkerd watches live traffic to the
# service in question and builds a profile.

# View the new profile.
$ kubectl get serviceprofile -n emojivoto web-svc.emojivoto.svc.cluster.local -o yaml

# You will see the object created with two routes, list and vote.

# View the updated route data for web. You may need to allow a minute
# for data to populate.
$ linkerd viz routes deploy/web -n emojivoto
```

# Linkerd 默认路由

Linkerd 的 ServiceProfile 对象旨在定义整个 API，但当我们犯错误或者 API 在没有更新 ServiceProfile 的情况下发生变化时会发生什么？这就是默认路由的作用：任何未在 ServiceProfile 中明确定义的路由都将视为默认路由。

默认路由受到关于重试和超时的默认策略的约束。有关默认路由的流量数据被聚合到总括的 `[DEFAULT]` 路由条目中。

### 为 booksapp 构建路由

现在我们已经完成了 emojivoto 应用程序，需要为 booksapp 应用程序进行设置。

而 emojivoto 包含一些 API 的 protobuf 文件，booksapp 则使用 OpenAPI 定义。与 protobuf 文件类似，OpenAPI 定义（通常称为“Swagger 定义”，源自较早版本的标准）用于定义如何使用 API，并且 Linkerd 能够读取这些定义以创建 ServiceProfiles。

使用 OpenAPI 或 Swagger 定义创建 ServiceProfile 几乎与使用 protobuf 文件创建 ServiceProfile 完全相同，如 示例 10-7 所示。请务必跟着进行操作，因为我们将在 第十一章 再次使用这些 ServiceProfiles！

##### 示例 10-7\. 使用 OpenAPI 定义创建 ServiceProfiles

```
# Create routes for booksapp.
$ linkerd profile --open-api booksapp/swagger/authors.swagger authors -n booksapp |
  kubectl apply -f -

$ linkerd profile --open-api booksapp/swagger/webapp.swagger webapp -n booksapp |
  kubectl apply -f -

$ linkerd profile --open-api booksapp/swagger/books.swagger books -n booksapp |
  kubectl apply -f -

# With that, we've profiled our applications. We can now wait a minute
# and view the relevant route information.

# View route data for booksapp.
$ linkerd viz routes deploy -n booksapp

# You should see a number of routes with varying success rates.
# In Chapter 11 we'll use some of Linkerd's reliability
# features to help address the issues booksapp is having.
```

## 拓扑结构

通过路由、指标和 Tap 数据，我们有很多有用的方法来理解我们的应用程序在做什么，而无需开发人员在其应用程序中包含仪表。另一个常见的挑战是弄清楚这些可能的调用中实际发生了哪些，以及从哪个工作负载到哪个工作负载。Linkerd 也可以为您展示这些信息。

在 示例 10-8 中，我们将检查 booksapp 应用程序组件之间的关系。您可以自行尝试探索 emojivoto 应用程序。

##### 示例 10-8\. 在 Linkerd 中查看边缘

```
# Start by getting the deployments in the booksapp namespace.
$ kubectl get deploy -n booksapp

# You'll see four deployments: traffic, webapp, authors, and books.

# Now, dig into the relationship between these components with
# the linkerd viz edges command.
$ linkerd viz edges deploy -n booksapp
```

输出被分为五列：

`SRC`

流量的源头

`DST`

流量的目的地

`SRC_NS`

流量起源的命名空间

`DST_NS`

流量所去的命名空间

`SECURED`

流量是否通过 Linkerd 的 mTLS 进行加密

结果输出将为您展示书籍应用组件之间的关系概述。它显示 `linkerd-viz` 命名空间中的 Prometheus 实例与 `booksapp` 命名空间中的每个部署之间的通信。此外，我们可以看到 `traffic` 与 `webapp` 之间的通信，`webapp` 与 `books` 以及 `authors` 之间的通信，以及 `books` 和 `authors` 之间的互通。

`linkerd viz edges` 命令可用于 Kubernetes 中的 Pod 或任何其他工作负载类型。

# Linkerd Viz

您可能已经注意到，本章中使用的许多命令都是 `linkerd viz` 命令。这是我们在第二章介绍的 Linkerd Viz 扩展。它与核心 Linkerd 系统一起发布，因为它通常非常有用，但是从 Linkerd 2.10 版本开始，它被分离为扩展，因此不是每个人都被强制运行它。

Viz 扩展提供了许多 CLI 工具，用于观察您的 Linkerd 安装，以及一个基于 Web 的仪表盘，提供图形界面，用于探索您的 Linkerd 环境。

# 保护 Viz 仪表盘取决于您自己

如第二章所述，Linkerd Viz 仪表盘中没有内置用户认证功能。如果您希望将 Linkerd Viz 暴露给网络，您需要使用 API 网关或类似的工具来处理，或者简单地通过端口转发，在集群外部访问时将仪表盘设为不可用，并使用 `linkerd viz dashboard` CLI 命令在 Web 浏览器中打开仪表盘。

使用以下命令打开 Viz 仪表盘：

```
$ linkerd viz dashboard
```

现在您可以自己探索。尝试查找按命名空间和工作负载的度量标准。还可以查看单个命名空间，例如 `emojivoto`，并探索其拓扑结构。

Linkerd 的 Viz 仪表盘包含 Prometheus，并可以轻松与 Grafana 配合使用，如示例 10-9 所示。正如我们之前多次提到的那样，重要的是要意识到，默认的 Linkerd Viz 安装将创建一个仅适用于演示目的的内存中 Prometheus 实例，*绝不能*依赖于生产使用。我们建议您使用单独的 Prometheus 实例来收集 Linkerd 的指标。

# Linkerd 和 Grafana

在较早版本的 Linkerd 中，`linkerd viz install` 自动安装了 Grafana。从 Linkerd 2.12 开始，由于 Grafana 许可证变更，我们不再允许这样做。Grafana 仍然可以与 Linkerd Viz 配合良好，但对于 Linkerd 2.12 及更高版本，您需要手动安装并配置它，使其与 Linkerd Viz 使用相同的 Prometheus 通信。

##### 示例 10-9\. 生产就绪的 Linkerd Viz 安装

```
# The first step of a production-ready Viz dashboard install
# involves installing a standalone Prometheus instance.
# This guide assumes you've done that in the linkerd-viz
# namespace.

# With that done, you can install Grafana.
$ helm repo add grafana https://grafana.github.io/helm-charts
$ helm repo update
$ helm install grafana -n grafana --create-namespace grafana/grafana \
  -f https://raw.githubusercontent.com/linkerd/linkerd2/main/grafana/values.yaml

# The example install uses a values file provided by the
# Linkerd team. It includes important configurations that
# allow the dashboard to properly use Grafana. You can
# read more in the official Linkerd docs:
# https://linkerd.io/2/tasks/grafana/

# After Grafana is installed, install Linkerd Viz and
# tell it to use your Grafana instance.
$ linkerd viz install --set grafana.url=grafana.grafana:3000 \
  | kubectl apply -f -
```

# 审计轨迹和访问日志

强化我们的环境以防止入侵并不仅仅是减少事件风险和影响。拥有强大的安全姿态还意味着能够快速检测到异常事件的发生，并提供数据，使您的安全团队能够准确理解您已经采取的措施。对于 Linkerd，许多这些数据包含在控制平面容器的系统日志中，可通过`kubectl log`访问。确保您的安全团队能够访问和分析日志消息的策略绝对值得。

除了普通的日志消息和事件之外，一些用户需要详细的历史记录，记录所有通过代理传输的 HTTP 请求。这需要*访问日志*。

## 访问日志：好的、坏的和丑陋的

Linkerd 中的访问日志意味着代理将为它处理的每个 HTTP 请求编写一条日志消息。在一个环境中，您有多个服务彼此通信时，这可能很快变成大量的日志消息，因此在实施访问日志之前，务必查阅[官方 Linkerd 文档](https://oreil.ly/zEFj_)。我们将讨论高级概念和实际步骤，但日志是一个在 Linkerd 版本之间可能发生变化的领域，因此在查阅文档后务必测试您的设置。

### 好的

访问日志将为您提供关于应用程序之间交互的极其详细的信息。它是可配置的；您可以以`apache`或`json`格式发出消息，以便更容易以编程方式消费。启用访问日志后，您的安全团队将拥有大量数据，帮助他们了解任何安全事件的影响和范围。

### 糟糕的

存储和处理这些日志非常昂贵，需要显著的工程开销，并且在您的集群中消耗大量资源。您的平台或安全团队将需要管理日志聚合工具和集群上的日志收集代理。启用访问日志将增加运行平台的成本。

### 不好的

在 Linkerd 中，默认情况下禁用 HTTP 访问日志，因为它对代理的 CPU 和延迟性能有影响。这意味着启用后，您的应用响应时间和计算成本将会增加。具体影响程度将大大取决于您的流量级别和类型。

## 启用访问日志

您可以在工作负载、命名空间或 Pod 级别设置访问日志配置。在任何情况下，您都需要设置以下注释：

```
config.linkerd.io/access-log: apache
```

或：

```
config.linkerd.io/access-log: json
```

设置完成后，您需要重新启动目标工作负载以开始收集日志。

我们建议您在将访问日志启用到生产环境之前，测试其对应用性能的影响。这将为您的组织提供决策所需的数据，以便对 Linkerd 中的访问日志做出明智的决策。

# 摘要

在 Linkerd 中，可观察性从简单的指标到访问日志都有涵盖。Linkerd 允许我们了解应用程序的行为、性能和特性，而无需应用程序开发人员进行任何修改。服务网格的强大之处在于允许平台团队将可观察性作为平台功能提供给应用团队。它还确保所有应用可以以统一的方式被理解和比较。
