# 第九章：**Daemon Service**

*Daemon Service* 模式允许您在目标节点上放置和运行优先级高、基础设施相关的 Pods。主要由管理员使用，以增强 Kubernetes 平台的能力。

# 问题

在软件系统中，**daemon** 的概念存在于多个级别。在操作系统级别，*daemon* 是一个长时间运行的、自我恢复的计算机程序，作为后台进程运行。在 Unix 系统中，daemon 的名称以 *d* 结尾，如 httpd、named 和 sshd。在其他操作系统中，使用替代术语如 *services-started tasks* 和 *ghost jobs*。

不管这些程序被称为什么，它们的共同特征是作为进程运行，并且通常不与监视器、键盘和鼠标交互，并在系统启动时启动。在应用程序级别也存在类似的概念。例如，在 Java 虚拟机中，daemon 线程在后台运行，并为用户线程提供支持服务。这些 daemon 线程优先级低，后台运行，并且在应用程序生命周期中没有参与，并执行垃圾回收或 finalization 等任务。

同样地，Kubernetes 也有 **DaemonSet** 的概念。考虑到 Kubernetes 是一个跨多个节点分布的分布式平台，并且其主要目标是管理应用程序 Pods，**DaemonSet** 由在集群节点上运行的 Pods 表示，并为集群的其余部分提供一些后台功能。

# 解决方案

**ReplicaSet** 及其前身 **ReplicationController** 是控制结构，负责确保特定数量的 Pods 正在运行。这些控制器不断监视运行中的 Pods 列表，并确保实际的 Pods 数量始终与期望的数量匹配。在这方面，**DaemonSet** 是一个类似的结构，负责确保一定数量的 Pods 始终在运行。不同之处在于前两者运行特定数量的 Pods，通常由应用程序要求驱动，用于高可用性和用户负载，而不考虑节点数。

另一方面，**DaemonSet** 不受消费者负载的驱动，来决定运行多少个 Pod 实例以及在哪里运行。它的主要目的是在每个节点或特定节点上保持运行一个单独的 Pod。接下来我们看一个如下所示的 Example 9-1 中的 DaemonSet 定义。

##### Example 9-1\. **DaemonSet** 资源

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: random-refresher
spec:
  selector:
    matchLabels:
      app: random-refresher
  template:
    metadata:
      labels:
        app: random-refresher
    spec:
      nodeSelector:            ![1](img/1.png)
        feature: hw-rng
      containers:
      - image: k8spatterns/random-generator:1.0
        name: random-generator
        command: [ "java", "RandomRunner", "/numbers.txt", "10000", "30" ]
        volumeMounts:          ![2](img/2.png)
        - mountPath: /host_dev
          name: devices
      volumes:
      - name: devices
        hostPath:              ![3](img/3.png)
          path: /dev
```

![1](img/#co_daemon_service_CO1-1)

仅使用标签 `feature` 设置为值 `hw-rng` 的节点。

![2](img/#co_daemon_service_CO1-2)

**DaemonSets** 通常会挂载节点文件系统的一部分，以执行维护操作。

![3](img/#co_daemon_service_CO1-3)

用于直接访问节点目录的 `hostPath`。

鉴于这种行为，守护进程集的主要候选对象通常是基础架构相关的进程，例如集群存储提供者、日志收集器、度量导出器，甚至 kube-proxy，这些进程执行集群范围的操作。守护进程集和副本集管理方式有很多不同之处，但主要的区别如下：

+   默认情况下，守护进程集在每个节点上放置一个 Pod 实例。可以通过使用`nodeSelector`或`affinity`字段来控制和限制放置在节点子集上的 Pod。

+   守护进程集创建的 Pod 已经指定了`nodeName`。因此，守护进程集无需存在 Kubernetes 调度器来运行容器。这也允许您使用守护进程集来运行和管理 Kubernetes 组件。

+   守护进程集创建的 Pod 可以在调度器启动之前运行，这使它们能够在任何其他 Pod 被放置到节点上之前运行。

+   由于调度器未被使用，节点的`unschedulable`字段不会被守护进程集控制器所尊重。

+   由守护进程集创建的 Pod 的`RestartPolicy`只能设置为`Always`或者不指定，默认为`Always`。这是为了确保当存活探针失败时，容器将被终止并始终重新启动。

+   守护进程集管理的 Pod 只能在目标节点上运行，因此在许多控制器中被视为具有较高优先级。例如，去调度器将避免驱逐这些 Pod，集群自动伸缩器将单独管理它们，等等。

守护进程集的主要用例是在集群中某些节点上运行系统关键的 Pod。守护进程集控制器通过直接将 Pod 分配给节点的方式，通过设置 Pod 规范的`nodeName`字段，确保所有符合条件的节点运行 Pod 的副本。这允许守护进程集的 Pod 在默认调度器启动之前就能够被调度，并且使其免受用户配置的任何调度器自定义的影响。只要节点上有足够的资源并且在放置其他 Pod 之前完成了，这种方法就能够正常工作。当节点资源不足时，守护进程集控制器无法为该节点创建 Pod，并且无法进行任何释放节点资源的操作，如抢占。守护进程集控制器和调度器中调度逻辑的重复创建了维护挑战。守护进程集的实现也无法从新调度器的新特性（如亲和性、反亲和性和抢占）中获益。因此，从 Kubernetes v1.17 及更新版本开始，守护进程集通过设置`nodeAffinity`字段而非`nodeName`字段来为守护进程集的 Pod 进行调度，使用默认调度器进行调度成为运行守护进程集的必备依赖项，但同时也将污点、容忍性、Pod 优先级和抢占引入守护进程集，并在资源匮乏时改善在所需节点上运行守护进程集 Pod 的整体体验。

通常，DaemonSet 在每个节点或节点子集上创建一个单独的 Pod。基于此，有几种方法可以访问由 DaemonSet 管理的 Pods：

服务

创建一个带有与 DaemonSet 相同的 Pod 选择器的服务，并使用该服务来访问通过随机节点进行负载均衡的守护 Pod。

DNS

创建一个无头服务，其 Pod 选择器与 DaemonSet 相同，可用于从 DNS 中检索包含所有 Pod IP 和端口的多个 A 记录。

使用 `hostPort` 的节点 IP

DaemonSet 中的 Pods 可以指定 `hostPort` 并通过节点 IP 地址和指定的端口可达。由于节点 IP、`hostPort` 和 `protocol` 的组合必须是唯一的，可以安排 Pod 的位置有限。

此外，DaemonSets Pods 中的应用可以将数据推送到已知位置或外部服务，不需要消费者直接访问 DaemonSets Pods。

静态 Pods 可用于启动 Kubernetes 系统进程或其他容器的容器化版本。然而，与静态 Pods 相比，DaemonSets 与平台的其余部分更好地集成，并推荐使用它们。

# 讨论

使用其他方法在每个节点上运行守护进程的方式，但它们都有局限性。静态 Pod 由 Kubelet 管理，但无法通过 Kubernetes API 进行管理。裸 Pod（没有控制器的 Pod）如果意外删除或终止，将无法生存，也无法在节点故障或节点维护中生存。诸如 upstartd 或 systemd 的初始化脚本需要不同的工具链进行监控和管理，并且无法从用于应用工作负载的 Kubernetes 工具中获益。所有这些都使得 Kubernetes 和 DaemonSet 成为运行守护进程过程的一个吸引人选项。

在本书中，我们描述的模式和 Kubernetes 特性主要由开发人员使用，而不是平台管理员。DaemonSet 处于中间位置，更倾向于管理员工具箱，但我们在这里包括它，因为它也与应用开发人员相关。DaemonSets 和 CronJobs 也是 Kubernetes 将单节点概念（如 crontab 和守护脚本）转换为用于管理分布式系统的多节点集群基元的完美示例。这些是开发人员必须熟悉的新分布式概念。

# 更多信息

+   [守护进程示例服务](https://oreil.ly/_YRZc)

+   [DaemonSet](https://oreil.ly/62c3q)

+   [对 DaemonSet 执行滚动更新](https://oreil.ly/nTSbi)

+   [DaemonSets 和 Jobs](https://oreil.ly/CnHin)

+   [创建静态 Pod](https://oreil.ly/yYHft)
