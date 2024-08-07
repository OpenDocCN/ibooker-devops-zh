# 第十五章：调试 Linkerd

如果您已经走到这一步，您就了解了 Linkerd 作为一个平台提供的有价值工具。它保护您的应用程序，为这些应用程序提供强大的洞察力，并通过使您的连接更可靠来解决基础应用程序和网络问题。在本章中，我们将探讨在需要排除 Linkerd 自身问题时应该做什么。

在所有情况下，诊断 Linkerd 问题的第一步是使用 Linkerd CLI 的内置健康检查工具运行`linkerd check`，正如我们在第六章中详细讨论的那样。 `linkerd check`是一种快速定位 Linkerd 安装中许多常见问题的方法；例如，它可以立即诊断到期的证书，这是实际中导致 Linkerd 停机最常见的问题。

# 诊断数据平面问题

Linkerd 因不需要大量手动处理数据平面而颇有名气；然而，掌握一些基本的故障排除仍然很有用。许多代理问题最终涉及到相似的解决方案集。

## “常见”的 Linkerd 数据平面故障

虽然 Linkerd 通常是一个容错性强的服务网格，但我们看到有几种情况比其他情况更常见。知道如何应对这些情况可能非常有帮助。

### Pod 启动失败

如果遇到注入 Pod 无法启动的情况，第一步将是确定故障发生的确切位置。这就是`kubectl describe pod`和`kubectl logs`命令能提供大量有用信息的地方。

通常最有用的方法是从描述失败的 Pod 开始，了解 Kubernetes 认为发生了什么。例如，Linkerd 初始化容器失败了吗？Pod 是否因其探针未能报告为准备好而被杀死？这些信息可以帮助您决定需要从哪些容器中获取日志，如果需要的话。

如果失败的容器属于应用程序而不是 Linkerd，则最好查看非 Linkerd 特定的潜在原因，如共同依赖项失败，或者节点资源不足。不过，如果是 Linkerd 容器失败，第二步就是了解故障是否影响所有新的 Pod 或仅影响某些新的 Pod：

单个 Pod 失败

如果单个 Pod 失败，您需要查看该 Pod 的特征。哪些容器无法启动？是代理、初始化容器还是应用本身？如果其他 Pod 正常启动，则这不是一个系统性问题，您需要深入研究特定 Pod 的细节。

所有 Pod 都无法启动

如果所有新的 Pod 都无法启动，那么您可能遇到了更严重的系统性问题。正如前面提到的，导致所有 Pod 无法启动的最常见原因是证书问题。`linkerd check`命令将立即显示这类问题，因此我们建议首先运行它。

另一个可能的——尽管不常见的——问题是 Linkerd 代理注入器未运行或不健康。请注意，在高可用模式下运行 Linkerd 时，我们推荐用于生产环境（如第十四章所讨论的），Kubernetes 将拒绝启动任何新的应用程序 Pod，直到注入器健康为止。

一部分 Pod 未能启动

如果只有一些新的 Pod 未能启动，那么现在是开始隔离那些 Pod 上存在的共同因素的时候了：

+   查看 `kubectl get pods -o wide` 的输出，看它们是否已安排在同一节点或多个节点上。

+   查看 `kubectl describe pod` 的输出：初始化容器是否未能成功完成？

+   如果您正在使用 Linkerd CNI 插件，您需要检查网络验证器的状态，并可能需要重新启动该节点上的 CNI 容器。

+   如果您没有使用 Linkerd CNI 插件，请查看 Linkerd 初始化容器的 `kubectl logs` 输出。如果未能成功完成，请尝试查看其运行所在节点的唯一之处。

### 间歇性代理错误

间歇性代理错误可能是最难解决的问题之一。代理的任何间歇性问题都必须难以捕捉和解决。在生产环境中运行 Linkerd 时，建议构建监控以捕获包括以下错误：

权限被拒绝的事件

这些可能代表您平台中的危险配置错误或对环境的真正威胁。您需要收集并分析 Linkerd 代理的日志以便检测这些事件。

协议检测超时

讨论了协议检测，见第四章，这是 Linkerd 在自动识别两个 Pod 之间的流量之前执行的过程。这是一个重要的步骤，在代理可以开始发送和接收流量之前发生。偶尔情况下，协议检测可能会超时，然后回退到将连接视为标准的 TCP 连接。这意味着代理将引入不必要的 10 秒延迟，然后连接将无法从 Linkerd 的请求级路由等功能中受益。

协议检测超时事件通常表明应将某个端口标记为跳过或不透明（再次见第四章）。特别是，代理永远无法正确执行服务器优先协议的协议检测。如果您使用 Linkerd 的策略资源，您可以在给定端口上声明协议。这使您可以完全跳过协议检测，并将提高应用程序的总体可用性和安全性。

代理也可能因过载而无法处理协议检测。

内存不足事件

对于代理的内存不足事件代表着资源分配问题。重要的是要监控、测试和管理 Linkerd 代理的资源限制。它拦截和管理流入和流出应用程序的流量，管理其资源是平台团队的核心职责。

确保为代理提供处理流经其的流量所需的资源，否则你将会度过一个糟糕的一天。

HTTP 错误

一般来说，Linkerd 会重用 Pod 之间的持久 TCP 连接，并显示发生的任何应用程序级错误，因此如果您的应用程序有任何底层配置问题，应该能看到应用程序实际生成的错误。

但是，有时候 Linkerd 代理会直接对请求作出 502、503 或 504 响应，了解导致这些响应的原因非常重要：

502 错误

当新的服务网格用户开始将应用程序添加到服务网格时，看到 502 错误的频率上升并不罕见。这是因为每当 Linkerd 发现代理之间的连接错误时，它将显示为 502 错误。如果您在环境中看到大量的 502 错误，请参考[Linkerd 文档](https://oreil.ly/77p8P)，了解更多可以采取的故障排除步骤。

503 和 504

当 Linkerd 代理发现请求超出工作负载的能力时，会显示 503 和 504 错误。

在正常操作中，Linkerd 代理会维护一个请求调度队列。传入的请求几乎不会在队列中停留：它被排队，代理选择一个可用的端点来处理请求，然后立即调度它。

然而，假设有大量的传入请求涌入，超过工作负载的处理能力。当队列变得过长时，Linkerd 开始*负载剪裁*，任何新到达的请求会直接从代理服务器收到一个 503 错误。要停止负载剪裁，传入请求的速率需要足够慢，以便队列中的请求得以调度，从而允许队列收缩。

此外，端点池是动态的：例如，断路器可以将给定的端点从池中移除。如果池变为空（即没有可用的端点），则队列进入一种称为*快速失败*的状态。在快速失败中，队列中的所有请求立即收到 504 响应，然后 Linkerd 转而进行负载剪裁，因此新的请求再次收到 503 响应。

要摆脱快速失败，某个后端必须再次变得可用。如果由于断路器将所有后端标记为不可用而导致快速失败，那么您最有可能的方式是断路器将允许探测请求通过，它将成功，然后后端将再次标记为可用。在那时，Linkerd 可以将工作负载从快速失败状态中恢复，并且事情将重新开始正常处理。

# 503 不等同于 504！

请注意那里的不同响应！ 504s 仅在负载均衡器进入快速失败时发生，而 503s 指示负载过重——这可能是由于快速失败，也可能是由于流量过大。

## 设置代理日志级别

在正常操作中，Linkerd 代理不会记录调试信息。如果需要，您可以更改 Linkerd 代理的日志级别，而无需重新启动代理。

# 调试日志可能代价高昂

如果设置过于详细的日志级别，代理将消耗更多资源，性能会降低。除非必要，否则不要修改代理的日志级别，并确保在不进行活动调试时重置日志级别。

当您需要开始调试时，可以按照示例 15-1 所示打开调试级别的日志记录。

##### 示例 15-1。打开调试日志

```
# Be sure to replace $POD_NAME with the name of the Pod in question.
$ kubectl port-forward $POD_NAME linkerd-admin

$ curl -v --data 'linkerd=debug' -X PUT localhost:4191/proxy-log-level
```

调试结束后，请务必关闭调试级别的日志记录，如示例 15-2 所示。*不要让代理长时间运行调试级别的日志记录*：这*将*影响性能。

##### 示例 15-2。关闭调试日志

```
# Be sure to replace $POD_NAME with the name of the Pod in question.
$ kubectl port-forward $POD_NAME linkerd-admin

$ curl -v --data 'warn,linkerd2_proxy=info' -X PUT localhost:4191/proxy-log-level
```

日志级别在“访问 Linkerd 日志”中有讨论；您可以在[官方 Linkerd 文档](https://oreil.ly/GNv9I)中找到有关配置 Linkerd 代理日志级别的更多信息。

# 调试 Linkerd 控制平面

Linkerd 的控制平面分为核心控制平面及其扩展（如 Linkerd Viz 和 Linkerd Multicluster）。由于控制平面的每个组件都使用 Linkerd 代理进行通信，您可以利用 Linkerd 的可观察性来进行调试。

## Linkerd 控制平面和可用性

控制平面是所有网格操作的一部分，因此其健康状况至关重要。正如本章开头提到的那样，获取控制平面健康状况的最快方式——除了支付托管服务之外——是运行`linkerd check`。

`linkerd check`将执行一系列详细测试，并验证是否存在已知的配置错误。如果存在问题，`linkerd check`将指导您查看有关如何解决问题的文档。

# 总是从 linkerd check 开始

很难过分强调`linkerd check`的实用性。每当您发现您的网格出现异常情况时，*始终*从`linkerd check`开始。

也要注意`linkerd check`在运行其测试时的顺序是经过深思熟虑的：它首先运行核心控制平面的所有测试，然后按顺序进行每个扩展。在每个部分内部，`linkerd check`通常会按顺序执行其测试，每个测试在能够运行之前都必须通过。如果测试失败，其所在的部分及其位置本身就可以为您提供关于从哪里开始调试的大量信息。

## 核心控制平面

核心控制平面控制 Pod 创建、证书管理以及 Pod 之间的流量路由。正如在第二章中讨论的那样，核心控制平面由三个主要组件组成：代理注入器、目的地控制器和身份控制器。现在我们将深入讨论这些组件的故障模式及其应对方法。

# 在生产中使用 HA 模式

在生产中运行的任何 Linkerd 实例都必须使用高可用性模式，绝不能例外！没有 HA 模式，控制平面中的单个故障可能会使整个 mesh 处于风险之中，这在生产中是不可接受的。您可以在第十四章中详细了解 HA 模式。

### 身份控制器

身份控制器的任何故障都将影响工作负载证书的发放和更新。如果 Pod 在启动时无法获取证书，代理将无法启动，并会记录相关消息。另一方面，如果代理的证书过期，它将开始记录未能更新的日志消息，并且将无法连接到任何新的 Pod。

截至目前，我们只熟悉身份控制器的两种已知故障模式：

过期证书

安装 Linkerd 时，您需要为控制平面提供信任锚证书和身份发行者证书，如第七章中详细讨论的那样。如果身份发行者证书过期，身份控制器将无法继续发出新证书，这将导致整个 mesh 停机，因为各个代理无法再建立安全连接。

# 永远不要让您的证书过期

过期证书将导致整个 mesh 停滞不前，并且它们是导致 Linkerd 生产中断的最常见原因。您*必须*定期监控证书的健康状态。

身份控制器超载

每次创建新的 meshed Pod 时都会访问 Linkerd 的身份控制器。因此，通过创建大量 Pod 可能（虽然困难）会使身份控制器不堪重负。这将表现为 Pod 创建的长时间延迟，以及身份控制器日志中关于证书创建的大量消息。

处理超载身份控制器的最简单方法是进行水平扩展：向`linkerd`命名空间中的`linkerd-identity`部署添加更多副本。您可能还希望考虑允许其请求更多 CPU 资源。

### 目的地控制器

Linkerd 的目的地控制器负责为各个代理提供其路由信息，并提供环境有效策略的详细信息。它对 mesh 的正常运行至关重要，任何退化都应立即视为需要紧急处理的关键问题。这里有两个主要注意事项：

内存

目标控制器的内存使用量随着集群中端点数量的线性增长而增加。对于大多数集群而言，这导致目标控制器随时间消耗的内存量相对稳定。这意味着如果目标控制器被 Kubernetes 的 OOMKilled，它非常可能会达到其内存限制，并在每次重新启动时再次被 OOMKilled。因此，积极监视和管理目标控制器上的内存限制非常重要。

代理缓存

Linkerd 代理会维护一个端点的缓存，以便在目标控制器不可用时由代理重复使用。默认情况下，代理将缓存其端点列表 5 秒，并在任何给定的 5 秒间隔内重复使用该缓存。您可以通过 `linkerd-control-plane` Helm 图表的 `outboundDiscoveryCacheUnusedTimeout` 属性来配置该超时时间。增加超时时间将增加您在目标控制器故障时的整体弹性，特别是对于流量较少的服务而言。

### 代理注入器

Linkerd 的代理注入器是对 Pod 创建事件做出响应的突变 Webhook。在核心控制平面的所有元素中，代理注入器最不可能出现问题，但是积极监视其健康状态是非常重要的：

+   在 HA 模式下，如果代理注入器崩溃，则新的 Pod 将不被允许启动，直到代理注入器恢复在线状态。

+   在非 HA 模式下，如果代理注入器崩溃，则新的 Pod 不会被注入到网格中，直到其恢复在线状态。

# 在生产环境中使用 HA 模式

从上述描述中很明显，为什么在生产环境中使用高可用模式是如此重要。您可以在 第十四章 中了解更多关于 HA 模式的信息。

## Linkerd 扩展

没有 Linkerd 扩展程序位于网格操作的关键路径上，但许多 Linkerd 用户特别使用 Linkerd Viz 扩展程序来收集有关其服务操作的额外细节。如果您发现从 Linkerd Viz 看到异常行为，`linkerd check` 几乎总是能显示出故障是什么，通常最佳的修复方法是升级或重新安装扩展程序。

这种解决 Linkerd Viz 故障的“重启”方法的一个例外是，如果您正在使用其内置的 Prometheus 实例。正如我们在 第十四章 和其他地方讨论的那样，内置的 Prometheus 实例只在内存中存储指标数据。这意味着随着您添加更多指标数据，它*将*周期性地被 Kubernetes 的 OOMKilled，您将丢失它存储的任何数据。这是内置 Prometheus 的已知限制。

# 在生产环境中不要使用内置的 Prometheus

从前面的描述中清楚地可以看出，在生产环境中*绝不能使用内置的 Prometheus 实例*。您可以在 第十四章 中了解更多相关信息。

# 摘要

一般来说，**Linkerd** 是一个表现良好、容错性强的网格（mesh）——特别是在高可用模式下。然而，像所有软件一样，出现问题时需要注意。阅读本章后，你应该知道在出现问题时从何处开始查找。
