# 第十六章：管理状态和有状态应用程序

在容器编排的早期阶段，目标工作负载通常是无状态应用程序，当需要时使用外部系统来存储状态。当时的想法是容器非常短暂，保持状态一致性的后备存储编排在最佳情况下也是困难的。随着时间的推移，基于容器的保持状态的工作负载成为了现实，并且在某些情况下，这种需求可能更高效。随着越来越多的组织寻求云计算能力，并且 Kubernetes 成为应用程序的事实标准容器运行时，阻碍因素成为数据量和对数据的高效访问，有时称为“数据引力”。Kubernetes 在多次迭代中进行了适应。现在，它不仅允许将存储卷挂载到 Pod 中，还允许 Kubernetes 直接管理这些卷。这是与需要存储的工作负载编排中的一个重要组成部分。

如果能够将外部卷挂载到容器就足够了，那么在 Kubernetes 中运行的规模化有状态应用程序的示例将更多。现实是，在有状态应用程序的大计划中，卷挂载只是一个简单的组成部分。大多数需要在节点故障后维护状态的应用程序是复杂的数据状态引擎，如关系数据库系统、分布式键/值存储和复杂文档管理系统。这类应用程序需要更多协调，包括集群应用程序成员之间的通信方式、成员的识别方式以及成员出现或消失的顺序。

本章重点介绍了管理状态的最佳实践，从简单模式，比如将文件保存到网络共享，到复杂的数据管理系统，如 MongoDB、MySQL 或 Kafka。还有一个关于名为操作员的新模式的小节，该模式不仅引入了 Kubernetes 的基本组件，还允许将业务或应用逻辑添加为自定义控制器，有助于更轻松地操作复杂的数据管理系统。

# 卷和卷挂载

并非每个需要维护状态的工作负载都需要成为复杂的数据库或高吞吐量数据队列服务。通常，将应用程序移动到容器化工作负载中的应用程序希望某些目录存在，并且能够读取和写入这些目录中的相关信息。可以从第四章中注入数据到可以由 Pod 中的容器读取的卷中，但是从 ConfigMaps 或秘密中挂载的数据通常是只读的，本节关注提供容器可以写入并能够在容器或甚至更好，Pod 故障后幸存的卷。

每个主要的容器运行时，如 Docker、rkt、CRI-O，甚至 Singularity，都允许将卷挂载到映射到外部存储系统的容器中。在其最简单形式下，外部存储可以是内存位置、容器主机上的路径，或外部文件系统，如 NFS、Glusterfs、CIFS 或 Ceph。为什么会需要这样做呢？一个有用的例子是，一个传统应用程序被编写为将应用程序特定信息记录到本地文件系统。许多可能的解决方案，如更新应用程序代码以将日志输出到侧车容器的`stdout`或`stderr`，可以通过共享的 pod 卷流式传输日志数据到外部源。有些方法会采用基础设施方法，通过使用可以读取容器应用程序日志和主机日志的卷的主机基于日志工具来进行卷挂载，使用 Kubernetes `hostPath` 挂载，如下所示：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-webserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-webserver
  template:
    metadata:
      labels:
        app: nginx-webserver
    spec:
      containers:
      - name: nginx-webserver
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
          - name: hostvol
            mountPath: /usr/share/nginx/html
      volumes:
        - name: hostvol
          hostPath:
            path: /home/webcontent
```

# 卷最佳实践

+   尽量限制将卷用于需要多个容器共享数据的 pod，如适配器或大使类型模式。对于这些共享模式，请使用`emptyDir`。

+   当需要通过基于节点的代理或服务访问数据时，请使用`hostDir`。

+   尝试识别任何将其关键应用程序日志和事件写入本地磁盘的服务，并在可能的情况下将其更改为`stdout`或`stderr`，让真正的 Kubernetes 感知日志聚合系统流式传输日志，而不是依赖卷映射。

# Kubernetes 存储

到目前为止，我们走过的示例展示了将卷基本映射到 pod 中的容器中，这只是基本的容器引擎能力。真正的关键是允许 Kubernetes 管理支持卷挂载的存储。这允许更动态的场景，其中 pod 可以根据需要存活和死亡，支持 pod 的存储将相应地过渡到 pod 可能生活的任何地方。Kubernetes 使用两个不同的 API 管理 pod 的存储，即 PersistentVolume 和 PersistentVolumeClaim。

## 持久卷

最好将 PersistentVolume 视为支持挂载到 pod 的任何卷的磁盘。PersistentVolume 将具有声明策略，定义卷的生命周期范围，独立于使用卷的 pod 的生命周期。Kubernetes 可以使用动态或静态定义的卷。要允许动态创建卷，必须在 Kubernetes 中定义一个 StorageClass。可以在集群中创建不同类型和类别的 PersistentVolumes，只有当 PersistentVolumeClaim 匹配 PersistentVolume 时，它才会真正分配给一个 pod。卷本身由卷插件支持。直接在 Kubernetes 中支持多种插件，并且每种插件都有不同的配置参数可供调整：

```
apiVersion: v1
kind: PersistentVolume
metadata:
name: pv001
labels:
  tier: "silver"
spec:
capacity:
  storage: 5Gi
accessModes:
- ReadWriteMany
persistentVolumeReclaimPolicy: Recycle
storageClassName: nfs
mountOptions:
  - hard
  - nfsvers=4.1
nfs:
  path: /tmp
  server: 172.17.0.2
```

## PersistentVolumeClaims

PersistentVolumeClaims 是 Kubernetes 中为存储定义资源需求的一种方式，Pods 将引用这些声明，如果存在与声明请求匹配的 `persistentVolume`，则将为该特定 Pod 分配该卷。至少需要定义存储请求大小和访问模式，但也可以定义特定的 StorageClass。选择器也可以用来确保分配符合特定条件的 PersistentVolumes。在以下示例中，具有键 `tier` 和值 `"silver"` 的标签：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClass: nfs
    accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  selector:
    matchLabels:
      tier: "silver"
```

此声明将与之前创建的 PersistentVolume 匹配，因为 StorageClass 名称、选择器匹配、大小和访问模式都相等。

Kubernetes 将匹配 PersistentVolume 与声明并将它们绑定在一起。要使用该卷，`pod.spec` 应通过名称引用声明，如下所示：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-webserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-webserver
  template:
    metadata:
      labels:
        app: nginx-webserver
    spec:
      containers:
      - name: nginx-webserver
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
          - name: hostvol
            mountPath: /usr/share/nginx/html
      volumes:
        - name: hostvol
          persistentVolumeClaim:
            claimName: my-pvc
```

## StorageClasses

管理员可以选择创建 StorageClass 对象，而不是手动预先定义 PersistentVolumes，这些对象定义了要使用的卷插件。他们还可以创建所有该类 PersistentVolumes 将使用的任何特定挂载选项和参数。然后可以根据 StorageClass 的参数和选项动态创建 PersistentVolume，并定义声明要使用的特定 StorageClass，Kubernetes 将基于 StorageClass 参数和选项动态创建 PersistentVolume：

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
name: nfs
provisioner: cluster.local/nfs-client-provisioner
parameters:
  archiveOnDelete: True
```

Kubernetes 还允许操作员使用 DefaultStorageClass 准入插件创建默认存储类。如果已在 API 服务器上启用此选项，则可以定义默认 StorageClass，并且任何未明确定义 StorageClass 的 PersistentVolumeClaims 将分配给默认类。某些云提供商将包括默认存储类，以映射到其实例允许的最便宜的存储。

### 容器存储接口和 FlexVolume

大多数卷插件今天需要等待 Kubernetes 代码库的直接代码添加。然而，容器存储接口 (CSI) 和 FlexVolume（通常称为“Out-of-Tree”卷插件）使存储供应商能够创建自定义存储插件，而无需等待这些直接代码添加。

CSI 和 FlexVolume 插件作为运营商在 Kubernetes 集群上部署的扩展，并且可以在需要时由存储供应商进行更新以公开新功能。

CSI 在 [GitHub](https://oreil.ly/AuMgE) 上表明其目标是：

> 定义一个行业标准的容器存储接口，使存储供应商（SP）可以开发一次插件并在多个容器编排系统（CO）中运行。

FlexVolume 接口一直是添加存储提供程序附加功能的传统方法。它确实需要在将使用它的集群的所有节点上安装特定驱动程序。这基本上成为集群主机上安装的可执行文件。这个最后的组件是使用 FlexVolumes 的主要缺点，特别是在托管服务提供商中，因为访问节点是不受欢迎的，而访问控制平面几乎是不可能的。CSI 插件通过暴露相同的功能并像部署 pod 到集群一样容易使用解决了这个问题。

## Kubernetes 存储最佳实践

云原生应用程序设计原则尽可能强制无状态应用程序设计；然而，基于容器的服务的不断增长使得数据存储持久性成为必要。这些关于 Kubernetes 存储的最佳实践将有助于设计一种有效的方法来提供应用程序设计所需的存储实现：

+   如果可能的话，启用 DefaultStorageClass 准入插件，并定义一个默认的存储类。通常，需要 PersistentVolumes 的应用程序的 Helm charts 默认使用 `default` 存储类，这使得应用程序可以在不太多修改的情况下安装。

+   在设计集群架构时（不论是在本地还是在云提供商），考虑计算和数据层之间的区域和连接。您将希望为节点和 PersistentVolumes 使用适当的标签，并使用亲和性使数据和工作负载尽可能接近。你最不希望的情况是，位于 A 区域的节点上的 pod 尝试挂载附加到 B 区域节点上的卷。

+   非常仔细地考虑哪些工作负载需要在磁盘上维护状态。这是否可以通过外部服务来处理，比如数据库系统？或者，您的实例是否可以在云提供商中运行，通过与当前使用的 API 一致的托管服务，比如作为服务的 MongoDB 或 MySQL？

+   确定修改应用程序代码以实现更无状态化需要付出多大的努力。

+   虽然 Kubernetes 将跟踪和挂载卷随着工作负载的调度，但它尚未处理存储在这些卷中的数据的冗余和备份。CSI 规范已添加了一个 API，供供应商插入本地快照技术，如果存储后端支持的话。

+   验证卷将保存的数据的适当生命周期。默认情况下，为动态配置的 PersistentVolumes 设置回收策略，这将在删除 pod 时从后备存储提供程序中删除卷。应设置敏感数据或可用于取证分析的数据以回收。

# 有状态应用程序

与流行观念相反，Kubernetes 从其初始阶段就支持有状态应用程序，从 MySQL、Kafka 和 Cassandra 到其他技术。然而，这些先驱时期充满了复杂性，并且通常只适用于小型工作负载，需要大量工作才能使缩放和耐久性等功能正常运行。

要完全理解关键差异，你必须了解典型 ReplicaSet 如何调度和管理 Pods，以及这如何对传统有状态应用程序有害：

+   ReplicaSet 中的 Pods 在调度时被分配随机名称并进行扩展。

+   ReplicaSet 中的 Pods 随意缩减。

+   ReplicaSet 中的 Pods 不会通过其名称或 IP 地址直接调用，而是通过与 Service 的关联。

+   ReplicaSet 中的 Pods 可以随时重新启动并移动到另一个节点。

+   ReplicaSet 中具有映射 PersistentVolume 的 Pods 仅通过声明链接，但如果需要重新调度，任何具有新名称的新 Pod 都可以接管该声明。

那些对集群数据管理系统只有粗浅了解的人可以立即看到 ReplicaSet 基础 Pods 的这些特性存在问题。想象一下，一个拥有当前可写副本的数据库的 Pod 突然被删除！肯定会造成纯粹的混乱。

Kubernetes 中的大多数新手都认为 StatefulSet 应用程序自动是数据库应用程序，因此等同起来。事实并非如此。Kubernetes 不知道它正在部署的应用程序类型。它不知道你的数据库系统需要领导选举过程，它是否能够处理集合成员之间的数据复制，或者它是否是数据库系统。这就是 StatefulSets 发挥作用的地方。

## StatefulSets

StatefulSets 的作用是更容易运行期望更可靠节点/Pod 行为的应用系统。如果我们回顾一下 ReplicaSet 中典型 Pods 特性的列表，StatefulSets 几乎完全相反。最初在 Kubernetes 版本 1.3 中提出的 PetSets（现在的 StatefulSets）是为了满足诸如复杂数据管理系统等有状态应用程序的关键调度和管理需求：

+   StatefulSet 中的 Pods 将会被扩展并分配顺序名称。随着集合的扩展，Pods 将得到序数名称，默认情况下，新的 Pod 必须完全在线（通过其健康检查和/或准备就绪探针）才能添加下一个 Pod。

+   StatefulSet 中的 Pods 按相反的顺序缩减。

+   StatefulSet 中的 Pods 可以通过头部无服务的方式按名称单独访问。

+   StatefulSet 中需要挂载卷的 Pods 必须使用定义好的 PersistentVolume 模板。StatefulSet 中的 Pods 所声明的卷在删除 StatefulSet 时不会被删除。

StatefulSet 规范与 Deployment 非常相似，除了 Service 声明和 PersistentVolume 模板之外。应首先创建无头 Service，它定义了将逐个为 pod 寻址的 Service。无头 Service 与常规 Service 相同，但不执行正常的负载均衡：

```
apiVersion: v1
kind: Service
metadata:
  name: mongo
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None #This creates the headless Service
  selector:
    role: mongo
```

StatefulSet 定义看起来也和 Deployment 几乎一模一样，只有少许不同之处：

```
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 3
  template:
    metadata:
      labels:
        role: mongo
        environment: test
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo:3.4
          command:
            - mongod
            - "--replSet"
            - rs0
            - "--bind_ip"
            - 0.0.0.0
            - "--smallfiles"
            - "--noprealloc"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-persistent-storage
              mountPath: /data/db
        - name: mongo-sidecar
          image: cvallance/mongo-k8s-sidecar
          env:
            - name: MONGO_SIDECAR_POD_LABELS
              value: "role=mongo,environment=test"
  volumeClaimTemplates:
  - metadata:
      name: mongo-persistent-storage
      annotations:
        volume.beta.kubernetes.io/storage-class: "fast"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 2Gi
```

## 运算符

StatefulSets 在将复杂有状态数据系统引入 Kubernetes 中作为可行工作负载方面起了重要作用。正如前面所述，唯一真正的问题是 Kubernetes 实际上并不了解在 StatefulSet 中运行的工作负载。所有其他复杂操作，如备份、故障转移、领导者注册、新副本注册和升级，都是需要定期执行的操作，在作为 StatefulSets 运行时需要仔细考虑。

在 Kubernetes 的早期阶段，CoreOS 的可靠性工程师（SREs）创建了一种称为运算符的新型云原生软件，用于 Kubernetes。最初的目的是将运行特定应用程序的领域特定知识封装到一个控制器中，从而扩展 Kubernetes 的能力。想象一下在 StatefulSet 控制器的基础上构建能够部署、扩展、升级、备份和运行 Cassandra 或 Kafka 等操作的控制器。最早创建的一些运算符是为 etcd 和 Prometheus 创建的，后者使用时间序列数据库长期保存指标。通过运算符可以正确创建、备份和恢复配置 Prometheus 或 etcd 实例，并且它们是新的由 Kubernetes 管理的对象，就像 pod 或 Deployment 一样。

直到最近，运算符一直是由 SRE 或软件供应商为其特定应用程序创建的一次性工具。在 2018 年中期，Red Hat 创建了 Operator Framework，这是一套工具，包括 SDK 生命周期管理器和未来模块，将支持计量、市场和注册表类型功能。运算符不仅适用于有状态应用程序，而且由于其自定义控制逻辑，它们确实更适合复杂数据服务和有状态系统。

运算符不仅成为扩展 Kubernetes API 的标准方式，还将最佳实践和操作监控引入到 Kubernetes 中复杂系统流程中。发现已发布的 Kubernetes 生态系统运算符的好地方是[OperatorHub](http://operatorhub.io)。他们维护一个更新的经过策划的运算符列表。

如果你有兴趣了解运算符的工作原理，第二十一章是这版新增的内容，会为你提供开发运算符和使用的最佳实践基础。此外，可以查阅《*Kubernetes Operators*》（O'Reilly）由 Jason Dobies 和 Joshua Wood 撰写，深入了解构建运算符的详细步骤。

## StatefulSet 和运营商最佳实践

需要状态和可能复杂管理和配置操作的大型分布式应用程序受益于 Kubernetes StatefulSets 和 Operators。运营商仍在发展中，但它们得到了社区的大力支持，因此这些最佳实践基于发布时的当前能力：

+   使用 StatefulSets 应谨慎，因为状态型应用通常需要更深入的管理，目前编排器无法很好地管理（请阅读“运营商”，可能会对 Kubernetes 的这一不足提供未来的解决方案）。

+   StatefulSet 的无头 Service 不会自动创建，必须在部署时创建，以正确地将 pod 地址作为单独的节点处理。

+   当应用程序需要顺序命名和可靠的扩展时，并不总是意味着需要分配 PersistentVolumes。

+   如果集群中的节点变得无响应，则 StatefulSet 的任何 pod 都不会自动删除；相反，在优雅期限后，它们将进入 `Terminating` 或 `Unknown` 状态。清除此 pod 的唯一方法是从集群中删除节点对象，kubelet 开始重新工作并直接删除 pod，或者运营商强制删除 pod。强制删除应该是最后的选择，并且应该小心确保删除 pod 的节点不会再次上线，因为现在集群中将有两个具有相同名称的 pod。您可以使用 `kubectl delete pod nginx-0 --grace-period=0 --force` 强制删除该 pod。

+   即使强制删除 pod 后，它可能仍然处于 `Unknown` 状态，因此对 API 服务器的修补将删除该条目，并导致 StatefulSet 控制器创建已删除 pod 的新实例：`kubectl patch pod nginx-0 -p '{"metadata":{"finalizers":null}}'`。

+   如果正在运行具有某种类型的领导者选举或数据复制确认过程的复杂数据系统，请使用 `preStop hook` 来适当关闭任何连接，强制进行领导者选举或在使用优雅关闭进程删除 pod 前验证数据同步。

+   当需要有状态数据的应用程序是复杂的数据管理系统时，请查看是否存在运营商来帮助管理应用程序更复杂的生命周期组件。如果应用程序是内部构建的，则值得调查是否将应用程序打包为运营商以增加应用程序的可管理性。参见[CoreOS 运营商 SDK](https://oreil.ly/gRIej) 作为示例。

# 概要

大多数组织希望将其无状态应用程序容器化，而将有状态应用程序保持不变。随着越来越多的云原生应用在云服务提供商的 Kubernetes 平台上运行，数据引力成为一个问题。有状态应用程序需要更多的尽职调查，但通过引入 StatefulSets 和 Operators，将它们运行在集群中的现实性已经加快。将卷映射到容器中允许 Operators 将存储子系统的具体细节与任何应用程序开发分离开来。在 Kubernetes 中管理诸如数据库系统之类的有状态应用程序仍然是一个复杂的分布式系统，需要使用 Pods、ReplicaSets、Deployments 和 StatefulSets 等原生 Kubernetes 基元进行仔细的编排。使用具有特定应用程序知识的 Operators 作为 Kubernetes 本地 API 可能有助于将这些系统提升到基于生产的集群中。
