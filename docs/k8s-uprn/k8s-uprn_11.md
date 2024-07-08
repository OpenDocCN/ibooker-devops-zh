# 第十一章：DaemonSets

部署（Deployments）和 ReplicaSet 通常用于创建具有多个副本以提高冗余性的服务（例如 Web 服务器）。但复制 Pod 集群的另一个原因是在集群内的每个节点上调度单个 Pod。一般来说，将 Pod 复制到每个节点的动机是为了在每个节点上部署某种代理或守护程序，而 Kubernetes 中实现这一目标的对象就是 DaemonSet。

DaemonSet 确保在 Kubernetes 集群的一组节点上运行 Pod 的副本。DaemonSet 用于部署系统守护程序，如日志收集器和监控代理，这些程序通常需要在每个节点上运行。DaemonSet 与 ReplicaSet 具有类似的功能；它们都创建预期长时间运行的 Pod，并确保集群的期望状态和观察状态匹配。

鉴于 DaemonSet 和 ReplicaSet 的相似性，了解何时使用其中之一是很重要的。当您的应用程序与节点完全解耦并且可以在给定节点上运行多个副本而无需特别考虑时，应使用 ReplicaSet。当必须在集群中的所有或一部分节点上运行应用程序的单个副本时，应使用 DaemonSet。

通常不应使用调度限制或其他参数来确保 Pod 不共同存在于同一节点上。如果发现自己希望每个节点只有一个 Pod，则 DaemonSet 是正确的 Kubernetes 资源。同样，如果发现自己正在构建一个用于服务用户流量的同质化复制服务，则 ReplicaSet 可能是正确的 Kubernetes 资源。

您可以使用标签在特定节点上运行 DaemonSet Pod；例如，您可能希望在暴露给边缘网络的节点上运行专门的入侵检测软件。

您还可以使用 DaemonSet 在基于云的集群中的节点上安装软件。对于许多云服务来说，升级或扩展集群可能会删除和/或重新创建新的虚拟机。这种动态的*不可变基础设施*方法可能会在您希望（或被中央 IT 要求）在每个节点上具有特定软件时造成问题。为了确保尽管升级和扩展事件，每台机器上都安装了特定软件，DaemonSet 是正确的方法。您甚至可以挂载主机文件系统并运行安装 RPM/DEB 包的脚本到主机操作系统上。通过这种方式，您可以拥有一个符合企业 IT 部门要求的云原生集群。

# DaemonSet 调度器

默认情况下，DaemonSet 会在每个节点上创建一个 Pod 的副本，除非使用了节点选择器，该选择器将限制符合一组标签的节点为符合条件的节点。DaemonSet 在 Pod 创建时通过在 Pod 规范中指定 `nodeName` 字段来确定 Pod 将在哪个节点上运行。因此，DaemonSets 创建的 Pods 将被 Kubernetes 调度程序忽略。

像 ReplicaSets 一样，DaemonSets 由一个协调控制循环管理，该循环通过测量期望状态（所有节点上都存在一个 Pod）和观察状态（特定节点上是否存在该 Pod？）来工作。根据这些信息，DaemonSet 控制器在每个当前没有匹配 Pod 的节点上创建一个 Pod。

如果向集群添加了新节点，则 DaemonSet 控制器会注意到缺少一个 Pod 并将该 Pod 添加到新节点上。

###### 注意

DaemonSets 和 ReplicaSets 很好地展示了解耦架构的价值。也许看起来正确的设计是 ReplicaSet 拥有它管理的 Pods，并且 Pods 是 ReplicaSet 的子资源。同样，由 DaemonSet 管理的 Pods 将是该 DaemonSet 的子资源。然而，这种封装会要求为处理 Pods 编写两次工具：一次为 DaemonSets，一次为 ReplicaSets。相反，Kubernetes 使用了一种解耦的方法，其中 Pods 是顶级对象。这意味着您在 ReplicaSets 上下文中学习的用于检查 Pods 的每个工具（例如 `kubectl logs <*pod-name*>`）同样适用于 DaemonSets 创建的 Pods。

# 创建 DaemonSets

DaemonSets 是通过向 Kubernetes API 服务器提交 DaemonSet 配置来创建的。示例 11-1 中的 DaemonSet 将在目标集群的每个节点上创建一个 `fluentd` 日志代理。

##### 示例 11-1\. fluentd.yaml

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  labels:
    app: fluentd
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd:v0.14.10
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

在给定 Kubernetes 命名空间中，DaemonSets 需要唯一的名称。每个 DaemonSet 必须包含一个 Pod 模板规范，该规范将用于根据需要创建 Pods。这是 ReplicaSets 和 DaemonSets 相似性的结束之处。与 ReplicaSets 不同，DaemonSets 默认会在集群中的每个节点上创建 Pods，除非使用节点选择器。

一旦您有了有效的 DaemonSet 配置，您可以使用 `kubectl apply` 命令将 DaemonSet 提交给 Kubernetes API。在本节中，我们将创建一个 DaemonSet 来确保 `fluentd` HTTP 服务器在我们集群中的每个节点上运行：

```
$ kubectl apply -f fluentd.yaml
daemonset.apps/fluentd created
```

一旦成功将 `fluentd` DaemonSet 提交到 Kubernetes API，您可以使用 `kubectl describe` 命令查询其当前状态：

```
$ kubectl describe daemonset fluentd
Name:           fluentd
Selector:       app=fluentd
Node-Selector:  <none>
Labels:         app=fluentd
Annotations:    deprecated.daemonset.template.generation: 1
Desired Number of Nodes Scheduled: 3
Current Number of Nodes Scheduled: 3
Number of Nodes Scheduled with Up-to-date Pods: 3
Number of Nodes Scheduled with Available Pods: 3
Number of Nodes Misscheduled: 0
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
...
```

此输出表明 `fluentd` Pod 已成功部署到我们集群中的所有三个节点。我们可以使用 `kubectl get pods` 命令，并使用 `-o` 标志打印每个 `fluentd` Pod 分配到的节点来验证这一点：

```
$ kubectl get pods -l app=fluentd -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP             NODE
fluentd-1q6c6   1/1     Running   0          13m   10.240.0.101   k0-default...
fluentd-mwi7h   1/1     Running   0          13m   10.240.0.80    k0-default...
fluentd-zr6l7   1/1     Running   0          13m   10.240.0.44    k0-default...
```

有了 `fluentd` DaemonSet 后，在集群中添加新节点将自动部署一个 `fluentd` Pod 到该节点：

```
$ kubectl get pods -l app=fluentd -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP             NODE
fluentd-1q6c6   1/1     Running   0          13m   10.240.0.101   k0-default...
fluentd-mwi7h   1/1     Running   0          13m   10.240.0.80    k0-default...
fluentd-oipmq   1/1     Running   0          43s   10.240.0.96    k0-default...
fluentd-zr6l7   1/1     Running   0          13m   10.240.0.44    k0-default...
```

这正是管理日志守护程序和其他整个集群服务时所需的行为。我们不需要采取任何行动；这就是 Kubernetes DaemonSet 控制器将其观察到的状态与我们期望的状态进行对比的方式。

# 限制 DaemonSets 只能部署到特定节点

DaemonSets 最常见的用例是在 Kubernetes 集群中的每个节点上运行一个 Pod。但是，也有一些情况下，您只希望将 Pod 部署到节点的子集中。例如，可能有一些工作负载需要 GPU 或仅在集群中特定节点上可用的快速存储访问。在这些情况下，可以使用节点标签来标记满足工作负载需求的特定节点。

## 向节点添加标签

限制 DaemonSet 只能部署到特定节点的第一步是在节点子集上添加所需的标签集。可以使用 `kubectl label` 命令实现此操作。

以下命令向单个节点添加 `ssd=true` 标签：

```
$ kubectl label nodes k0-default-pool-35609c18-z7tb ssd=true
node/k0-default-pool-35609c18-z7tb labeled
```

与其他 Kubernetes 资源一样，如果没有标签选择器，则列出没有标签选择器的节点将返回集群中的所有节点：

```
$ kubectl get nodes
NAME                            STATUS   ROLES    AGE   VERSION
k0-default-pool-35609c18-0xnl   Ready    agent    23m   v1.21.1
k0-default-pool-35609c18-pol3   Ready    agent    1d    v1.21.1
k0-default-pool-35609c18-ydae   Ready    agent    1d    v1.21.1
k0-default-pool-35609c18-z7tb   Ready    agent    1d    v1.21.1
```

使用标签选择器，我们可以根据标签筛选节点。要仅列出具有设置为 `true` 的 `ssd` 标签的节点，请使用 `kubectl get nodes` 命令并带有 `--selector` 标志：

```
$ kubectl get nodes --selector ssd=true
NAME                            STATUS   ROLES   AGE   VERSION
k0-default-pool-35609c18-z7tb   Ready    agent   1d    v1.21.1
```

## 节点选择器

在给定的 Kubernetes 集群中创建 DaemonSet 时，可以将节点选择器作为 Pod 规范的一部分来限制 Pod 可以运行的节点。示例中的 DaemonSet 配置示例 11-2 限制 NGINX 仅在带有 `ssd=true` 标签设置的节点上运行。

##### 示例 11-2\. nginx-fast-storage.yaml

```
apiVersion: apps/v1
kind: "DaemonSet"
metadata:
  labels:
    app: nginx
    ssd: "true"
  name: nginx-fast-storage
spec:
  selector:
    matchLabels:
      app: nginx
      ssd: "true"
  template:
    metadata:
      labels:
        app: nginx
        ssd: "true"
    spec:
      nodeSelector:
        ssd: "true"
      containers:
        - name: nginx
          image: nginx:1.10.0
```

让我们看看当我们向 Kubernetes API 提交 `nginx-fast-storage` DaemonSet 时会发生什么：

```
$ kubectl apply -f nginx-fast-storage.yaml
daemonset.apps/nginx-fast-storage created
```

由于只有一个带有 `ssd=true` 标签的节点，`nginx-fast-storage` Pod 将仅在该节点上运行：

```
$ kubectl get pods -l app=nginx -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP            NODE
nginx-fast-storage-7b90t   1/1     Running   0          44s   10.240.0.48   ...
```

将 `ssd=true` 标签添加到其他节点将导致 `nginx-fast-storage` Pod 部署在这些节点上。反之亦然：如果从节点中删除了所需的标签，则 DaemonSet 控制器将删除由该 DaemonSet 管理的 Pod。

###### 警告

从具有 DaemonSet 节点选择器所需标签的节点上删除标签将导致该 DaemonSet 管理的 Pod 从节点中删除。

# 更新 DaemonSet

DaemonSets 适用于在整个集群中部署服务，但是如何进行升级呢？在 Kubernetes 1.6 之前，更新由 DaemonSet 管理的 Pod 的唯一方法是更新 DaemonSet，然后手动删除每个由 DaemonSet 管理的 Pod，以便使用新配置重新创建它们。随着 Kubernetes 1.6 的发布，DaemonSets 获得了类似于 Deployment 对象的功能，在集群内管理 ReplicaSet 的滚动更新。

可以使用与 Deployments 相同的 `RollingUpdate` 策略推出 DaemonSets。可以使用 `spec.update​Strat⁠egy.type` 字段配置更新策略，该字段的值应为 `RollingUpdate`。当 DaemonSet 具有 `RollingUpdate` 的更新策略时，对 DaemonSet 中的 `spec.template` 字段（或子字段）进行任何更改将启动滚动更新。

与 Deployments 的滚动更新类似（参见 第十章），`RollingUpdate` 策略逐步更新 DaemonSet 的成员，直到所有 Pod 都在新配置下运行。有两个参数控制 DaemonSet 的滚动更新：

`spec.minReadySeconds`

确定 Pod 必须“准备就绪”多长时间，然后滚动更新才能继续升级后续 Pod

`spec.updateStrategy.rollingUpdate.maxUnavailable`

指示可以同时更新多少个 Pod 的滚动更新

通常应设置 `spec.minReadySeconds` 为一个合理较长的值，例如 30-60 秒，以确保 Pod 在继续推出之前确实健康。

设置 `spec.updateStrategy.rollingUpdate.maxUnavailable` 更可能依赖于应用程序。将其设置为 `1` 是一种安全的通用策略，但完全推出可能需要一段时间（节点数 × `minReady​Sec⁠onds`）。增加最大不可用性将加快推出速度，但会增加失败推出的“影响范围”。应用程序和集群环境的特性决定了速度与安全性之间的相对值。一个好的方法可能是将 `maxUnavailable` 设置为 `1`，并仅在用户或管理员投诉 DaemonSet 推出速度时才增加它。

一旦开始滚动更新，您可以使用 `kubectl rollout` 命令查看 DaemonSet 推出的当前状态。例如，`kubectl rollout status daemonSets my-daemon-set` 将显示名为 `my-daemon-set` 的 DaemonSet 的当前推出状态。

# 删除 DaemonSet

使用 `kubectl delete` 命令删除 DaemonSet 非常简单。只需确保提供要删除的 DaemonSet 的正确名称：

```
$ kubectl delete -f fluentd.yaml
```

###### 警告

删除 DaemonSet 还将删除由该 DaemonSet 管理的所有 Pod。设置 `--cascade` 标志为 `false`，以确保仅删除 DaemonSet 而不是 Pod。

# 总结

DaemonSets 提供了在 Kubernetes 集群中每个节点上运行一组 Pod 的易用抽象，或者根据标签在节点子集上运行。DaemonSet 提供了自己的控制器和调度器，以确保关键服务如监控代理始终在集群中的正确节点上运行。

对于某些应用程序，你只需安排一定数量的副本；只要它们具有足够的资源和分布以可靠地运行，你并不关心它们运行在哪里。然而，有一类不同的应用程序，如代理和监控应用程序，需要在集群中的每台机器上都存在才能正常工作。这些守护集并不是传统的服务应用程序，而是为 Kubernetes 集群本身添加额外功能和特性。因为守护集是由控制器管理的主动声明对象，因此很容易声明你希望代理在每台机器上运行，而无需显式地将其放置在每台机器上。在自动缩放的 Kubernetes 集群中尤其有用，因为节点可能会不断地添加和移除而无需用户干预。在这种情况下，守护集会在自动缩放器将节点添加到集群时自动向每个节点添加适当的代理。
