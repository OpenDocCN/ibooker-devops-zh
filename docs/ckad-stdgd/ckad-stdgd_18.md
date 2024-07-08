# 第十八章：资源需求、限制和配额

在 Pod 中执行的工作负载将消耗一定量的资源（例如 CPU 和内存）。您应该为这些应用程序定义资源需求。在容器级别，可以定义运行应用程序所需的最小资源量，以及应用程序允许消耗的最大资源量。应用程序开发人员应通过负载测试或在运行时通过监控资源消耗来确定合适的资源大小。

###### 注释

Kubernetes 使用毫核 CPU 和字节内存来衡量资源。这就是为什么您可能会看到资源定义为 600m 或 100Mi。深入了解这些资源单位，值得参考官方文档中关于 [“Kubernetes 中的资源单位”](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-units-in-kubernetes) 的部分。

Kubernetes 管理员可以采取措施来强制使用可用资源容量。我们将讨论两个 Kubernetes 原语，ResourceQuota 和 LimitRange。ResourceQuota 在命名空间级别定义聚合资源约束。LimitRange 是一种策略，用于约束或为特定类型的单个对象（如 Pod 或 PersistentVolumeClaim）的资源分配设置默认值。

# 处理资源需求

推荐的最佳实践是为每个容器指定资源请求和限制。确定这些资源期望并不总是容易，特别是对于尚未在生产环境中运行的应用程序。在开发周期早期对应用程序进行负载测试可以帮助分析资源需求。将应用程序部署到集群后，可以通过监控应用程序的资源消耗进一步进行调整。

## 定义容器资源请求

用于工作负载调度的一个指标是 Pod 中容器定义的 *资源请求*。常用的可指定资源包括 CPU 和内存。调度器确保节点的资源容量能够满足 Pod 的资源需求。具体而言，调度器确定 Pod 中所有容器定义的每种资源类型的资源请求总和，并将其与节点的可用资源进行比较。

每个 Pod 中的每个容器都可以定义自己的资源请求。表 18-1 描述了可用的选项，包括一个示例值。

表 18-1\. 资源请求选项

| YAML 属性 | 描述 | 示例值 |
| --- | --- | --- |
| `spec.containers[].resources.requests.cpu` | CPU 资源类型 | `500m`（五百毫 CPU） |
| `spec.containers[].resources.requests.memory` | 内存资源类型 | `64Mi`（2²⁶ 字节） |
| `spec.containers[].resources.requests.hugepages-<size>` | 大页资源类型 | `hugepages-2Mi: 60Mi` |
| `spec.containers[].resources.requests.ephemeral-storage` | 临时存储资源类型 | `4Gi` |

为了澄清这些资源请求的用途，我们将看一个示例定义。在 示例 18-1 中显示的 Pod YAML 清单定义了两个容器，每个容器都有自己的资源请求。允许运行该 Pod 的任何节点需要能够支持至少 320Mi 的内存容量和 1250m 的 CPU，这是两个容器资源的总和。

##### 示例 18-1\. 设置容器资源请求

```
apiVersion: v1
kind: Pod
metadata:
  name: rate-limiter
spec:
  containers:
  - name: business-app
    image: bmuschko/nodejs-business-app:1.0.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "256Mi"
        cpu: "1"
  - name: ambassador
    image: bmuschko/nodejs-ambassador:1.0.0
    ports:
    - containerPort: 8081
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
```

当节点上的资源不足以安排 Pod 时，Pod 的事件日志将使用 `PodExceedsFreeCPU` 或 `PodExceedsFreeMemory` 的原因指示此情况。有关如何排查和解决此情况的详细信息，请参阅相关 [文档部分](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#troubleshooting)。

## 定义容器资源限制

另一个您可以为容器设置的度量是 *资源限制*。资源限制确保容器不能消耗超过分配的资源量。例如，您可以表达在容器中运行的应用程序应受到 1000m CPU 和 512Mi 内存的限制。

根据集群使用的容器运行时，超过允许的任何资源限制都会导致终止在容器中运行的应用程序进程或导致系统阻止超出限制的资源分配。有关容器运行时 Docker Engine 如何处理资源限制的深入讨论，请参阅 [文档](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#how-pods-with-resource-limits-are-run)。

表 18-2 描述了可用选项，包括示例值。

表 18-2\. 资源限制选项

| YAML 属性 | 描述 | 示例值 |
| --- | --- | --- |
| `spec.containers[].resources.limits.cpu` | CPU 资源类型 | `500m` (500 毫核) |
| `spec.containers[].resources.limits.memory` | 内存资源类型 | `64Mi` (2²⁶ 字节) |
| `spec.containers[].resources.limits.hugepages-<size>` | 大页资源类型 | `hugepages-2Mi: 60Mi` |
| `spec.containers[].resources.limits.ephemeral-storage` | 临时存储资源类型 | `4Gi` |

示例 18-2 展示了限制定义的实际操作。在这里，名为 `business-app` 的容器不能使用超过 512Mi 的内存。名为 `ambassador` 的容器定义了 128Mi 内存的限制。

##### 示例 18-2\. 设置容器资源限制

```
apiVersion: v1
kind: Pod
metadata:
  name: rate-limiter
spec:
  containers:
  - name: business-app
    image: bmuschko/nodejs-business-app:1.0.0
    ports:
    - containerPort: 8080
    resources:
      limits:
        memory: "256Mi"
  - name: ambassador
    image: bmuschko/nodejs-ambassador:1.0.0
    ports:
    - containerPort: 8081
    resources:
      limits:
        memory: "64Mi"
```

## 定义容器资源请求和限制

要向 Kubernetes 提供你的应用程序资源期望的全貌，你必须为每个容器指定资源请求和限制。示例 18-3 将资源请求和限制结合在单个 YAML 清单中。

##### 示例 18-3\. 设置容器资源请求和限制

```
apiVersion: v1
kind: Pod
metadata:
  name: rate-limiter
spec:
  containers:
  - name: business-app
    image: bmuschko/nodejs-business-app:1.0.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "256Mi"
        cpu: "1"
      limits:
        memory: "256Mi"
  - name: ambassador
    image: bmuschko/nodejs-ambassador:1.0.0
    ports:
    - containerPort: 8081
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "64Mi"
```

分配静态容器资源需求是一个近似的过程。你希望在你的 Kubernetes 集群中最大化资源的有效利用。不幸的是，Kubernetes 文档并没有提供很多关于最佳实践的指导。博客文章 [“**停止在 Kubernetes 上使用 CPU 限制**”](https://home.robusta.dev/blog/stop-using-cpu-limits) 提供了以下建议：

+   总是定义内存请求。

+   总是定义内存限制。

+   总是将内存请求设置为与限制相等。

+   总是定义 CPU 请求。

+   不要使用 CPU 限制。

在将你的应用程序投入生产后，仍然需要监控应用程序的资源消耗模式。在运行时审查资源消耗，并跟踪实际的调度行为以及应用程序接收负载后可能出现的不良行为。找到一个合适的平衡可能会令人沮丧。类似 [Goldilocks](https://www.fairwinds.com/blog/introducing-goldilocks-a-tool-for-recommending-resource-requests) 和 [KRR](https://github.com/robusta-dev/krr) 的项目出现，提供了关于适当确定资源请求的建议和指导。其他选择，如 Kubernetes 1.27 中引入的 [容器调整策略](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/#container-resize-policies)，允许更精细地控制容器的 CPU 和内存资源在运行时如何自动调整大小。

# 使用 Resource Quotas

Kubernetes 的原始 ResourceQuota 定义了每个命名空间可用的资源的最大数量。一旦生效，Kubernetes 调度器将负责强制执行这些规则。下面的列表应该让你对可以定义的规则有一个了解：

+   设置特定类型可以创建的对象数量的上限（例如，最多三个 Pod）。

+   限制计算资源的总和（例如，3Gi RAM）。

+   期望 Pod 的服务质量（QoS）类别（例如，`BestEffort` 表示 Pod 不应设置任何内存或 CPU 的限制或请求）。

## 创建 ResourceQuotas

创建 ResourceQuota 对象通常是 Kubernetes 管理员的任务，但定义和创建这样的对象相对简单。首先，创建应用此配额的命名空间：

```
$ kubectl create namespace team-awesome
namespace/team-awesome created

```

接下来，在 YAML 中定义 ResourceQuota。为了展示 ResourceQuota 的功能性，向命名空间添加约束，如 示例 18-4 所示。

##### 示例 18-4\. 使用 ResourceQuota 定义硬资源限制

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: awesome-quota
  namespace: team-awesome
spec:
  hard:
    pods: 2                   ![1](img/1.png)
    requests.cpu: "1"         ![2](img/2.png)
    requests.memory: 1024Mi   ![2](img/2.png)
    limits.cpu: "4"           ![3](img/3.png)
    limits.memory: 4096Mi     ![3](img/3.png)
```

![1](img/#co_resource_requirements__limits__and_quotas_CO1-1)

将 Pod 的数量限制为 2 个。

![2](img/#co_resource_requirements__limits__and_quotas_CO1-2)

定义处于非终端状态的所有 Pod 的最小资源请求为 1 个 CPU 和 1024Mi RAM。

![3](img/#co_resource_requirements__limits__and_quotas_CO1-4)

定义处于非终端状态的所有 Pod 使用的最大资源为 4 个 CPU 和 4096Mi RAM。

您已准备好为命名空间创建 ResourceQuota：

```
$ kubectl create -f awesome-quota.yaml
resourcequota/awesome-quota created

```

## 渲染 ResourceQuota 详情

你可以使用`kubectl describe`命令渲染已使用资源与硬限制的表格概述：

```
$ kubectl describe resourcequota awesome-quota -n team-awesome
Name:            awesome-quota
Namespace:       team-awesome
Resource         Used  Hard
--------         ----  ----
limits.cpu       0     4
limits.memory    0     4Gi
pods             0     2
requests.cpu     0     1
requests.memory  0     1Gi

```

在硬限制列出了与 ResourceQuota 定义提供的相同值。只要您不修改对象的规范，这些值不会改变。在已使用列下，您可以找到命名空间内的实际聚合资源消耗。目前，所有值均为 0，因为尚未创建任何 Pod。

## 探索 ResourceQuota 的运行时行为

在命名空间`team-awesome`中实施了配额规则后，我们希望看到其执行情况。我们将从创建超过最大 Pod 数目的实验开始，这个数目是两个。为了测试这一点，我们可以创建任何我们喜欢的定义 Pod。例如，我们可以使用一个最基本的定义，该定义在容器中运行图像`nginx:1.25.3`，如示例 18-5 所示。

##### 示例 18-5\. 没有资源需求的 Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: team-awesome
spec:
  containers:
  - image: nginx:1.25.3
    name: nginx
```

根据存储在 *nginx-pod.yaml* 中的 YAML 定义，让我们创建一个 Pod 并看看会发生什么。实际上，Kubernetes 将拒绝创建对象，并显示以下错误信息：

```
$ kubectl apply -f nginx-pod.yaml
Error from server (Forbidden): error when creating "nginx-pod.yaml": \
pods "nginx" is forbidden: failed quota: awesome-quota: must specify \
limits.cpu for: nginx; limits.memory for: nginx; requests.cpu for: \
nginx; requests.memory for: nginx

```

因为我们在命名空间中为对象定义了最小和最大资源配额，所以我们必须确保 Pod 对象实际上定义了资源请求和限制。通过更新`resources`指令来修改初始定义，如示例 18-6 所示。

##### 示例 18-6\. 具有资源需求的 Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: team-awesome
spec:
  containers:
  - image: nginx:1.25.3
    name: nginx
    resources:
      requests:
        cpu: "0.5"
        memory: "512Mi"
      limits:
        cpu: "1"
        memory: "1024Mi"
```

我们应该能够创建两个具有唯一名称的 Pod——`nginx1`和`nginx2`——使用该清单；合并后的资源需求仍符合 ResourceQuota 中定义的边界：

```
$ kubectl apply -f nginx-pod1.yaml
pod/nginx1 created
$ kubectl apply -f nginx-pod2.yaml
pod/nginx2 created
$ kubectl describe resourcequota awesome-quota -n team-awesome
Name:            awesome-quota
Namespace:       team-awesome
Resource         Used  Hard
--------         ----  ----
limits.cpu       2     4
limits.memory    2Gi   4Gi
pods             2     2
requests.cpu     1     1
requests.memory  1Gi   1Gi

```

假设我们尝试使用`nginx1`和`nginx2`的定义创建另一个 Pod，你或许能够想象出会发生什么。它会因为两个原因而失败。第一个原因是我们不被允许在命名空间中创建第三个 Pod，因为最大数目被设置为两个。第二个原因是我们将超出`requests.cpu`和`requests.memory`的最大分配限制。以下错误信息提供了这些信息：

```
$ kubectl apply -f nginx-pod3.yaml
Error from server (Forbidden): error when creating "nginx-pod3.yaml": \
pods "nginx3" is forbidden: exceeded quota: awesome-quota, requested: \
pods=1,requests.cpu=500m,requests.memory=512Mi, used: pods=2,requests.cpu=1,\
requests.memory=1Gi, limited: pods=2,requests.cpu=1,requests.memory=1Gi

```

# 使用限制范围

在前一节中，您了解了资源配额如何限制特定命名空间中资源的总体消耗。对于单个 Pod 对象，资源配额无法设置任何约束。这就是 LimitRange 的作用。在处理 API 请求时，LimitRange 规则的执行发生在准入控制阶段。

LimitRange 是 Kubernetes 的一个原语，用于约束或默认特定对象类型的资源分配：

+   在命名空间中强制 Pod 或容器的最小和最大计算资源使用

+   在命名空间中强制每个 PersistentVolumeClaim 的最小和最大存储请求

+   强制在命名空间中资源请求和限制之间设置比率

+   在命名空间中为计算资源设置默认请求/限制，并在运行时自动注入到容器中

# 在命名空间中定义多个 LimitRange

最好每个命名空间只创建一个 LimitRange 对象。在同一命名空间中由多个 LimitRange 对象指定的默认资源请求和限制会导致这些规则的非确定性选择。只有一个默认定义会生效，但无法预测哪一个会生效。

## 创建 LimitRange

LimitRange 提供了一系列可配置的约束属性列表。所有这些属性在 Kubernetes API 文档中的[LimitRangeSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#limitrangeitem-v1-core)中都有详细描述。示例 18-7 展示了使用某些约束属性的 LimitRange 的 YAML 清单。

##### 示例 18-7。定义多个约束条件的 LimitRange

```
apiVersion: v1
kind: LimitRange
metadata:
  name: cpu-resource-constraint
spec:
  limits:
  - type: Container   ![1](img/1.png)
    defaultRequest:   ![2](img/2.png)
      cpu: 200m
    default:          ![3](img/3.png)
      cpu: 200m
    min:              ![4](img/4.png)
      cpu: 100m
    max:              ![4](img/4.png)
      cpu: "2"
```

![1](img/#co_resource_requirements__limits__and_quotas_CO2-1)

应用约束的上下文。在这种情况下，应用到运行在 Pod 中的容器。

![2](img/#co_resource_requirements__limits__and_quotas_CO2-2)

如果未提供，分配给容器的默认 CPU 资源请求值。

![3](img/#co_resource_requirements__limits__and_quotas_CO2-3)

如果未提供，分配给容器的默认 CPU 资源限制值。

![4](img/#co_resource_requirements__limits__and_quotas_CO2-4)

可分配给容器的最小和最大 CPU 资源请求和限制值。

通常情况下，我们可以使用`kubectl create`或`kubectl apply`命令从清单中创建对象。LimitRange 的定义已存储在文件*cpu-resource-constraint-limitrange.yaml*中：

```
$ kubectl apply -f cpu-resource-constraint.yaml
limitrange/cpu-resource-constraint created

```

当创建新对象时，约束将自动应用。更改现有 LimitRange 对象的约束对已运行的 Pods 没有影响。

## 渲染 LimitRange 详细信息

使用`kubectl describe`命令可以查看实时的 LimitRange 对象。以下命令呈现了名为`cpu-resource-constraint`的 LimitRange 对象的详细信息：

```
$ kubectl describe limitrange cpu-resource-constraint
Name:       cpu-resource-constraint
Namespace:  default
Type        Resource  Min   Max  Default Request  Default Limit   ...
----        --------  ---   ---  ---------------  -------------
Container   cpu       100m  2    200m             200m            ...

```

命令的输出将每个限制约束渲染为单独的行。任何对象未显式设置的约束属性将显示破折号 (`-`) 作为分配的值。

## 探索 LimitRange 的运行行为

让我们演示 LimitRange 对 Pod 创建的影响。我们将讨论两种不同的使用情况：

1.  如果 Pod 定义没有提供资源要求，则自动设置资源要求。

1.  如果声明的资源要求被 LimitRange 禁止，则阻止创建 Pod。

### 设置默认资源要求

LimitRange 定义了默认的 CPU 资源请求为 200m，CPU 资源限制为 200m。这意味着如果要创建一个 Pod，并且它没有定义 CPU 资源请求和限制，LimitRange 将自动分配默认值。

示例 18-8 展示了一个没有资源要求的 Pod 定义。

##### 示例 18-8\. 定义无资源要求的 Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-without-resource-requirements
spec:
  containers:
  - image: nginx:1.25.3
    name: nginx
```

根据存储在文件 *nginx-without-resource-requirements.yaml* 中的内容创建对象将按预期工作：

```
$ kubectl apply -f nginx-without-resource-requirements.yaml
pod/nginx-without-resource-requirements created

```

Pod 对象将以两种方式被修改。首先，LimitRange 设置的默认资源要求将被应用。其次，将添加一个键为 `kubernetes.io/limit-ranger` 的注释，提供关于已更改内容的元信息。您可以在 `describe` 命令的输出中找到这两个信息：

```
$ kubectl describe pod nginx-without-resource-requirements
...
Annotations:      kubernetes.io/limit-ranger: LimitRanger plugin set: cpu \
request for container nginx; cpu limit for container nginx
...
Containers:
  nginx:
    ...
    Limits:
      cpu: 200m
    Requests:
      cpu: 200m
...

```

### 强制资源要求

LimitRange 也可以强制资源限制。对于我们之前创建的 LimitRange 对象，CPU 的最小值设置为 100m，最大值设置为 2\. 为了查看执行行为，我们将创建一个新的 Pod，如 示例 18-9 中所示。

##### 示例 18-9\. 定义 CPU 资源请求和限制的 Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-resource-requirements
spec:
  containers:
  - image: nginx:1.25.3
    name: nginx
    resources:
      requests:
        cpu: "50m"
      limits:
        cpu: "3"
```

这个 Pod 的资源要求不符合 LimitRange 对象期望的约束。CPU 资源请求少于 100m，CPU 资源限制高于 2\. 因此，对象将不会被创建，并将显示适当的错误消息：

```
$ kubectl apply -f nginx-with-resource-requirements.yaml
Error from server (Forbidden): error when creating "nginx-with-resource-\
requirements.yaml": pods "nginx-with-resource-requirements" is forbidden: \
[minimum cpu usage per Container is 100 m, but request is 50 m, maximum cpu \
usage per Container is 2, but limit is 3]

```

错误消息提供了关于预期资源定义的一些指导。不幸的是，消息没有指出实施这些期望的 LimitRange 对象的名称。请主动检查命名空间中是否已创建 LimitRange 对象，并使用 `kubectl get limitranges` 命令查看设置的参数。

# 概要

资源请求是 kube-scheduler 算法在决定 Pod 可以被调度到哪个节点时考虑的众多因素之一。一个容器可以使用 `spec.containers[].resources.requests` 来指定请求。调度程序根据节点的可用硬件容量选择节点。资源限制确保容器不能消耗超过分配的资源量。可以使用属性 `spec.con⁠tainers[].resources.limits` 为容器定义限制。如果应用程序因为实现中的内存泄漏等原因消耗超过允许的资源量，容器运行时很可能会终止应用程序进程。

资源配额定义了一个命名空间中可用的计算资源（例如 CPU、RAM 和临时存储），以防止运行其中的 Pod 消耗无限资源。因此，Pod 必须在这些资源边界内工作，声明它们的最小和最大资源期望。您还可以限制允许创建的资源类型的数量（例如 Pods、Secrets 或 ConfigMaps）。Kubernetes 调度程序将在请求对象创建时强制执行这些边界。

限制范围不同于 ResourceQuota，它为特定类型的单个对象定义资源约束。它还可以通过指定资源默认值来帮助对象的治理，这些值应在 API 创建请求未提供信息时自动应用。

# 考试要点

体验资源需求对调度和自动扩展的影响

由 Pod 定义的容器可以指定资源请求和限制。通过定义单个和多个容器 Pod 的这些要求，解决场景。在创建 Pod 时，您应该能够看到将对象调度到节点的影响。此外，练习如何识别节点的可用资源容量。

理解资源配额的目的和运行时效果

ResourceQuota 定义了命名空间中对象的资源边界。最常用的边界适用于计算资源。练习定义它们并了解它们对 Pod 创建的影响。了解列出 ResourceQuota 的硬性要求和当前正在使用的资源的命令非常重要。您会发现 ResourceQuota 提供了其他选项。详细了解这些选项，以更广泛地了解这个主题。

理解限制范围的目的和运行时效果

LimitRange 可以指定特定基元的资源约束和默认值。如果在创建对象时遇到错误消息，请检查是否有限制范围对象强制执行这些约束。不幸的是，错误消息没有指出强制执行它的对象，因此您可能需要主动列出 LimitRange 对象以识别约束。

# 示例练习

这些练习的解决方案在附录 A 中可用。

1.  你被要求创建一个 Pod 以在容器中运行应用程序。在应用程序开发期间，你运行了负载测试，以确定所需的最小资源量和应用程序允许增长到的最大资源量。定义该 Pod 的资源请求和限制。

    定义一个名为`hello-world`的 Pod，运行容器镜像`bmuschko/nodejs-hello-world:1.0.0`。容器公开端口 3000。

    添加一个类型为`emptyDir`的卷，并将其挂载到容器路径`/var/log`。

    对于容器，指定以下最小资源数量：

    +   CPU：100m

    +   内存：500Mi

    +   临时存储：1Gi

    对于容器，指定以下最大资源数量：

    +   内存：500Mi

    +   临时存储：2Gi

    从 YAML 清单创建 Pod。检查 Pod 的详细信息。Pod 运行在哪个节点上？

1.  在此练习中，你将为新名称空间创建一个具有特定 CPU 和内存限制的资源配额。在名称空间中创建的 Pod 将必须遵守这些限制。

    使用以下 YAML 定义在文件 *resourcequota.yaml* 中，在名称空间 `rq-demo` 下创建一个名为 `app` 的 ResourceQuota：

    ```
    apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: app
    spec:
      hard:
        pods: "2"
        requests.cpu: "2"
        requests.memory: 500Mi
    ```

    创建一个超出资源配额要求限制的新 Pod，例如定义了 1Gi 的内存但保持低于 CPU，例如 0.5。记录下错误消息。

    更改请求限制以满足要求，以确保成功创建 Pod。记录下命令的输出，显示名称空间中使用的资源量。

1.  LimitRange 可以限制名称空间中 Pod 的资源消耗，并在未定义资源需求时分配默认的计算资源。你将练习 LimitRange 在不同场景下对 Pod 创建的影响。

    转到 GitHub 仓库 *bmuschko/ckad-study-guide* 中的目录 *app-a/ch18/limitrange*。检查文件 *setup.yaml* 中 YAML 清单的定义。从 YAML 清单文件创建对象。

    在名称空间`d92`中创建一个名为`pod-without-resource-requirements`的新 Pod，使用容器镜像`nginx:1.23.4-alpine`，没有任何资源要求。检查 Pod 的详细信息。你期望分配哪些资源定义？

    在名称空间`d92`中创建一个名为`pod-with-more-cpu-resource-requirements`的新 Pod，使用容器镜像`nginx:1.23.4-alpine`，CPU 资源请求为 400m，限制为 1.5。你期望看到什么运行时行为？

    在名称空间`d92`中创建一个名为`pod-with-less-cpu-resource-requirements`的新 Pod，使用容器镜像`nginx:1.23.4-alpine`，CPU 资源请求为 350m，限制为 400m。你期望看到什么运行时行为？
