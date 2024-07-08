# 第二章：可预测需求

在共享云环境中成功部署、管理和共存应用程序的基础取决于识别和声明应用程序的资源需求和运行时依赖。这种*可预测需求*模式表明了您应如何声明应用程序的需求，无论是硬运行时依赖还是资源需求。声明您的需求对于 Kubernetes 在集群中找到适当位置以运行您的应用程序至关重要。

# 问题

Kubernetes 可以管理用不同编程语言编写的应用程序，只要这些应用程序可以在容器中运行。然而，不同语言具有不同的资源需求。通常，编译语言运行速度更快，通常比即时运行时或解释语言需要更少的内存。考虑到同一类别中许多现代编程语言具有类似的资源需求，从资源消耗的角度来看，更重要的方面是应用程序的领域、业务逻辑和实际实现细节。

除了资源需求外，应用程序运行时还依赖于平台管理的能力，如数据存储或应用程序配置。

# 解决方案

了解容器的运行时需求主要有两个重要原因。首先，通过定义所有运行时依赖和资源需求，Kubernetes 可以智能地决定在集群中何处放置容器，以实现最高效的硬件利用率。在具有不同优先级的大量进程共享资源的环境中，确保成功共存的唯一方法是提前了解每个进程的需求。然而，智能放置只是其中一方面。

容器资源配置文件对于容量规划也是至关重要的。根据特定服务的需求和服务的总数，我们可以为不同的环境做一些容量规划，并提出最具成本效益的主机配置文件，以满足整个集群的需求。服务资源配置文件和容量规划手段长期成功管理集群密不可分。

在深入了解资源配置文件之前，让我们看看如何声明运行时依赖。

## 运行时依赖

最常见的运行时依赖之一是文件存储，用于保存应用程序状态。容器文件系统是临时的，在容器关闭时会丢失。Kubernetes 提供了卷作为 Pod 级别的存储工具，可在容器重新启动时保留数据。

最直接的卷类型是 `emptyDir`，它与 Pod 同存亡。当 Pod 被移除时，其内容也会丢失。该卷需要由另一种存储机制支持，以便在 Pod 重启时生存下来。如果您的应用程序需要读取或写入文件到这种长期存储中，必须在容器定义中明确声明这种依赖关系，如示例 2-1 所示。

##### 示例 2-1\. 对 PersistentVolume 的依赖

```
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    volumeMounts:
    - mountPath: "/logs"
      name: log-volume
  volumes:
  - name: log-volume
    persistentVolumeClaim:  ![1](img/1.png)
      claimName: random-generator-log
```

![1](img/#co_predictable_demands_CO1-1)

PersistentVolumeClaim (PVC) 的依赖要求其存在且已绑定。

调度器评估 Pod 需要的卷类型，这影响 Pod 放置的位置。如果 Pod 需要的卷在集群的任何节点上都没有提供，那么根本不会调度该 Pod。卷是运行时依赖关系的一个例子，它影响 Pod 可以运行在什么样的基础设施上，以及是否可以被调度。

当您要求 Kubernetes 通过 `hostPort` 在主机系统上的特定端口上公开容器端口时，会发生类似的依赖关系。使用 `hostPort` 在集群中的每个节点上保留端口，并且每个节点最多只能调度一个 Pod。由于端口冲突，您可以扩展到 Kubernetes 集群中的节点数量相同的 Pod。

配置是另一种依赖类型。几乎每个应用程序都需要一些配置信息，Kubernetes 提供的推荐解决方案是通过 ConfigMaps。您的服务需要一种消耗设置的策略——通过环境变量或文件系统。无论哪种情况，这都会将容器对命名 ConfigMaps 的命名引入运行时依赖。如果未创建所有预期的 ConfigMaps，容器会被调度到节点上，但不会启动。

与 ConfigMaps 类似，Secrets 提供了一种稍微更安全的方法来向容器分发特定于环境的配置。使用 Secret 的方式与 ConfigMaps 的方式相同，并且使用 Secret 从容器到命名空间引入了相同类型的依赖关系。

ConfigMaps 和 Secrets 在第二十章，“配置资源”中有更详细的解释，示例 2-2 展示了这些资源作为运行时依赖的使用方式。

##### 示例 2-2\. 对 ConfigMap 的依赖

```
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    env:
    - name: PATTERN
      valueFrom:
        configMapKeyRef:  ![1](img/1.png)
          name: random-generator-config
          key: pattern
```

![1](img/#co_predictable_demands_CO2-1)

对 ConfigMap `random-generator-config` 的强制依赖。

虽然创建 ConfigMap 和 Secret 对象是我们必须执行的简单部署任务，但集群节点提供存储和端口号。其中一些依赖限制了 Pod 可以调度的位置（如果有的话），而其他依赖可能会阻止 Pod 启动。在设计带有这些依赖的容器化应用程序时，始终要考虑它们将在运行时创建的约束条件。

## 资源配置文件

指定诸如 ConfigMap、Secret 和卷等容器依赖关系非常简单。我们需要更多的思考和实验来确定容器的资源需求。在 Kubernetes 的上下文中，计算资源被定义为容器可以请求、分配和消耗的东西。这些资源被分类为*可压缩*（即可以被限制，如 CPU 或网络带宽）和*不可压缩*（即不能被限制，如内存）。

区分可压缩和不可压缩资源的区别非常重要。如果您的容器消耗了太多的可压缩资源，例如 CPU，它们将被限制，但如果它们使用了太多的不可压缩资源（如内存），它们将被终止（因为没有其他方法要求应用程序释放已分配的内存）。

根据您的应用程序的性质和实现细节，您必须指定所需的最小资源量（称为 `requests`）以及它可以增长到的最大量（`limits`）。每个容器定义可以指定它所需的 CPU 和内存量，形式为请求和限制。在高级别上，`requests`/`limits` 的概念类似于软限制/硬限制。例如，类似地，我们通过使用 `-Xms` 和 `-Xmx` 命令行选项为 Java 应用程序定义堆大小。

`requests` 量（但不包括 `limits`）在调度 Pod 到节点时由调度程序使用。对于给定的 Pod，调度程序仅考虑仍然具有足够容量来容纳 Pod 及其所有容器的节点，通过汇总请求的资源量。从这个意义上说，每个容器的 `requests` 字段影响 Pod 是否可以被调度。

##### Example 2-3\. 资源限制

```
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    resources:
      requests:        ![1](img/1.png)
        cpu: 100m
        memory: 200Mi
      limits:          ![2](img/2.png)
        memory: 200Mi
```

![1](img/#co_predictable_demands_CO3-1)

初始的 CPU 和内存资源请求。

![2](img/#co_predictable_demands_CO3-2)

直到我们希望我们的应用程序增长到最大的上限。我们故意不指定 CPU 的限制。

下面的资源类型可以作为 `requests` 和 `limits` 规范中的键使用：

`memory`

该类型用于应用程序的堆内存需求，包括配置为 `medium: Memory` 的 `emptyDir` 类型卷。内存资源是不可压缩的，因此超出配置的内存限制的容器将触发 Pod 被驱逐；即，它可能会被删除并且可能重新创建在另一个节点上。

`cpu`

`cpu` 类型用于指定应用程序所需的 CPU 循环范围。然而，它是一个可压缩资源，这意味着在节点超分配的情况下，所有运行容器的分配 CPU 槽位将按照其指定的请求进行限制。因此，强烈建议您为 CPU 资源设置 `requests`，但是不要设置 `limits`，这样它们可以从所有多余的 CPU 资源中受益，否则这些资源将被浪费。

`ephemeral-storage`

每个节点都有一些专门用于临时存储的文件系统空间，用于保存日志和可写容器层。未存储在内存文件系统中的 `emptyDir` 卷也会使用临时存储。使用此请求和限制类型，您可以指定应用程序的最小和最大需求。`ephemeral-storage` 资源是不可压缩的，如果 Pod 使用的存储空间超过其 `limit` 中指定的值，则会导致 Pod 从节点驱逐。

`hugepage-<size>`

*Huge pages* 是大的、连续预分配的内存页，可以作为卷挂载。根据您的 Kubernetes 节点配置，有多种大小的 huge pages 可供选择，如 2 MB 和 1 GB 的页。您可以指定请求和限制，以确定您想要消耗某种类型的 huge pages 的数量（例如，`hugepages-1Gi: 2Gi` 表示请求两个 1 GB 的 huge pages）。Huge pages 不能超分配，因此请求和限制必须相同。

根据您是否指定了 `requests`、`limits` 或两者，平台提供三种类型的服务质量（QoS）：

Best-Effort

对于其容器未设置任何请求和限制的 Pod，其 QoS 是 *Best-Effort*。这样的 *Best-Effort* Pod 被认为是最低优先级的，当放置 Pod 的节点耗尽不可压缩资源时，很可能首先被杀死。

Burstable

对于 `requests` 和 `limits` 值不相等（并且如预期的那样，`limits` 大于 `requests`）的 Pod 被标记为 *Burstable*。这样的 Pod 具有最小的资源保证，但也愿意在可用时消耗更多资源直到其 `limit`。当节点处于不可压缩资源压力下时，这些 Pod 如果没有 *Best-Effort* Pod，则可能会被杀死。

Guaranteed

对于具有相等的 `request` 和 `limit` 资源的 Pod 属于 *Guaranteed* QoS 类别。这些是最高优先级的 Pod，在 *Best-Effort* 和 *Burstable* Pod 之前保证不会被杀死。对于您的应用程序的内存资源，这种 QoS 模式是最佳选择，因为它涉及最少的意外并避免由于内存不足而触发的驱逐。

因此，您为容器定义或省略的资源特性直接影响其 QoS，并定义 Pod 在资源匮乏情况下的相对重要性。考虑到这一后果，请定义您的 Pod 资源需求。

## Pod 优先级

我们解释了容器资源声明如何定义 Pods 的 QoS 并在资源匮乏时影响 Kubelet 杀死 Pod 中的容器的顺序。另外两个相关概念是 Pod 优先级 和 抢占。*Pod 优先级* 允许您指示 Pod 相对于其他 Pods 的重要性，这影响 Pods 被调度的顺序。让我们在 示例 2-4 中看看它是如何运作的。

##### 示例 2-4\. Pod 优先级

```
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority  ![1](img/1.png)
value: 1000            ![2](img/2.png)
globalDefault: false   ![3](img/3.png)
description: This is a very high-priority Pod class
---
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
  labels:
    env: random-generator
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
  priorityClassName: high-priority  ![4](img/4.png)
```

![1](img/#co_predictable_demands_CO4-1)

优先级类对象的名称。

![2](img/#co_predictable_demands_CO4-2)

对象的优先级值。

![3](img/#co_predictable_demands_CO4-3)

`globalDefault` 设为 `true` 用于未指定 `priorityClassName` 的 Pods。只能有一个 PriorityClass 的 `globalDefault` 设为 `true`。

![4](img/#co_predictable_demands_CO4-4)

用于此 Pod 的优先级类，如在 PriorityClass 资源中定义。

我们创建了一个 PriorityClass，这是一个非命名空间对象，用于定义基于整数的优先级。我们的 PriorityClass 名为 `high-priority`，优先级为 1,000。现在我们可以通过其名称将此优先级分配给 Pods，例如 `priorityClassName:` `high-priority`。PriorityClass 是一种指示 Pods 相对重要性的机制，其中更高的值表示更重要的 Pods。

Pod 优先级影响调度程序在节点上放置 Pods 的顺序。首先，优先级准入控制器使用 `priorityClassName` 字段为新 Pods 填充优先级值。当多个 Pods 等待放置时，调度程序首先按最高优先级对待处理 Pods 排序。任何等待处理的 Pod 都会在排队中的任何其他优先级较低的 Pod 之前被选中，并且如果没有阻止其调度的约束条件，则会被调度。

现在来看关键部分。如果没有节点具有足够的容量来放置一个 Pod，则调度程序可以抢占（移除）节点上的优先级较低的 Pods，以释放资源并放置优先级较高的 Pods。因此，如果满足所有其他调度要求，则具有更高优先级的 Pod 可能比具有较低优先级的 Pods 更早地被调度。该算法有效地使集群管理员能够通过允许调度程序驱逐优先级较低的 Pods 为更关键的工作负载腾出空间，并首先放置它们。如果无法调度一个 Pod，则调度程序继续安排其他优先级较低的 Pods。

假设您希望您的 Pod 以特定优先级调度，但不想驱逐任何现有的 Pods。在这种情况下，您可以将 PriorityClass 标记为字段 `preemptionPolicy: Never`。分配到此优先级类别的 Pods 将不会触发任何正在运行的 Pods 的驱逐，但仍会根据其优先级值进行调度。

Pod QoS（前文讨论过）和 Pod 优先级是两个独立的特性，它们没有连接并且只有少量重叠。QoS 主要由 Kubelet 在可用计算资源不足时用于保护节点稳定性。Kubelet 在驱逐之前首先考虑 QoS，然后再考虑 Pods 的 PriorityClass。另一方面，调度程序的驱逐逻辑在选择抢占目标时完全忽略 Pods 的 QoS。调度程序尝试选择一组具有最低优先级的 Pods，以满足等待放置的更高优先级 Pods 的需求。

当 Pod 指定了优先级时，可能会对被驱逐的其他 Pod 产生不良影响。例如，尽管尊重了 Pod 的优雅终止策略，但如第十章，“单例服务”所述的 PodDisruptionBudget 并不受保证，这可能会破坏依赖于一组 Pod 团队的较低优先级集群应用。

另一个问题是恶意或不知情的用户创建具有最高优先级的 Pods 并驱逐所有其他 Pods。为防止这种情况发生，ResourceQuota 已扩展以支持 PriorityClass，并且较高优先级的数字保留用于不应通常被抢占或驱逐的关键系统 Pods。

总之，应谨慎使用 Pod 优先级，因为用户指定的数值优先级会指导调度程序和 Kubelet 放置或终止哪些 Pods，用户可能会利用这一点进行操作。任何更改都可能影响多个 Pods，并可能阻止平台提供可预测的服务级别协议。

## 项目资源

Kubernetes 是一个自助服务平台，使开发人员可以在指定的隔离环境中按其需求运行应用程序。然而，在共享的多租户平台上工作也需要特定边界和控制单元，以防止某些用户消耗平台的所有资源。其中一种工具是 ResourceQuota，它提供了命名空间中限制聚合资源消耗的约束条件。通过 ResourceQuotas，集群管理员可以限制在命名空间中消耗的计算资源总和（如 CPU、内存）和存储。它还可以限制创建在命名空间中的对象总数（如 ConfigMaps、Secrets、Pods 或 Services）。示例 2-5 展示了限制某些资源使用的实例。请参阅官方 Kubernetes 文档中有关 [Resource Quotas](https://oreil.ly/TLRMe) 的完整支持资源列表，您可以使用 ResourceQuotas 限制这些资源的使用。

##### 示例 2-5\. 资源约束的定义。

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
  namespace: default   ![1](img/1.png)
spec:
  hard:
    pods: 4            ![2](img/2.png)
    limits.memory: 5Gi ![3](img/3.png)
```

![1](img/#co_predictable_demands_CO5-1)

应用资源约束的命名空间。

![2](img/#co_predictable_demands_CO5-2)

允许此命名空间中有四个活跃的 Pods。

![3](img/#co_predictable_demands_CO5-3)

此命名空间中所有 Pods 的内存限制总和不能超过 5 GB。

在这一领域的另一个有用工具是 LimitRange，它允许你为每种资源类型设置资源使用限制。除了指定不同资源类型的最小和最大允许量及这些资源的默认值之外，它还允许你控制 `requests` 和 `limits` 之间的比例，也称为*过度分配级别*。示例 2-6 展示了一个 LimitRange 和可能的配置选项。

##### 示例 2-6\. 允许和默认资源使用限制的定义。

```
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
  namespace: default
spec:
  limits:
  - min:                  ![1](img/1.png)
      memory: 250Mi
      cpu: 500m
    max:                  ![2](img/2.png)
      memory: 2Gi
      cpu: 2
    default:              ![3](img/3.png)
      memory: 500Mi
      cpu: 500m
    defaultRequest:       ![4](img/4.png)
      memory: 250Mi
      cpu: 250m
    maxLimitRequestRatio: ![5](img/5.png)
      memory: 2
      cpu: 4
    type: Container       ![6](img/6.png)
```

![1](img/#co_predictable_demands_CO6-1)

请求和限制的最小值。

![2](img/#co_predictable_demands_CO6-2)

请求和限制的最大值。

![3](img/#co_predictable_demands_CO6-3)

在未指定限制时的默认值。

![4](img/#co_predictable_demands_CO6-4)

在未指定请求时的默认值。

![5](img/#co_predictable_demands_CO6-5)

最大比例限制/请求，用于指定允许的过度分配级别。在这里，内存限制不能大于内存请求的两倍，而 CPU 限制可以高达 CPU 请求的四倍。

![6](img/#co_predictable_demands_CO6-6)

类型可以是 `Container`、`Pod`（所有容器的合并），或 `PersistentVolumeClaim`（为请求持久卷指定范围）。

LimitRange 有助于控制容器资源配置文件，以便没有容器需要比集群节点提供的资源更多。LimitRange 还可以防止集群用户创建消耗大量资源的容器，使得节点无法为其他容器分配资源。考虑到 `requests`（而不是 `limits`）是调度程序用于放置的主要容器特征，LimitRequestRatio 允许你控制容器的 `requests` 和 `limits` 之间的差异量。`requests` 和 `limits` 的大差距增加了在节点上过度分配的可能性，并且当许多容器同时需要比最初请求的资源更多时，可能会降低应用程序的性能。

请记住，其他共享的节点级资源，如进程 ID（PIDs），可能在达到任何资源限制之前耗尽。Kubernetes 允许您为系统使用保留一些节点 PIDs，并确保它们永远不会被用户工作负载耗尽。类似地，Pod PID 限制允许集群管理员限制 Pod 中运行的进程数量。由于这些设置是由集群管理员设置为 Kubelet 配置选项，而不是由应用程序开发人员使用，我们在此不对其进行详细审查。

## 容量规划

考虑到容器在不同环境中可能具有不同的资源配置文件和各种实例数量，显然，多用途环境的容量规划并不简单。例如，在非生产集群上，为了实现最佳硬件利用率，您可能主要有 *Best-Effort* 和 *Burstable* 类型的容器。在这样一个动态的环境中，许多容器同时启动和关闭，即使一个容器在资源饥饿时被平台终止，也不是致命的。在生产集群中，我们希望事情更稳定和可预测，容器可能主要是 *Guaranteed* 类型，部分可能是 *Burstable* 类型。如果一个容器被终止，这很可能是集群容量应增加的迹象。

表 2-1 展示了几个具有 CPU 和内存需求的服务。

表 2-1. 容量规划示例

| Pod | CPU 请求 | 内存请求 | 内存限制 | 实例数 |
| --- | --- | --- | --- | --- |
| A | 500 m | 500 Mi | 500 Mi | 4 |
| B | 250 m | 250 Mi | 1000 Mi | 2 |
| C | 500 m | 1000 Mi | 2000 Mi | 2 |
| D | 500 m | 500 Mi | 500 Mi | 1 |
| **总计** | **4000 m** | **5000 Mi** | **8500 Mi** | **9** |

当然，在实际场景中，您使用 Kubernetes 等平台的更可能原因是有更多的服务需要管理，其中一些即将退役，一些仍处于设计和开发阶段。即使它是一个不断变化的目标，基于先前描述的类似方法，我们可以计算每个环境所有服务所需的总资源量。

在不同的环境中，请记住，容器的数量各不相同，甚至可能需要留出一些空间用于自动扩展、构建作业、基础设施容器等。根据这些信息和基础设施提供者，您可以选择提供所需资源的最具成本效益的计算实例。

# 讨论

容器不仅用于进程隔离和作为打包格式，还是成功容量规划的构建块，具有确定的资源配置文件。执行一些早期测试，以了解每个容器的资源需求，并将该信息作为未来容量规划和预测的基础。

Kubernetes 可以通过 *垂直 Pod 自动缩放器*（VPA）来帮助你，在一段时间内监控 Pod 的资源消耗，并推荐请求和限制。VPA 在 “垂直 Pod 自动缩放” 中有详细描述。

然而，更重要的是，资源配置文件是应用程序与 Kubernetes 通信以帮助调度和管理决策的方式。如果你的应用程序没有提供任何 `requests` 或 `limits`，那么 Kubernetes 只能将你的容器视为在集群变满时会被丢弃的不透明盒子。因此，对于每个应用程序来说，思考并提供这些资源声明几乎是强制性的。

现在你知道如何调整我们的应用程序大小，在 第三章，“声明式部署”，你将学习在 Kubernetes 上安装和更新我们的应用程序的多种策略。

# 更多信息

+   [可预测需求示例](https://oreil.ly/HYIqJ)

+   [配置 Pod 使用 ConfigMap](https://oreil.ly/c54Gh)

+   [Kubernetes 最佳实践：资源请求和限制](https://oreil.ly/8bKD5)

+   [Pod 和容器的资源管理](https://oreil.ly/a37eO)

+   [管理 HugePages](https://oreil.ly/RXQD1)

+   [为命名空间配置默认内存请求和限制](https://oreil.ly/ozlU1)

+   [节点压力驱逐](https://oreil.ly/fxRvs)

+   [Pod 的优先级和抢占](https://oreil.ly/FpUoH)

+   [配置 Pod 的服务质量](https://oreil.ly/x07OT)

+   [Kubernetes 的资源服务质量](https://oreil.ly/yORlL)

+   [资源配额](https://oreil.ly/rFSLa)

+   [限制范围](https://oreil.ly/1bXfO)

+   [进程 ID 的限制和预留](https://oreil.ly/lkmMK)

+   [亲爱的上帝，请停止在 Kubernetes 上使用 CPU 限制](https://oreil.ly/Yk-Ag)

+   [Kubernetes 内存限制：所有人都应该知道的内容](https://oreil.ly/cdJkP)
