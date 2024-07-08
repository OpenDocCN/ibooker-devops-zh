# 第九章：ReplicaSets

我们已经讨论了如何将单个容器作为 Pods 运行，但这些 Pods 本质上是一次性单例。通常情况下，出于多种原因，您希望在特定时间运行多个容器的副本：

冗余性

通过运行多个实例来容忍故障。

规模

通过运行多个实例提高请求处理能力。

分片

不同的副本可以并行处理计算的不同部分。

当然，你可以手动使用多个不同的 Pod 清单创建多个 Pod 的副本，但这样做既繁琐又容易出错。从逻辑上讲，管理一组副本 Pods 的用户将其视为一个单一实体来定义和管理——这正是 ReplicaSet 的作用。ReplicaSet 充当集群范围的 Pod 管理器，确保正确类型和数量的 Pods 随时运行。

由于 ReplicaSets 可以轻松创建和管理一组副本的 Pods，它们是常见应用部署模式和基础设施级自愈应用的基础模块。

将 ReplicaSet 理解为将 cookie 切割机和所需数量的 cookie 结合成一个单一的 API 对象的最简单方法。当我们定义一个 ReplicaSet 时，我们定义了要创建的 Pods 的规范（“cookie 切割机”）和所需数量的副本。此外，我们需要定义一种查找 ReplicaSet 应控制的 Pods 的方法。管理复制 Pods 的实际行为是*协调循环*的一个示例。这样的循环是 Kubernetes 设计和实现的大部分基础。

# 协调循环

协调循环背后的核心概念是“期望”状态与“观察到”的或“当前”的状态的概念。期望状态是你想要的状态。在 ReplicaSet 中，它是副本的期望数量和要复制的 Pod 的定义。例如，“期望的状态是有三个 `kuard` 服务器的 Pod 副本正在运行”。相反，当前状态是系统当前观察到的状态。例如，“当前只有两个 `kuard` Pod 正在运行”。

协调循环不断运行，观察世界的当前状态并采取行动，以尝试使观察到的状态与期望状态匹配。例如，在前面的例子中，协调循环将创建一个新的 `kuard` Pod，以使观察到的状态与期望的三个副本的状态匹配。

采用对和解调循环管理状态的方法有许多好处。它是一个本质上以目标驱动、自我修复的系统，但通常可以用几行代码轻松表达。例如，ReplicaSets 的解调循环是一个单一的循环，但它处理用户动作来扩展或缩减 ReplicaSet，以及节点故障或节点重新加入集群后的情况。

我们将在本书的其余部分中看到大量和解调循环相关的示例。

# 关联 Pods 和 ReplicaSets

解耦是 Kubernetes 的一个关键主题。特别重要的是，Kubernetes 的所有核心概念在彼此之间都是模块化的，并且它们可以互换和替换为其他组件。在这种精神下，ReplicaSet 和 Pod 之间的关系是松散耦合的。虽然 ReplicaSet 创建和管理 Pods，但它们不拥有它们创建的 Pods。ReplicaSet 使用标签查询来标识它们应该管理的一组 Pods。然后，它们使用您在 第五章 中直接使用的完全相同的 Pod API 来创建它们正在管理的 Pods。这种“从前门进入”的概念是 Kubernetes 中的另一个核心设计概念。在类似的解耦中，创建多个 Pods 的 ReplicaSets 和负载均衡到这些 Pods 的服务也是完全分开的、解耦的 API 对象。除了支持模块化外，解耦 Pods 和 ReplicaSets 还启用了几个重要的行为，将在接下来的几节中讨论。

## 采用现有的容器

尽管声明式配置很有价值，但有时以命令式方式构建某些东西会更容易。特别是在初期阶段，您可能只是部署一个带有容器镜像的单个 Pod，而不是由 ReplicaSet 进行管理。您甚至可以定义一个负载均衡器来为该单个 Pod 提供流量服务。

但是在某些时候，您可能希望将您的单例容器扩展为复制服务，并创建和管理一系列类似的容器。如果 ReplicaSets 拥有它们创建的 Pods，那么复制您的 Pod 的唯一方法将是删除它，并通过 ReplicaSet 重新启动它。这可能会造成干扰，因为在您的容器不运行的时候会有一段时间。然而，由于 ReplicaSets 与它们管理的 Pods 解耦，您可以简单地创建一个“采用”现有 Pod 的 ReplicaSet，并扩展这些容器的额外副本。通过这种方式，您可以从一个命令式的单一 Pod 无缝过渡到由 ReplicaSet 管理的复制 Pods 集合。

## 隔离容器

通常情况下，当服务器表现不佳时，Pod 级别的健康检查会自动重新启动该 Pod。但是如果您的健康检查不完整，一个 Pod 可能会表现不佳，但仍然是复制集的一部分。在这些情况下，简单地杀死 Pod 可以解决问题，但这样做会使开发人员只能依赖日志来调试问题。相反，您可以修改患病 Pod 上的标签集。这样做将使其与 ReplicaSet（和服务）解除关联，以便您可以调试该 Pod。ReplicaSet 控制器将注意到一个 Pod 缺失并创建一个新副本，但由于 Pod 仍在运行，开发人员可以进行交互式调试，这比仅仅依赖日志进行调试要有价值得多。

# 使用 ReplicaSets 进行设计

ReplicaSets 的设计用于表示体系结构中的单个可扩展微服务。它们的关键特性是 ReplicaSet 控制器创建的每个 Pod 都是完全同质的。通常，这些 Pod 然后由 Kubernetes 服务负载均衡器前端化，该负载均衡器在服务中分发流量到组成服务的各个 Pod 上。一般来说，ReplicaSets 设计用于无状态（或几乎无状态）服务。它们创建的元素是可互换的；当缩减 ReplicaSet 时，会选择一个任意的 Pod 进行删除。由于这种缩减操作，你的应用行为不应该发生改变。

###### 注意

通常，你会看到应用程序使用 Deployment 对象，因为它允许你管理新版本的发布。Repli⁠ca​Sets 在幕后支持 Deployments，了解它们的操作方式非常重要，以便在需要排除故障时进行调试。

# ReplicaSet 规范

就像 Kubernetes 中的所有对象一样，ReplicaSets 是使用规范定义的。所有的 ReplicaSets 必须有一个唯一的名称（使用 `metadata.name` 字段定义），一个 `spec` 部分来描述集群中任何给定时间应该运行的 Pod（副本）数量，以及一个 Pod 模板，描述当定义的副本数未达到时应创建的 Pod。 示例 9-1 展示了一个最小化的 ReplicaSet 定义。注意规范中的 replicas、selector 和 template 部分，因为它们提供了关于 ReplicaSets 操作方式更深入的见解。

##### 示例 9-1\. kuard-rs.yaml

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  labels:
    app: kuard
    version: "2"
  name: kuard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
      version: "2"
  template:
    metadata:
      labels:
        app: kuard
        version: "2"
    spec:
      containers:
        - name: kuard
          image: "gcr.io/kuar-demo/kuard-amd64:green"
```

## Pod 模板

正如前面提到的，当当前状态中的 Pod 数量少于期望状态中的 Pod 数量时，ReplicaSet 控制器将使用 ReplicaSet 规范中包含的模板创建新的 Pod。这些 Pod 的创建方式与您在前几章中从 YAML 文件创建 Pod 的方式完全相同，但不是使用文件，而是直接基于 Pod 模板创建和提交 Pod 清单到 API 服务器。以下是 ReplicaSet 中的一个 Pod 模板示例：

```
template:
  metadata:
    labels:
      app: helloworld
      version: v1
  spec:
    containers:
      - name: helloworld
        image: kelseyhightower/helloworld:v1
        ports:
          - containerPort: 80
```

## 标签

在任何合理大小的集群中，许多不同的 Pod 同时运行—那么 ReplicaSet 协调循环如何发现特定 ReplicaSet 的一组 Pod？ReplicaSets 使用一组 Pod 标签来监视集群状态，以过滤 Pod 列表并跟踪集群中运行的 Pods。初始创建时，ReplicaSet 从 Kubernetes API 获取一个 Pod 列表，并通过标签进行结果过滤。根据查询返回的 Pod 数量，ReplicaSet 删除或创建 Pods 以满足所需的副本数量。这些过滤标签在 ReplicaSet 的`spec`部分中定义，并且是理解 ReplicaSets 工作原理的关键。

###### 注意

ReplicaSet 中的选择器`spec`应该是 Pod 模板中标签的一个合适的子集。

# 创建 ReplicaSet

ReplicaSets 是通过向 Kubernetes API 提交一个 ReplicaSet 对象来创建的。在本节中，我们将使用配置文件和`kubectl apply`命令创建一个 ReplicaSet。

在示例 9-1 中的 ReplicaSet 配置文件将确保`gcr.io/kuar-demo/kuard-amd64:green`容器的一个副本在任何给定时间内运行。使用`kubectl apply`命令将`kuard` ReplicaSet 提交到 Kubernetes API：

```
$ kubectl apply -f kuard-rs.yaml
replicaset "kuard" created
```

一旦接受了`kuard` ReplicaSet，ReplicaSet 控制器将检测到没有符合所需状态的`kuard` Pods 正在运行，并基于 Pod 模板的内容创建一个新的`kuard` Pod：

```
$ kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
kuard-yvzgd   1/1       Running   0          11s
```

# 检查 ReplicaSet

与 Pod 和其他 Kubernetes API 对象一样，如果您对 ReplicaSet 的更多详细信息感兴趣，可以使用`describe`命令提供关于其状态的更多信息。以下是使用`describe`获取我们之前创建的 ReplicaSet 详细信息的示例：

```
$ kubectl describe rs kuard
Name:         kuard
Namespace:    default
Selector:     app=kuard,version=2
Labels:       app=kuard
              version=2
Annotations:  <none>
Replicas:     1 current / 1 desired
Pods Status:  1 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
```

您可以查看 ReplicaSet 的标签选择器，以及它管理的所有副本的状态。

## 从 Pod 查找 ReplicaSet

有时您可能会想知道 Pod 是否由 ReplicaSet 管理，如果是，是哪一个。为了启用这种发现功能，ReplicaSet 控制器会向它创建的每个 Pod 添加一个`ownerReferences`部分。如果您运行以下命令，请查找`ownerReferences`部分：

```
$ kubectl get pods <*pod-name*> -o=jsonpath='{.metadata.ownerReferences[0].name}'
```

如果适用，这将列出管理此 Pod 的 ReplicaSet 的名称。

## 查找 ReplicaSet 的一组 Pods

您还可以确定由 ReplicaSet 管理的 Pod 集合。首先，使用`kubectl describe`命令获取标签集合。在前面的示例中，标签选择器是`app=kuard,version=2`。要查找与此选择器匹配的 Pods，使用`--selector`标志或简写`-l`：

```
$ kubectl get pods -l app=kuard,version=2
```

这与 ReplicaSet 执行以确定当前 Pod 数量完全相同的查询。

# 扩展 ReplicaSets

通过更新存储在 Kubernetes 中的 ReplicaSet 对象上的`spec.replicas`键，您可以将 ReplicaSets 进行缩放。当您扩展一个 ReplicaSet 时，它会使用 ReplicaSet 上定义的 Pod 模板向 Kubernetes API 提交新的 Pods。

## 使用 kubectl scale 进行命令式扩展

使用`kubectl`中的`scale`命令是实现这一点的最简单方法。例如，要扩展到四个副本，您可以运行：

```
$ kubectl scale replicasets kuard --replicas=4
```

尽管这样的命令式命令对于演示和快速响应紧急情况（如负载突然增加）非常有用，但同样重要的是，更新任何文本文件配置以匹配通过命令式`scale`命令设置的副本数。当您考虑以下情景时，其重要性就变得明显起来。

当 Alice 正在值班时，服务的负载突然增加。Alice 使用`scale`命令将响应请求的服务器数量增加到 10 个，并解决了这种情况。然而，Alice 忘记更新已经检入源代码控制的 ReplicaSet 配置。

几天后，Bob 正在准备每周的部署。Bob 编辑存储在版本控制中的 ReplicaSet 配置，以使用新的容器映像，但他没有注意到文件中当前副本的数量是 5，而不是 Alice 在响应增加负载时设置的 10 个。Bob 继续进行部署，这既更新了容器映像，又将副本数量减半。这导致立即超载，进而导致停机。

这个虚构的案例研究说明了确保任何命令式更改立即后跟源代码控制中的声明式更改的必要性。确实，如果需求不紧急，我们通常建议只进行如下部分所述的声明性更改。

## 使用 kubectl apply 进行声明性扩展

在声明性世界中，您通过编辑版本控制中的配置文件进行更改，然后将这些更改应用于集群。要扩展`kuard` ReplicaSet，请编辑*kuard-rs.yaml*配置文件，并将`replicas`计数设置为`3`：

```
...
spec:
  replicas: 3
...
```

在多用户环境中，您可能会对此更改进行文档化的代码审查，并最终将更改检入版本控制。无论哪种方式，然后您可以使用`kubectl apply`命令将更新的`kuard` ReplicaSet 提交到 API 服务器：

```
$ kubectl apply -f kuard-rs.yaml
replicaset "kuard" configured
```

现在更新后的`kuard` ReplicaSet 已经就位，ReplicaSet 控制器将检测到期望的 Pod 数量已更改，并且需要采取措施来实现该期望状态。如果您在前一节中使用了命令式的`scale`命令，ReplicaSet 控制器将销毁一个 Pod，使数量变为三个。否则，它将使用在`kuard` ReplicaSet 上定义的 Pod 模板向 Kubernetes API 提交两个新的 Pod。无论如何，请使用`kubectl get pods`命令列出正在运行的`kuard` Pods。您应该会看到类似以下内容的输出，其中有三个处于运行状态的 Pod；两个由于最近启动，其年龄较小：

```
$ kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
kuard-3a2sb   1/1       Running   0          26s
kuard-wuq9v   1/1       Running   0          26s
kuard-yvzgd   1/1       Running   0          2m
```

## 自动缩放 ReplicaSet

尽管有时您希望显式控制 ReplicaSet 中副本的数量，但通常您只需具有足够多的副本即可。定义的具体内容取决于 ReplicaSet 中容器的需求。例如，对于像 NGINX 这样的 Web 服务器，您可能会因 CPU 使用率而进行扩展。对于内存缓存，您可能会根据内存消耗进行扩展。在某些情况下，您可能希望根据自定义应用程序指标进行扩展。Kubernetes 可以通过 *Horizontal Pod Autoscaling*（HPA）处理所有这些情况。

“Horizontal Pod Autoscaling” 这个名字有点拗口，您可能会想为什么不直接叫 “autoscaling”。Kubernetes 区分了 *horizontal* 缩放（涉及创建 Pod 的额外副本）和 *vertical* 缩放（涉及增加特定 Pod 所需的资源，例如增加 Pod 所需的 CPU）。许多解决方案还支持 *cluster* 自动缩放，即根据资源需求调整集群中机器的数量，但这超出了本章的范围。

###### 注意

自动缩放需要在集群中存在 `metrics-server`。`metrics-server` 负责跟踪指标并提供一个 API 用于消耗 HPA 在制定缩放决策时使用的指标。大多数 Kubernetes 安装默认包含 `metrics-server`。您可以通过列出 `kube-system` 命名空间中的 Pods 来验证其存在：

```
$ kubectl get pods --namespace=kube-system
```

在列表中应该看到一个名称以 `metrics-server` 开头的 Pod。如果没有看到它，自动扩展将无法正常工作。

基于 CPU 使用率进行扩展是 Pod 自动缩放的最常见用例。您还可以基于内存使用情况进行扩展。基于 CPU 的自动缩放对于根据请求消耗 CPU 的请求式系统最为有用，而内存消耗相对静态的系统则不然。

要扩展 ReplicaSet，可以运行以下命令：

```
$ kubectl autoscale rs kuard --min=2 --max=5 --cpu-percent=80
```

该命令创建一个自动缩放器，其在 CPU 利用率达到 80% 时在两个到五个副本之间进行缩放。要查看、修改或删除此资源，可以使用标准的 `kubectl` 命令和 `horizontalpodautoscalers` 资源。输入 `horizontalpodautoscalers` 这个词相当长，但可以缩写为 `hpa`：

```
$ kubectl get hpa
```

###### 警告

由于 Kubernetes 的解耦特性，HPA 与 ReplicaSet 之间没有直接的链接。尽管这对于模块化和组合是很好的，但也会导致一些反模式。特别是，在命令式或声明式管理副本数量的同时与自动缩放器结合使用是一个坏主意。如果您和自动缩放器同时尝试修改副本数量，很可能会发生冲突，导致意外行为。

# 删除 ReplicaSets

当不再需要 ReplicaSet 时，可以使用`kubectl delete`命令删除它。默认情况下，这也会删除由 ReplicaSet 管理的 Pods：

```
$ kubectl delete rs kuard
replicaset "kuard" deleted
```

运行`kubectl get pods`命令显示，所有由`kuard` ReplicaSet 创建的`kuard` Pods 也已被删除：

```
$ kubectl get pods
```

如果不想删除 ReplicaSet 管理的 Pods，可以将`--cascade`标志设置为`false`，以确保仅删除 ReplicaSet 对象而不是 Pods：

```
$ kubectl delete rs kuard --cascade=false
```

# 摘要

使用 ReplicaSet 组合 Pods 为构建具有自动故障转移功能的健壮应用程序提供了基础，并通过启用可扩展和合理的部署模式使部署这些应用程序变得轻松。无论是单个 Pod，都可以使用 ReplicaSets 进行关注。有些人甚至默认使用 ReplicaSets 而不是 Pods。典型的集群将拥有许多 ReplicaSets，因此在受影响的区域大量应用它们。
