# 第三章：工作负载

当我们谈论[Kubernetes 中的工作负载](https://oreil.ly/Uz1gG)时，我们指的是运行应用程序的 API 资源类型。这些 API 资源类型包括 Deployment、ReplicaSet、StatefulSet、DaemonSet、Job、CronJob 和当然 Pod。CKA 的课程对你需要熟悉的工作负载类型非常具体。考试只包括 Deployment、ReplicaSet 和 Pod。您需要理解 Deployment 管理的复制和升级功能，并理解将配置数据注入 Pod 的 API 原语。

###### 注意

本章将使用体积的概念。如果您对 Kubernetes 的持久存储选项不熟悉，请查看第六章以获取更多信息。

在高层次上，本章涵盖以下概念：

+   对 Deployments 的基本理解

+   Deployment 的部署和回滚功能

+   由 ReplicaSet 控制的副本的手动和自动扩展

+   ConfigMap 和 Secret

# 使用 Deployments 管理工作负载

在 Kubernetes 中，工作负载在 Pod 中执行。有多种 API 资源来管理一个或多个 Pod。在本节中，我们将集中讨论 Deployment 和 ReplicaSet 这两个最与考试相关的 API 资源。此外，我们将简要介绍管理保持状态的工作负载的 StatefulSet。

## 理解 Deployments

在容器中运行应用程序的中心 API 资源是 Pod。使用单个 Pod 实例来运行应用程序存在缺陷。它代表一个单点故障，因为所有针对应用程序的流量都被引导到此 Pod。当负载因需求增加而增加时（例如，在电子商务应用程序的高峰购物季节期间，或者当中心微服务（如认证提供者）被系统内其他微服务越来越多地使用时），这种行为特别有问题。在 Pod 中运行应用程序的另一个重要方面是故障容忍性。在节点故障的情况下，Pod 将不会重新调度，因此可能导致终端用户的系统中断。在本节中，我们将讨论支持应用程序可伸缩性和故障容忍等方面的 Kubernetes 机制。

*ReplicaSet*是控制运行应用程序的多个相同 Pod 实例的 Kubernetes API 资源，称为副本。它具有根据需求扩展或缩减副本数量的能力。此外，它知道如何在所有副本中部署新版本的应用程序。

*Deployment*将 ReplicaSet 的功能抽象化，并在内部进行管理。在实际应用中，这意味着您无需自行创建、修改或删除 ReplicaSet 对象。Deployment 会保留应用程序版本的历史记录，并能够回滚到旧版本，以应对可能导致生产问题的阻塞或成本高昂的情况。此外，它还提供了扩展副本数量的能力。

图 3-1 说明了 Deployment、ReplicaSet 及其受控副本之间的关系。

![ckas 0301](img/ckas_0301.png)

###### 图 3-1\. Deployment 与 ReplicaSet 之间的关系

接下来的几节将解释如何管理 Deployments，包括扩展和回滚功能。

## 创建 Deployments

您可以使用命令`create deployment`创建一个 Deployment。该命令提供了多种选项，其中一些是必填的。至少，您需要提供 Deployment 的名称和副本应该使用的容器镜像。默认创建的副本数量为 1；不过，您可以使用选项`--replicas`定义更多的副本数量。

让我们看看命令的实际效果。以下命令创建名为`app-cache`的 Deployment，在四个副本中运行对象缓存[Memcached](https://memcached.org)：

```
$ kubectl create deployment app-cache --image=memcached:1.6.8 --replicas=4
deployment.apps/app-cache created
```

通过标签选择，Deployment 与其控制的副本之间的映射发生。当您运行命令式命令时，`kubectl`已为您设置好映射。示例 3-1 展示了 YAML 清单中的标签选择。此 YAML 清单可用于声明性地创建 Deployment，或通过检查前述命令式命令创建的现有对象来使用。

##### 示例 3-1\. 用于 Deployment 的 YAML 清单

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-cache
  labels:
    app: app-cache
spec:
  replicas: 4
  selector:
    matchLabels:
      app: app-cache
  template:
    metadata:
      labels:
        app: app-cache
    spec:
      containers:
      - name: memcached
        image: memcached:1.6.8
```

当通过命令式命令创建时，默认情况下，Deployment 使用的标签键是`app`。您可以在 YAML 输出的三个不同位置找到此键：

1.  `metadata.labels`

1.  `spec.selector.matchLabels`

1.  `spec.template.metadata.labels`

要使标签选择正常工作，`spec.selector.matchLabels`和`spec.template.metadata`的分配需要匹配，如图 3-2 所示。

![ckas 0302](img/ckas_0302.png)

###### 图 3-2\. Deployment 标签选择

`metadata.labels`的值对于将 Deployment 映射到 Pod 模板是无关紧要的。正如您在图 3-2 中所看到的，将`metadata.labels`的标签分配故意更改为`deploy: app-cache`，以强调对于 Deployment 到 Pod 模板选择而言并不重要。

## 列出 Deployments 及其 Pods

您可以使用命令`get deployments`在创建后检查 Deployment。该命令的输出呈现了其副本的重要细节，如下所示：

```
$ kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
app-cache   4/4     4            4           125m
```

你可以观察到与 Deployment 控制的副本相关的以下列标题，在表 3-1 中。

表 3-1\. 运行时副本信息在列出部署时

| 列标题 | 描述 |
| --- | --- |
| READY | 列出格式为<ready>/<desired>的副本数，用于向最终用户显示。所需副本数对应于`spec.replicas`的值。 |
| UP-TO-DATE | 列出已更新以达到期望状态的副本数。 |
| AVAILABLE | 列出对最终用户可用的副本数。 |

通过它们的名称中的前缀可以识别由部署控制的 Pods。在之前创建的部署的情况下，Pod 的名称以 `app-cache-` 开头。紧随前缀的哈希在创建时自动生成并分配给名称：

```
$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
app-cache-596bc5586d-84dkv   1/1     Running   0          6h5m
app-cache-596bc5586d-8bzfs   1/1     Running   0          6h5m
app-cache-596bc5586d-rc257   1/1     Running   0          6h5m
app-cache-596bc5586d-tvm4d   1/1     Running   0          6h5m
```

## 渲染部署详情

你可以渲染部署的详细信息。这些细节包括标签选择标准，当排除故障配置错误的部署时，这些细节可能非常有价值。以下输出提供了整体要点：

```
$ kubectl describe deployment app-cache
Name:                   app-cache
Namespace:              default
CreationTimestamp:      Sat, 07 Aug 2021 09:44:18 -0600
Labels:                 app=app-cache
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               app=app-cache
Replicas:               4 desired | 4 updated | 4 total | 4 available | \
                        0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=app-cache
  Containers:
   memcached:
    Image:        memcached:1.6.10
    Port:         <none>
    Host Port:    <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   app-cache-596bc5586d (4/4 replicas created)
Events:          <none>
```

你可能已经注意到输出中包含对 ReplicaSet 的引用。 ReplicaSet 的目的是*复制*一组相同的 Pods。你不需要深入理解 ReplicaSet 的核心功能来通过考试。只需知道 ReplicaSet 由部署自动创建，并使用与其控制的 Pods 类似的部署名称作为 ReplicaSet 的前缀。在前面命名为 `app-cache` 的部署的情况下，ReplicaSet 的名称为 `app-cache-596bc5586d`。

## 删除部署

部署全权负责创建和删除它所控制的对象，即 Pods 和 ReplicaSets。当你删除一个部署时，相应的对象也会被删除。假设你正在处理以下输出中显示的对象集：

```
$ kubectl get deployments,pods,replicasets
NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/app-cache   4/4     4            4           6h47m

NAME                             READY   STATUS    RESTARTS   AGE
pod/app-cache-596bc5586d-84dkv   1/1     Running   0          6h47m
pod/app-cache-596bc5586d-8bzfs   1/1     Running   0          6h47m
pod/app-cache-596bc5586d-rc257   1/1     Running   0          6h47m
pod/app-cache-596bc5586d-tvm4d   1/1     Running   0          6h47m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/app-cache-596bc5586d   4         4         4       6h47m
```

运行`delete deployment`命令以级联删除其管理的对象：

```
$ kubectl delete deployment app-cache
deployment.apps "app-cache" deleted
$ kubectl get deployments,pods,replicasets
No resources found in default namespace.
```

# 执行滚动更新和回滚

部分 API 资源内置了部署的发布和回滚功能。一旦更改了部署中的 Pod 模板的定义，Kubernetes 就知道如何将更改应用于对象管理的所有 Pods。在本节中，我们将讨论两种情况：部署应用程序的新版本和恢复应用程序的旧版本。

## 滚动部署新版本

部署使得将应用程序的新版本滚动到它所控制的所有副本变得非常容易。假设你想将 Memcached 的版本从 1.6.8 升级到 1.6.10，以便获得最新的功能和错误修复。你只需要通过更新 Pod 模板来更改对象的期望状态。部署会逐一更新所有副本到新版本。这个过程称为*滚动更新*。

随时可以使用命令`edit deployment`修改活动对象。或者，命令`set image`提供了一种快速便捷的方式来更改 Deployment 的镜像，如下所示：

```
$ kubectl set image deployment app-cache memcached=memcached:1.6.10 --record
deployment.apps/app-cache image updated
```

标志`--record`是可选的，默认值为`false`。如果提供而没有值或设置为`true`，则将记录用于更改的命令。在内部，`set image`命令通过分配具有键`kubernetes.io/change-cause`的注释来保留更改原因到 Deployment。

您可以在进行中检查滚动发布的当前状态。要使用的命令是`rollout status`。输出会给出已自命令发出后已更新的副本数量的指示：

```
$ kubectl rollout status deployment app-cache
Waiting for rollout to finish: 2 out of 4 new replicas have been updated...
deployment "app-cache" successfully rolled out
```

Kubernetes 在滚动历史中跟踪您对 Deployment 所做的更改。每个更改都由所谓的*修订版本*表示。您可以通过运行以下命令检查滚动历史。您将看到列出了两个修订版本：

```
$ kubectl rollout history deployment app-cache
deployment.apps/app-cache
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl set image deployment app-cache memcached=memcached:1.6.10 \
          --record=true
```

第一个修订版本是记录了创建对象时 Deployment 的原始状态。第二个修订版本是为更改镜像标签而添加的。请注意，“CHANGE-CAUSE”列显示了用于更改的命令。

要获取修订版本的更详细视图，请运行以下命令。您可以看到镜像使用的值为`memcached:1.6.10`：

```
$ kubectl rollout history deployments app-cache --revision=2
deployment.apps/app-cache with revision #2
Pod Template:
  Labels:	app=app-cache
	pod-template-hash=596bc5586d
  Annotations:	kubernetes.io/change-cause: kubectl set image deployment \
                app-cache memcached=memcached:1.6.10 --record=true
  Containers:
   memcached:
    Image:	memcached:1.6.10
    Port:	<none>
    Host Port:	<none>
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

## 回滚到之前的修订版本

在生产环境中可能会出现需要快速处理的问题。例如，假设刚刚发布的容器镜像包含一个关键的 bug。Kubernetes 允许您回滚到滚动历史中的先前修订版本之一。您可以使用`rollout undo`命令来实现这一点。要选择特定的修订版本，请提供命令行选项`--to-revision`。如果不提供选项，该命令将回滚到先前的修订版本。在这里，我们回滚到修订版本 1：

```
$ kubectl rollout undo deployment app-cache --to-revision=1
deployment.apps/app-cache rolled back
```

因此，Kubernetes 对所有副本执行了带有修订版本 1 的滚动更新。现在检查滚动历史，会列出修订版本 3。考虑到我们回滚到了修订版本 1，就没有必要再保留该条目作为重复的了。Kubernetes 简单地将修订版本 1 转换为 3，并从列表中删除 1：

```
$ kubectl rollout history deployment app-cache
deployment.apps/app-cache
REVISION  CHANGE-CAUSE
2         kubectl set image deployment app-cache memcached=memcached:1.6.10 \
          --record=true
3         <none>
```

# 扩展工作负载

扩展性是 Kubernetes 内置的能力之一。我们将学习如何根据应用程序负载的增加手动扩展副本数。此外，我们还将讨论 API 资源 Horizontal Pod Autoscaler，它允许根据 CPU 和内存等资源阈值自动扩展管理的 Pod 集合。

## 手动扩展 Deployment

增加或减少由 Deployment 控制的副本数量是一个简单的过程。您可以通过手动编辑活动对象使用`edit deployment`并更改`spec.replicas`属性的值，或者使用命令式`scale deployment`命令。以下命令将副本数量从四个增加到六个：

```
$ kubectl scale deployment app-cache --replicas=6
deployment.apps/app-cache scaled
```

您可以实时观察副本的创建过程。如果您速度足够快，您可能仍然可以看到新创建的 Pod 的状态从 `ContainerCreating` 变为 `Running`：

```
$ kubectl get pods
NAME                         READY   STATUS              RESTARTS   AGE
app-cache-5d6748d8b9-6cc4j   1/1     ContainerCreating   0          11s
app-cache-5d6748d8b9-6rmlj   1/1     Running             0          28m
app-cache-5d6748d8b9-6z7g5   1/1     ContainerCreating   0          11s
app-cache-5d6748d8b9-96dzf   1/1     Running             0          28m
app-cache-5d6748d8b9-jkjsv   1/1     Running             0          28m
app-cache-5d6748d8b9-svrxw   1/1     Running             0          28m
$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
app-cache-5d6748d8b9-6cc4j   1/1     Running   0          3m17s
app-cache-5d6748d8b9-6rmlj   1/1     Running   0          32m
app-cache-5d6748d8b9-6z7g5   1/1     Running   0          3m17s
app-cache-5d6748d8b9-96dzf   1/1     Running   0          32m
app-cache-5d6748d8b9-jkjsv   1/1     Running   0          31m
app-cache-5d6748d8b9-svrxw   1/1     Running   0          32m
```

手动扩展副本数需要一些猜测。您仍需监视系统负载，以确定副本数量是否足以处理传入的流量。

## 手动扩展 StatefulSet

另一个可以手动扩展的 API 资源是 StatefulSet。StatefulSet 用于通过一组 Pod（例如数据库）管理有状态应用程序。与部署类似，StatefulSet 定义了一个 Pod 模板；然而，每个副本保证具有独特和持久的标识。与部署类似，StatefulSet 使用 ReplicaSet 来管理副本。

我们不会详细讨论 StatefulSet，但您可以在 [文档](https://oreil.ly/dXJhh) 中进一步了解它们。我们在这里讨论 StatefulSet API 资源的原因是它可以像部署一样进行手动扩展。

假设我们要处理 StatefulSet 和 Service 的 YAML 定义，运行并公开 Redis 数据库，如 示例 3-2 所示。

##### 示例 3-2\. 用于 StatefulSet 和 Service 的 YAML 清单

```
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: default
  labels:
    app: redis
spec:
  ports:
  - port: 6379
    protocol: TCP
  selector:
    app: redis
  type: ClusterIP
  clusterIP: None
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  selector:
    matchLabels:
      app: redis
  replicas: 1
  serviceName: "redis"
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:6.2.5
        command: ["redis-server", "--appendonly", "yes"]
        ports:
        - containerPort: 6379
          name: web
        volumeMounts:
        - name: redis-vol
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: redis-vol
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

创建完成后，列出 StatefulSet 在 “READY” 列中的副本数。正如下面的输出所示，我们将副本数设置为 1：

```
$ kubectl create -f redis.yaml
service/redis created
statefulset.apps/redis created
$ kubectl get statefulset redis
NAME    READY   AGE
redis   1/1     2m10s
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
redis-0   1/1     Running   0          2m
```

我们在部署环境中探索的 `scale` 命令在这里同样有效。在以下命令中，我们将副本数从一个扩展到三个：

```
$ kubectl scale statefulset redis --replicas=3
statefulset.apps/redis scaled
$ kubectl get statefulset redis
NAME    READY   AGE
redis   3/3     3m43s
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
redis-0   1/1     Running   0          101m
redis-1   1/1     Running   0          97m
redis-2   1/1     Running   0          97m
```

需要提到的是，缩减 StatefulSet 的过程需要所有副本处于健康状态。由 StatefulSet 控制的 Pod 中长期未解决的问题可能会导致应用不可用于最终用户。

## 自动扩展部署

另一种扩展部署的方式是借助水平 Pod 自动扩展器（HPA）。HPA 是一个 API 原语，用于在特定条件下自动扩展副本数。当前稳定 API 版本的 HPA 仅支持 CPU 利用率作为扩展条件。运行时，HPA 会检查由 [metrics server](https://oreil.ly/Lmamb) 收集的指标，以确定部署的所有副本的平均最大 CPU 使用率是否低于或高于定义的阈值。图 3-3 展示了涉及 HPA 的高级架构图。

![ckas 0303](img/ckas_0303.png)

###### 图 3-3\. 自动扩展部署

## 创建水平 Pod 自动扩展器

您可以使用`autoscale deployment`命令为现有的 Deployment 创建 HPA。选项`--cpu-percent`定义平均最大 CPU 使用率阈值。选项`--min`和`--max`分别提供了缩减至的最小副本数以及 HPA 可创建的处理增加负载的最大副本数：

```
$ kubectl autoscale deployment app-cache --cpu-percent=80 --min=3 --max=5
horizontalpodautoscaler.autoscaling/app-cache autoscaled
```

上述命令是为 Deployment 创建 HPA 的绝佳快捷方式。HPA 对象的 YAML 清单表示如示例 3-3 所示。

##### 示例 3-3\. 用于 HPA 的 YAML 清单

```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: app-cache
spec:
  maxReplicas: 5
  minReplicas: 3
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-cache
  targetCPUUtilizationPercentage: 80
```

## 列出水平 Pod 自动缩放器

水平 Pod 自动缩放器的简称命令是`hpa`。透明地列出所有 HPA 对象描述了它们当前的状态：此时的 CPU 利用率使用情况和副本数量：

```
$ kubectl get hpa
NAME        REFERENCE              TARGETS         MINPODS   MAXPODS   REPLICAS \
  AGE
app-cache   Deployment/app-cache   <unknown>/80%   3         5         4        \
  58s
```

如果 Deployment 的 Pod 模板未定义 CPU 资源需求，或者无法从指标服务器检索 CPU 指标，则“TARGETS”列的左值显示为<unknown>。示例 3-4 设置 Pod 模板的资源需求，以便 HPA 正常工作。

##### 示例 3-4\. 设置 Pod 模板的 CPU 资源需求

```
# ...
spec:
  # ...
  template:
    # ...
    spec:
      containers:
      - name: memcached
        # ...
        resources:
          requests:
            cpu: 250m
          limits:
            cpu: 500m
```

一旦流量达到副本，当前 CPU 使用情况将以百分比的形式反映在以下输出中。这里平均最大 CPU 利用率为 15%：

```
$ kubectl get hpa
NAME        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
app-cache   Deployment/app-cache   15%/80%   3         5         4          58s
```

## 渲染水平 Pod 自动缩放器详情

HPA 的事件日志可以提供关于重新调整活动的额外见解。渲染 HPA 的详细信息可以是监视副本何时扩展或缩减及其缩放条件的好工具：

```
$ kubectl describe hpa app-cache
Name:                                                  app-cache
Namespace:                                             default
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Sun, 15 Aug 2021 \
                                                       15:54:11 -0600
Reference:                                             Deployment/app-cache
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (1m) / 80%
Min replicas:                                          3
Max replicas:                                          5
Deployment pods:                                       3 current / 3 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    ReadyForNewScale  recommended size matches current size
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully \
  calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  True    TooFewReplicas    the desired replica count is less \
  than the minimum replica count
Events:
  Type    Reason             Age   From                       Message
  ----    ------             ----  ----                       -------
  Normal  SuccessfulRescale  13m   horizontal-pod-autoscaler  New size: 3; \
  reason: All metrics below target
```

## 使用水平 Pod 自动缩放器的 Beta API 版本

Kubernetes 1.12 引入了 HPA 的 Beta API 版本`autoscaling/v2beta2`，该版本在 Kubernetes 1.23 中成为名为`autoscaling/v2`的最终 API。API 资源模型的 YAML 清单以更通用的方式观察指标。正如在示例 3-5 中所示，我们正在检查 CPU 和内存利用率，以确定是否需要扩展或缩减 Deployment 的副本。

##### 示例 3-5\. 使用 v2 的 HPA 的 YAML 清单

```
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-cache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app-cache
  minReplicas: 3
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 500Mi
```

为确保 HPA 确定当前使用的资源，我们还将设置 Pod 模板的内存资源需求，如示例 3-6 所示。

##### 示例 3-6\. 设置 Pod 模板的内存资源需求

```
...
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: memcached
        ...
        resources:
          requests:
            cpu: 250m
            memory: 100Mi
          limits:
            cpu: 500m
            memory: 500Mi
```

列出 HPA 会在“TARGETS”列中呈现两个指标，如此处显示的`get`命令输出：

```
$ kubectl get hpa
NAME        REFERENCE              TARGETS                 MINPODS   MAXPODS \
  REPLICAS   AGE
app-cache   Deployment/app-cache   1994752/500Mi, 0%/80%   3         5       \
  3          2m14s
```

# 定义和使用配置数据

遇到评估环境变量以控制其运行行为的应用程序是很常见的。例如，应用程序可以定义一个环境变量，指向外部服务的 URL，或者注入用于与另一个微服务进行身份验证的 API 密钥。

为容器声明环境变量很简单。你只需将它们作为键值对列在属性`spec.containers[].env[]`下即可。示例 3-7 定义了容器`memcached`的环境变量`MEMCACHED_CONNECTIONS`和`MEMCACHED_THREADS`。

##### 示例 3-7\. 为容器设置的环境变量

```
apiVersion: v1
kind: Pod
metadata:
  name: memcached
spec:
  containers:
  - name: memcached
    image: memcached:1.6.8
    env:
    - name: MEMCACHED_CONNECTIONS
      value: "2048"
    - name: MEMCACHED_THREADS
      value: "150"
```

如果这些环境变量成为同一命名空间内多个 Pod 清单中的常见商品，则无法避免复制粘贴定义。针对这种特定用例，Kubernetes 引入了由专用 API 资源表示的配置数据概念。

这些 API 资源称为 ConfigMap 和 Secret。它们都定义了一组键值对，并可以作为环境变量注入到容器中或者作为卷挂载。图 3-4 说明了这些选项。

# Secret 的值只能被编码

Secrets 期望每个条目的值都是 Base64 编码的。Base64 仅编码值，但不加密它。因此，任何拥有访问权限的人都可以无问题地解码它。Secret 仅分发给实际需要访问它的 Pod 所在的节点。此外，Secrets 存储在内存中，永远不会写入物理存储。

![ckas 0304](img/ckas_0304.png)

###### 图 3-4\. Kubernetes 中的配置数据

## 创建一个 ConfigMap

您可以通过发出命令`create configmap`来创建一个 ConfigMap。此命令要求您以选项的形式提供数据的来源。Kubernetes 区分了表 3-2 中显示的四种不同选项。

表 3-2\. 由 ConfigMap 解析数据的源选项

| 选项 | 示例 | 描述 |
| --- | --- | --- |
| `--from-literal` | `--from-literal=locale=en_US` | 字面值，即明文的键值对 |
| `--from-env-file` | `--from-env-file=config.env` | 包含键值对并期望它们成为环境变量的文件 |
| `--from-file` | `--from-file=app-config.json` | 一个带有任意内容的文件 |
| `--from-file` | `--from-file=config-dir` | 一个包含一个或多个文件的目录 |

很容易混淆选项`--from-env-file`和`--from-file`。选项`--from-env-file`期望一个文件，其中包含格式为`KEY=value`的环境变量，每行分隔。键值对遵循环境变量的典型命名约定（例如，键大写，单词之间用下划线分隔）。历史上，此选项已用于处理[Docker Compose `.env`文件](https://oreil.ly/Sd85f)，尽管您可以将其用于包含环境变量的任何其他文件。此选项不强制或规范环境变量的典型命名约定。选项`--from-file`指向包含*任何*任意内容的文件或目录。这是一个适合由应用程序读取的结构化配置数据文件的选项（例如，属性文件、JSON 文件或 XML 文件）。

以下命令显示了创建 ConfigMap 的过程。我们只是提供键值对作为字面值：

```
$ kubectl create configmap db-config --from-literal=DB_HOST=mysql-service \
  --from-literal=DB_USER=backend
configmap/db-config created
```

结果的 YAML 对象看起来像示例 3-8 中显示的对象。正如你所见，该对象在一个名为`data`的部分定义了键值对。ConfigMap 没有`spec`部分。

##### 示例 3-8\. ConfigMap YAML 文件

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_HOST: mysql-service
  DB_USER: backend
```

## 将 ConfigMap 作为环境变量使用

创建 ConfigMap 后，现在可以将其键值对作为环境变量注入到容器中。示例 3-9 展示了使用`spec.containers[].envFrom[].configMapRef`引用 ConfigMap 的名称。

##### 示例 3-9\. 将 ConfigMap 键值对注入容器

```
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - image: bmuschko/web-app:1.0.1
    name: backend
    envFrom:
    - configMapRef:
        name: db-config
```

从 YAML 文件清单创建 Pod 后，可以通过运行`env`Unix 命令检查容器中可用的环境变量。

```
$ kubectl exec backend -- env
...
DB_HOST=mysql-service
DB_USER=backend
...
```

## 将 ConfigMap 挂载为卷

另一种在运行时配置应用程序的方法是处理一个机器可读的配置文件。假设我们决定将数据库配置存储在名为`db.json`的 JSON 文件中，其结构如示例 3-10 所示。

##### 示例 3-10\. 用于配置数据库信息的 JSON 文件

```
{
    "db": {
      "host": "mysql-service",
      "user": "backend"
    }
}
```

鉴于我们不处理字面键值对，创建 ConfigMap 对象时需要提供选项`--from-file`：

```
$ kubectl create configmap db-config --from-file=db.json
configmap/db-config created
```

Pod 将 ConfigMap 作为卷挂载到容器内的特定路径。假设应用程序在启动时将读取配置文件。示例 3-11 演示了 YAML 定义。

##### 示例 3-11\. 将 ConfigMap 挂载为卷

```
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - image: bmuschko/web-app:1.0.1
    name: backend
    volumeMounts:
    - name: db-config-volume
      mountPath: /etc/config
  volumes:
  - name: db-config-volume
    configMap:
      name: db-config
```

若要验证正确的行为，请打开容器的交互式 shell。如以下命令所示，目录`/etc/config`包含一个使用在 ConfigMap 中使用的键的文件。内容表示 JSON 配置：

```
$ kubectl exec -it backend -- /bin/sh
# ls -1 /etc/config
db.json
# cat /etc/config/db.json
{
    "db": {
      "host": "mysql-service",
      "user": "backend"
    }
}
```

## 创建一个 Secret

您可以使用命令 `create secret` 创建一个秘密。此外，必须提供一个强制性子命令，确定秘密的类型。 表 3-3 列出了不同的类型。

表 3-3\. 创建秘密的选项

| 选项 | 描述 |
| --- | --- |
| `generic` | 从文件、目录或字面值创建秘密。 |
| `docker-registry` | 为 Docker 注册表创建秘密。 |
| `tls` | 创建一个 TLS 秘密。 |

最常用的秘密类型是 `generic`。通用秘密的选项与 ConfigMap 完全相同，如 表 3-4 所示。

表 3-4\. 由秘密解析的数据的源选项

| 选项 | 示例 | 描述 |
| --- | --- | --- |
| `--from-literal` | `--from-literal=password=secret` | 字面值，即键值对，以纯文本形式呈现 |
| `--from-env-file` | `--from-env-file=config.env` | 包含键值对并期望它们成为环境变量的文件 |
| `--from-file` | `--from-file=id_rsa=~/.ssh/id_rsa` | 包含任意内容的文件 |
| `--from-file` | `--from-file=config-dir` | 包含一个或多个文件的目录 |

为了演示其功能，让我们创建一个类型为 `generic` 的秘密。该命令将从作为命令行选项提供的字面值中获取键值对：

```
$ kubectl create secret generic db-creds --from-literal=pwd=s3cre!
secret/db-creds created
```

在使用命令创建时，秘密将自动对提供的值进行 Base64 编码。这可以通过查看生成的 YAML 清单来观察到。您可以在 示例 3-12 中看到，值 `s3cre!` 已经转换为 `czNjcmUh`，即其 Base64 编码等效形式。

##### 示例 3-12\. 带有 Base64 编码值的秘密

```
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:
  pwd: czNjcmUh
```

如果您从 YAML 清单开始创建秘密对象，则需要自行创建 Base64 编码的值。一个完成此任务的 Unix 工具是 `base64`。以下命令正是如此：

```
$ echo -n 's3cre!' | base64
czNjcmUh

```

或者，您还可以使用其中一个专门的 [秘密类型](https://oreil.ly/e1Cfz) 来避免提供 Base64 编码的值。类型 `kubernetes.io/basic-auth` 适用于基本身份验证，期望键 `username` 和 `password`。从此定义创建的对象将自动为这两个键的值进行 Base64 编码。 示例 3-13 展示了一个类型为 `kubernetes.io/basic-auth` 的秘密的 YAML 清单。请注意，定义键值对的属性称为 `stringData`，而不是 `Opaque` 秘密类型使用的 `data`。

##### 示例 3-13\. 使用类型为 kubernetes.io/basic-auth 的秘密

```
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: bmuschko
  password: secret
```

## 作为环境变量消耗秘密

作为环境变量消耗秘密与为 ConfigMaps 做法类似。在这里，您将使用 YAML 表达式`spec.containers[].envFrom[].secretRef`引用秘密的名称。示例 3-14 将名为`secret-basic-auth`的秘密注入为环境变量到名为`backend`的容器中。

##### 示例 3-14. 将秘密键值对注入到容器中

```
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - image: bmuschko/web-app:1.0.1
    name: backend
    envFrom:
    - secretRef:
        name: secret-basic-auth
```

检查容器中的环境变量，显示秘密值无需解码。这是 Kubernetes 自动完成的。因此，运行应用程序无需实现自定义逻辑来解码该值。请注意，Kubernetes 不验证也不规范环境变量的典型命名约定，正如您在以下输出中看到的：

```
$ kubectl exec backend -- env
...
username: bmuschko
password: secret
...
```

## 将一个秘密作为卷挂载

要演示将一个秘密作为卷挂载，我们将创建一个新的类型为`kubernetes.io/ssh-auth`的秘密。这种秘密类型捕获了一个 SSH 私钥的值，您可以使用命令`cat ~/.ssh/id_rsa`查看。要使用`create secret`命令处理 SSH 私钥文件，需要将其作为名为`ssh-privatekey`的文件可用：

```
$ cp ~/.ssh/id_rsa ssh-privatekey
$ kubectl create secret generic secret-ssh-auth --from-file=ssh-privatekey \
  --type=kubernetes.io/ssh-auth
secret/secret-ssh-auth created
```

将秘密作为卷挂载遵循两步方法：首先定义卷，然后将其引用为一个或多个容器的挂载路径。卷类型称为`secret`，如在示例 3-15 中使用的。

##### 示例 3-15. 将一个秘密作为卷挂载

```
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - image: bmuschko/web-app:1.0.1
    name: backend
    volumeMounts:
    - name: ssh-volume
      mountPath: /var/app
      readOnly: true
  volumes:
  - name: ssh-volume
    secret:
      secretName: secret-ssh-auth
```

您将在挂载路径`/var/app`中找到名为`ssh-privatekey`的文件。要验证，请打开一个交互式 shell 并渲染文件内容。文件的内容不是 Base64 编码：

```
$ kubectl exec -it backend -- /bin/sh
# ls -1 /var/app
ssh-privatekey
# cat /var/app/ssh-privatekey
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,8734C9153079F2E8497C8075289EBBF1
...
-----END RSA PRIVATE KEY-----
```

# 总结

CKA 涵盖的工作负载部分包括 API 资源部署、副本集和 Pod。部署控制着负责管理多个相同 Pod 的副本集。这些副本的数量可以通过手动或使用水平 Pod 自动缩放器自动缩放。对部署定义的副本模板进行的任何更改都将应用于副本。作为最终用户，您可以检查卷出历史记录、当前卷出状态及其进度。

应用程序的运行时行为可以通过将配置数据作为环境变量注入或将卷挂载到路径来控制。在 Kubernetes 中，这些配置数据以键值对的形式表示为 API 资源 ConfigMap 和 Secret。ConfigMap 用于纯文本数据，而 Secret 将值编码为 Base64 以混淆值。Secret 通常更适合包含敏感信息如凭证和 SSH 私钥的情况。

# 考试要点

知道如何扩展一个部署并滚动更新

部署具有一组 Pod 的卓越管理功能。与创建、更新和删除单个 Pod 相比，应优先使用它们。您需要熟悉部署的所有方面，包括手动扩展副本数量或借助水平 Pod 自动伸缩器进行自动扩展。回滚历史记录跟踪对 Pod 模板所做的修订。您可以部署新的修订版或回滚到先前的修订版。练习这些技术以及它对副本的影响。

练习创建和使用 ConfigMaps 和 Secrets。

可以使用 ConfigMap 或 Secret 将配置数据注入到 Pod 中。通过提供不同的数据源（例如文字值、文件和目录），实践使用命令式和声明式方法创建这些对象。Secrets 提供专门的类型。尝试不同的设置方式。您需要精通将 ConfigMaps 和 Secrets 定义的数据注入到容器中的不同方式。

# 示例练习

这些练习的解决方案位于附录中。

1.  创建名为`nginx`的部署（Deployment），使用镜像`nginx:1.17.0`。初始设置两个副本。

1.  使用`scale`命令将部署扩展到七个副本。确保存在正确数量的 Pod。

1.  为部署创建水平 Pod 自动伸缩器（Horizontal Pod Autoscaler），将 CPU 的平均利用率设置为 65%，内存的平均利用率设置为 1Gi。将最小副本数设置为 3，最大副本数设置为 20。

1.  更新部署的 Pod 模板，使用镜像`nginx:1.21.1`。确保记录更改。检查修订历史记录。应该呈现多少个修订版本？回滚到第一个修订版本。

1.  创建名为`basic-auth`的新 Secret，类型为`kubernetes.io/basic-auth`。分配键值对`username=super`和`password=my-s8cr3t`。将 Secret 作为卷挂载到路径`/etc/secret`，并为部署控制的 Pod 设置只读权限。
