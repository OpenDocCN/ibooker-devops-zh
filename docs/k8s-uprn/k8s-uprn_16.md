# 第十六章：集成存储解决方案与 Kubernetes

在许多情况下，将状态从应用程序中解耦，并构建你的微服务尽可能无状态，能够最大程度地提高系统的可靠性和可管理性。

然而，几乎每个有任何复杂性的系统在系统中都有状态，从数据库中的记录到为网页搜索引擎提供结果的索引碎片。总有一些时候，你必须将数据存储在某个地方。

将这些数据与容器和容器编排解决方案集成，通常是构建分布式系统中最复杂的方面。这种复杂性很大程度上源于向容器化架构的迁移也是向解耦、不可变和声明式应用程序开发的迁移。这些模式相对容易应用于无状态的 Web 应用程序，但即使是像 Cassandra 或 MongoDB 这样的“云原生”存储解决方案，也涉及一些手动或命令式的步骤来设置可靠的、复制的解决方案。

作为例子，考虑在 MongoDB 中设置一个 ReplicaSet，这涉及到部署 Mongo 守护进程，然后运行一个命令来识别 Mongo 集群中的领导者和参与者。当然，这些步骤可以通过脚本完成，但在容器化的世界中，很难看到如何将这些命令集成到部署中。同样，即使为一个容器化的副本集的单个容器获取 DNS 可解析的名称也是一项挑战。

额外的复杂性来自于数据引力的存在。大多数容器化系统并不是在真空中构建的；它们通常是从部署在虚拟机上的现有系统改编而来的，这些系统可能包括必须导入或迁移的数据。

最   最终，向云端演进通常意味着存储是一个外部化的云服务，在这种情况下，它永远不可能真正存在于 Kubernetes 集群内部。

本章涵盖了多种将存储集成到 Kubernetes 容器化微服务中的方法。首先，我们介绍如何将现有的外部存储解决方案（无论是云服务还是运行在虚拟机上的）导入到 Kubernetes 中。接下来，我们探讨如何在 Kubernetes 中运行可靠的单例服务，使你能够拥有一个与之前部署存储解决方案的虚拟机环境大致相同的环境。最后，我们介绍 StatefulSets，这是大多数人用来在 Kubernetes 中处理有状态工作负载的 Kubernetes 资源。

# 导入外部服务

在许多情况下，你在网络中运行着一台已有的机器，机器上运行着某种数据库。在这种情况下，你可能不希望立即将该数据库迁移到容器和 Kubernetes 中。也许它由不同的团队维护，或者你正在进行渐进式迁移，或者迁移数据的任务本身就是值得的麻烦。

不管停留在此的原因是什么，这个旧的服务器和服务都不会迁移到 Kubernetes⁠—但在 Kubernetes 中代表这个服务器仍然是有价值的。这样做的好处是您可以利用 Kubernetes 提供的所有内置命名和服务发现原语。此外，这使您能够配置所有应用程序，使其看起来像在某台机器上运行的数据库实际上是一个 Kubernetes 服务。这意味着可以轻松地将其替换为作为瞬态容器运行的测试数据库。例如，在生产环境中，您可能依赖运行在某台机器上的旧数据库，但对于连续测试，您可以部署一个测试数据库作为临时容器。由于它每次测试运行时都会创建和销毁，数据持久性在连续测试案例中并不重要。将这两个数据库都表示为 Kubernetes 服务使您能够在测试和生产中保持相同的配置。测试和生产之间的高度一致性确保通过测试将导致在生产环境中成功部署。

要具体了解如何在开发和生产之间保持高度一致，请记住所有 Kubernetes 对象都部署到 *命名空间* 中。假设我们定义了 `test` 和 `production` 命名空间。可以使用类似以下对象导入测试服务：

```
kind: Service
metadata:
  name: my-database
  # note 'test' namespace here
  namespace: test
...
```

生产服务看起来一样，只是使用了不同的命名空间：

```
kind: Service
metadata:
  name: my-database
  # note 'prod' namespace here
  namespace: prod
...
```

当您将 Pod 部署到 `test` 命名空间并查找名为 `my-database` 的服务时，它将接收指向 `my-database.test.svc.cluster.internal` 的指针，这将指向测试数据库。相比之下，当部署在 `prod` 命名空间中的 Pod 查找相同名称 (`my-database`) 时，它将收到指向 `my-database.prod.svc.cluster.internal` 的指针，这是生产数据库。因此，相同的服务名称在两个不同的命名空间中解析为两个不同的服务。有关此工作原理的更多详细信息，请参阅第七章。

###### 注意

所有以下技术都使用数据库或其他存储服务，但这些方法同样适用于没有在您的 Kubernetes 集群内运行的其他服务。

## 没有选择器的服务

当我们首次介绍服务时，我们详细讨论了标签查询及其如何用于识别特定服务的后端动态 Pod 集合。然而，对于外部服务，没有这样的标签查询。相反，通常有一个 DNS 名称，它指向运行数据库的特定服务器。在我们的示例中，假设此服务器命名为 `database.company.com`。要将此外部数据库服务导入 Kubernetes，我们首先创建一个服务，它没有 Pod 选择器，而是引用数据库服务器的 DNS 名称（示例 16-1）。

##### 示例 16-1\. dns-service.yaml

```
kind: Service
apiVersion: v1
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.company.com
```

当创建典型的 Kubernetes 服务时，还会创建一个 IP 地址，并且 Kubernetes DNS 服务会填充一个 A 记录，指向该 IP 地址。当您创建 `ExternalName` 类型的服务时，Kubernetes DNS 服务会填充一个 CNAME 记录，指向您指定的外部名称（在本例中为 `database.company.com`）。当集群中的应用程序对主机名 `external-database.svc.default.cluster` 进行 DNS 查找时，DNS 协议将该名称别名为 `database.company.com`。然后，这将解析为您外部数据库服务器的 IP 地址。通过这种方式，Kubernetes 中的所有容器都认为它们正在与支持其他容器的服务通信，而实际上它们被重定向到外部数据库。

请注意，这并不仅限于您自己基础设施上运行的数据库。许多云数据库和其他服务在访问数据库时会提供一个 DNS 名称（例如，`my-database.databases.cloudprovider.com`）。您可以将此 DNS 名称用作 `externalName`。这将把由云提供的数据库导入到您的 Kubernetes 集群的命名空间中。

有时，您可能没有外部数据库服务的 DNS 地址，只有一个 IP 地址。在这种情况下，仍然可以将此服务导入为 Kubernetes 服务，但操作略有不同。首先，创建一个没有标签选择器的服务，但也不使用我们之前使用的 `ExternalName` 类型（示例 16-2）。

##### 示例 16-2\. external-ip-service.yaml

```
kind: Service
apiVersion: v1
metadata:
  name: external-ip-database
```

Kubernetes 将为此服务分配一个虚拟 IP 地址，并为其填充一个 A 记录。然而，由于服务没有选择器，负载均衡器将不会填充任何端点以重定向流量。

鉴于这是一个外部服务，用户需负责手动使用 Endpoints 资源（示例 16-3）来填充端点。

##### 示例 16-3\. external-ip-endpoints.yaml

```
kind: Endpoints
apiVersion: v1
metadata:
  name: external-ip-database
subsets:
  - addresses:
    - ip: 192.168.0.1
    ports:
    - port: 3306
```

如果您有多个 IP 地址以实现冗余，可以在 `addresses` 数组中重复它们。一旦填充了端点，负载均衡器将开始将流量从 Kubernetes 服务重定向到 IP 地址端点。

###### 注意

由于用户已经承担了保持服务器 IP 地址更新的责任，您需要确保它永远不会更改，或者确保某些自动化流程更新 `Endpoints` 记录。

## 外部服务的限制：健康检查

Kubernetes 中的外部服务有一个重要的限制：它们不执行任何健康检查。用户需确保供给 Kubernetes 的端点或 DNS 名称对应的可靠性足够满足应用需求。

# 运行可靠的单例

在 Kubernetes 中运行存储解决方案的挑战通常在于，诸如 ReplicaSet 这样的基元期望每个容器都是相同且可替换的，但对于大多数存储解决方案来说，情况并非如此。解决这个问题的一个选项是使用 Kubernetes 基元，但不尝试复制存储。相反，只需运行一个单独的 Pod，其中运行数据库或其他存储解决方案即可。通过这种方式，运行 Kubernetes 中复制的存储的挑战不会发生，因为没有复制。

乍一看，这似乎与构建可靠的分布式系统的原则相悖，但总体而言，它并不比将数据库或存储基础设施运行在单个虚拟或物理机器上不可靠。实际上，如果正确结构化系统，您所牺牲的唯一东西就是在升级或机器故障时的潜在停机时间。尽管对于大规模或关键任务的系统而言，这可能是不可接受的，但对于许多较小规模的应用程序来说，这种有限的停机时间是减少复杂性的合理权衡。如果这对您不适用，请随意跳过本节，或按照前一节描述的导入现有服务的方式进行操作，或者继续阅读“使用 StatefulSets 实现 Kubernetes 本地存储”。对于其他人，我们将介绍如何构建可靠的数据存储单例。

## 运行 MySQL 单例

在这一节中，我们将描述如何在 Kubernetes 中将 MySQL 数据库的可靠单例实例作为 Pod 运行，并且如何将该单例暴露给集群中的其他应用程序。为此，我们将创建三个基本对象：

+   持久卷用于独立管理磁盘存储的生命周期，与运行中的 MySQL 应用程序的生命周期无关。

+   一个将运行 MySQL 应用程序的 MySQL Pod

+   一个服务，将此 Pod 暴露给集群中的其他容器

在第五章中，我们描述了持久卷：这些存储位置的生命周期与任何 Pod 或容器无关。持久卷在持久存储解决方案的情况下非常有用，其中数据库的磁盘表示应该在容器运行数据库应用程序崩溃或移动到不同机器时仍然存在。如果应用程序移动到不同的机器，卷应随之移动，并且数据应该得到保留。将数据存储分离为持久卷使这一切成为可能。

首先，我们将为我们的 MySQL 数据库创建一个持久卷供其使用。本示例使用 NFS 实现最大的可移植性，但 Kubernetes 支持许多不同的持久卷驱动程序类型。例如，有适用于所有主要公共云提供商以及许多私有云提供商的持久卷驱动程序。要使用这些解决方案，只需将 `nfs` 替换为适当的云提供商卷类型（例如 `azure`、`awsElasticBlockStore` 或 `gcePersistentDisk`）。在所有情况下，这些更改就足够了。Kubernetes 知道如何在相应的云提供商中创建适当的存储磁盘。这是 Kubernetes 简化可靠分布式系统开发的一个很好的例子。示例 16-4 展示了 PersistentVolume 对象。

##### 示例 16-4\. nfs-volume.yaml

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: database
  labels:
    volume: my-volume
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 1Gi
  nfs:
    server: 192.168.0.1
    path: "/exports"
```

这定义了一个具有 1 GB 存储空间的 NFS PersistentVolume 对象。我们可以像往常一样创建这个持久卷：

```
$ kubectl apply -f nfs-volume.yaml
```

现在我们已经创建了一个持久卷，我们需要为我们的 Pod 声明该持久卷。我们使用 PersistentVolumeClaim 对象来实现这一点（示例 16-5）。

##### 示例 16-5\. nfs-volume-claim.yaml

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: database
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      volume: my-volume
```

`selector` 字段使用标签来查找我们之前定义的匹配卷。

这种间接的方法可能看起来过于复杂，但它有其目的——它用来将我们的 Pod 定义与存储定义隔离开来。您可以直接在 Pod 规范中声明卷，但这会将该 Pod 规范锁定到特定的卷提供者（例如特定的公共或私有云）。通过使用卷声明，您可以保持您的 Pod 规范与云无关；简单地创建不同的卷，特定于云，并使用 PersistentVolumeClaim 将它们绑定在一起。此外，在许多情况下，持久卷控制器实际上会自动为您创建卷。关于此过程的更多详细信息，请参见下一节。

现在我们已经声明了我们的卷，我们可以使用 ReplicaSet 来构建我们的单例 Pod。也许使用 ReplicaSet 来管理单个 Pod 看起来有些奇怪，但这对于可靠性是必要的。请记住，一旦调度到一台机器上，一个裸 Pod 将永远绑定到该机器上。如果机器发生故障，则任何不受高级控制器（如 ReplicaSet）管理的 Pod 都将与该机器一起消失，并且不会在其他地方重新调度。因此，为了确保我们的数据库 Pod 在机器故障时重新调度，我们使用高级别的 ReplicaSet 控制器，其副本大小为 `1`，来管理我们的数据库（示例 16-6）。

##### 示例 16-6\. mysql-replicaset.yaml

```
apiVersion: extensions/v1
kind: ReplicaSet
metadata:
  name: mysql
  # Labels so that we can bind a Service to this Pod
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: database
        image: mysql
        resources:
          requests:
            cpu: 1
            memory: 2Gi
        env:
        # Environment variables are not a best practice for security,
        # but we're using them here for brevity in the example.
        # See Chapter 11 for better options.
        - name: MYSQL_ROOT_PASSWORD
          value: some-password-here
        livenessProbe:
          tcpSocket:
            port: 3306
        ports:
        - containerPort: 3306
        volumeMounts:
          - name: database
            # /var/lib/mysql is where MySQL stores its databases
            mountPath: "/var/lib/mysql"
      volumes:
      - name: database
        persistentVolumeClaim:
          claimName: database
```

一旦我们创建了 ReplicaSet，它将进而创建一个运行 MySQL 的 Pod，使用我们最初创建的持久磁盘。最后一步是将其公开为 Kubernetes 服务（示例 16-7）。

##### 示例 16-7\. mysql-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
  - port: 3306
    protocol: TCP
  selector:
    app: mysql
```

现在，我们在集群中运行了一个可靠的单例 MySQL 实例，并将其公开为名为`mysql`的服务的完整域名`mysql.svc.default.cluster`可访问。

类似的说明可以用于各种数据存储，并且如果您的需求简单，并且可以在面对机器故障或需要升级数据库软件时承受有限的停机时间，那么可靠的单例可能是您的应用程序存储的正确方法。

## 动态卷供应

许多集群还包括*动态卷供应*。通过动态卷供应，集群操作员创建一个或多个`StorageClass`对象。在 Kubernetes 中，`StorageClass`封装了特定类型存储的特征。一个集群可以安装多个不同的存储类。例如，您可能在网络上有一个 NFS 服务器的存储类和一个 iSCSI 块存储的存储类。存储类还可以封装不同的可靠性或性能级别。示例 16-8 展示了在 Microsoft Azure 平台上自动创建磁盘对象的默认存储类。

##### 示例 16-8\. storageclass.yaml

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: default
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
  labels:
    kubernetes.io/cluster-service: "true"
provisioner: kubernetes.io/azure-disk
```

一旦为集群创建了存储类，您可以在持久卷声明中引用此存储类，而不是引用任何特定的持久卷。当动态供应程序看到这个存储声明时，它会使用适当的卷驱动程序来创建卷并将其绑定到您的持久卷声明上。

示例 16-9 展示了一个使用我们刚刚定义的`default`存储类来声明新创建的持久卷的 PersistentVolumeClaim 的示例。

##### 示例 16-9\. dynamic-volume-claim.yaml

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: my-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

`volume.beta.kubernetes.io/storage-class`注释将此声明与我们创建的存储类关联起来。

###### 注意

自动提供持久卷是一个很棒的功能，它显著简化了在 Kubernetes 中构建和管理有状态应用程序的过程。但是，这些持久卷的生命周期取决于 PersistentVolumeClaim 的回收策略，默认情况下将其绑定到创建卷的 Pod 的生命周期。

这意味着如果您删除 Pod（例如通过缩减规模或其他事件），那么卷也将被删除。虽然在某些情况下这可能是您希望的结果，但您需要小心确保不会意外删除您的持久卷。

持久卷非常适合需要存储的传统应用程序，但是如果您需要以 Kubernetes 本地方式开发高可用性、可扩展的存储，那么新发布的 StatefulSet 对象可以代替。我们将在下一节中描述如何使用 StatefulSets 部署 MongoDB。

# 使用 StatefulSets 的 Kubernetes 本地存储

当 Kubernetes 最初开发时，对于复制集中的所有副本都强调了同质性。在这种设计中，没有任何副本具有独立的身份或配置。这要求应用开发者为他们的应用程序确定一个可以建立这种身份的设计。

虽然这种方法为编排系统提供了大量隔离性，但也使得开发有状态应用程序变得相当困难。在社区的大量参与和对各种现有有状态应用程序的大量实验后，StatefulSets 在 Kubernetes 版本 1.5 中被引入。

## StatefulSets 的特性

StatefulSets 是一组复制的 Pod，类似于 ReplicaSets。但与 ReplicaSet 不同，它们具有某些独特的特性：

+   每个副本都会获得一个带有唯一索引的持久主机名（例如 `database-0`，`database-1` 等）。

+   每个副本按照从最低到最高索引的顺序创建，创建过程将暂停，直到前一个索引处的 Pod 健康且可用。这也适用于扩展。

+   当删除 StatefulSet 时，管理的每个副本 Pod 也将按照从最高到最低的顺序删除。这也适用于减少副本数量时的缩容。

原来这简单的要求集使得在 Kubernetes 上部署存储应用程序变得极为简便。例如，稳定的主机名组合（例如 `database-0`）和顺序约束意味着，除了第一个副本外，所有副本都可以可靠地引用 `database-0` 用于发现和建立复制共识。

## 使用 StatefulSets 手动复制的 MongoDB

在本节中，我们将部署一个复制的 MongoDB 集群。现在，复制设置本身将手动完成，以让你感受一下 StatefulSets 的工作方式。最终，我们也将自动化这个设置过程。

首先，我们将使用 StatefulSet 对象创建一个包含三个 MongoDB Pod 的复制集（示例 16-10）。

##### 示例 16-10\. mongo-simple.yaml

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 3
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongodb
        image: mongo:3.4.24
        command:
        - mongod
        - --replSet
        - rs0
        ports:
        - containerPort: 27017
          name: peer
```

正如你所看到的，该定义与我们之前见过的 ReplicaSet 定义类似。唯一的变化在于 `apiVersion` 和 `kind` 字段。

创建 StatefulSet：

```
$ kubectl apply -f mongo-simple.yaml
```

一旦创建完成，ReplicaSet 和 StatefulSet 之间的区别就显而易见了。运行 `kubectl get pods`，你可能会看到如下情况：

```
NAME      READY     STATUS            RESTARTS   AGE
mongo-0   1/1       Running           0          1m
mongo-1   0/1       ContainerCreating 0          10s
```

这与使用 ReplicaSet 的情况有两个重要区别。首先，每个复制的 Pod 都有一个数字索引（`0`，`1`，…），而不是 ReplicaSet 控制器添加的随机后缀。其次，这些 Pod 是按顺序逐步创建的，而不是像 ReplicaSet 那样一次性创建。

创建 StatefulSet 之后，我们还需要创建一个“无头”服务来管理 StatefulSet 的 DNS 条目。在 Kubernetes 中，如果服务没有集群虚拟 IP 地址，则称为“无头”服务。由于 StatefulSets 中每个 Pod 都有唯一的标识，为复制服务创建负载均衡 IP 地址实际上并不合理。您可以使用服务规范中的`clusterIP: None`来创建无头服务（示例 16-11）。

##### 示例 16-11\. mongo-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  ports:
  - port: 27017
    name: peer
  clusterIP: None
  selector:
    app: mongo
```

一旦创建了该服务，通常会填充四个 DNS 条目。像往常一样，会创建`mongo.default.svc.cluster.local`，但与标准服务不同的是，对此主机名进行 DNS 查找会提供 StatefulSet 中所有地址。此外，还会创建`mongo-0.mongo.default.svc.cluster.local`、`mongo-1.mongo`和`mongo-2.mongo`的条目。这些条目分别解析为 StatefulSet 中复制索引的特定 IP 地址。因此，使用 StatefulSets 可以为集合中的每个复制提供明确定义的持久名称。当您配置复制存储解决方案时，这通常非常有用。您可以通过在 Mongo 复制品之一中运行以下命令来查看这些 DNS 条目的效果：

```
$ kubectl run -it --rm --image busybox busybox ping mongo-1.mongo
```

接下来，我们将使用这些逐个 Pod 主机名手动设置 Mongo 复制。我们将选择`mongo-0.mongo`作为我们的初始主节点。在该 Pod 中运行`mongo`工具：

```
$ kubectl exec -it mongo-0 mongo
> rs.initiate( {
  _id: "rs0",
  members:[ { _id: 0, host: "mongo-0.mongo:27017" } ]
 });
 OK
```

这条命令告诉`mongodb`使用`mongo-0.mongo`作为主复制集启动 ReplicaSet `rs0`。

###### 注意

`rs0`的名称是任意的。您可以使用任何您喜欢的名称，但在*mongo-simple.yaml* StatefulSet 定义中也需要进行更改。

一旦初始化了 Mongo ReplicaSet，您可以在`mongo-0.mongo` Pod 上的`mongo`工具中运行以下命令添加其余的副本：

```
> rs.add("mongo-1.mongo:27017");
> rs.add("mongo-2.mongo:27017");
```

正如您所看到的，我们正在使用特定于复制品的 DNS 名称将它们添加为 Mongo 集群中的副本。在此时，我们已经完成了。我们的复制 MongoDB 已经运行起来了。但实际上并没有像我们希望的那样自动化—在下一节中，我们将看到如何使用脚本来自动设置。

## 自动化 MongoDB 集群创建

为了自动化基于 StatefulSet 的 MongoDB 集群的部署，我们将向我们的 Pods 添加一个容器来执行初始化。为了配置此 Pod 而无需构建新的 Docker 镜像，我们将使用 ConfigMap 将脚本添加到现有的 MongoDB 镜像中。

我们将使用“初始化容器”来运行此脚本。“初始化容器”（或“init”容器）是在 Pod 启动时运行一次的专用容器。通常用于像这样需要做少量设置工作以便主应用程序运行的情况。在 Pod 定义中，有一个单独的`initContainers`列表，可以在其中定义 init 容器。这里提供了一个示例：

```
...
      initContainers:
      - name: init-mongo
        image: mongo:3.4.24
        command:
        - bash
        - /config/init.sh
        volumeMounts:
        - name: config
          mountPath: /config
 ...
      volumes:
      - name: config
        configMap:
          name: "mongo-init"
```

注意，它正在挂载一个名为 `mongo-init` 的 ConfigMap 卷，该 ConfigMap 包含执行我们初始化的脚本。首先，脚本确定它是否正在 `mongo-0` 上运行。如果它在不同的 Mongo 副本上，则等待 ReplicaSet 存在，然后将自身注册为该 ReplicaSet 的成员。

示例 16-12 包含完整的 ConfigMap 对象。

##### 示例 16-12\. mongo-configmap.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-init
data:
  init.sh: |
    #!/bin/bash

    # Need to wait for the readiness health check to pass so that the
    # Mongo names resolve. This is kind of wonky.
    until ping -c 1 ${HOSTNAME}.mongo; do
      echo "waiting for DNS (${HOSTNAME}.mongo)..."
      sleep 2
    done

    until /usr/bin/mongo --eval 'printjson(db.serverStatus())'; do
      echo "connecting to local mongo..."
      sleep 2
    done
    echo "connected to local."

    HOST=mongo-0.mongo:27017

    until /usr/bin/mongo --host=${HOST} --eval 'printjson(db.serverStatus())'; do
      echo "connecting to remote mongo..."
      sleep 2
    done
    echo "connected to remote."

    if [[ "${HOSTNAME}" != 'mongo-0' ]]; then
      until /usr/bin/mongo --host=${HOST} --eval="printjson(rs.status())" \
            | grep -v "no replset config has been received"; do
        echo "waiting for replication set initialization"
        sleep 2
      done
      echo "adding self to mongo-0"
      /usr/bin/mongo --host=${HOST} \
         --eval="printjson(rs.add('${HOSTNAME}.mongo'))"
    fi

    if [[ "${HOSTNAME}" == 'mongo-0' ]]; then
      echo "initializing replica set"
      /usr/bin/mongo --eval="printjson(rs.initiate(\
          {'_id': 'rs0', 'members': [{'_id': 0, \
           'host': 'mongo-0.mongo:27017'}]}))"
    fi
    echo "initialized"
```

这个脚本会立即退出，这在使用 `initContainers` 时非常重要。每个初始化容器都会等待前一个容器完成后再运行。主应用容器会等待所有初始化容器都完成。如果这个脚本没有退出，主 Mongo 服务器将永远无法启动。

综合起来，示例 16-13 是使用 ConfigMap 的完整 StatefulSet。

##### 示例 16-13\. mongo.yaml

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 3
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongodb
        image: mongo:3.4.24
        command:
        - mongod
        - --replSet
        - rs0
        ports:
        - containerPort: 27017
          name: web
      # This container initializes the MongoDB server, then sleeps.
      - name: init-mongo
        image: mongo:3.4.24
        command:
        - bash
        - /config/init.sh
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: "mongo-init"
```

给定所有这些文件，你可以创建一个 Mongo 集群：

```
$ kubectl apply -f mongo-config-map.yaml
$ kubectl apply -f mongo-service.yaml
$ kubectl apply -f mongo-simple.yaml
```

或者，如果你愿意，你可以将它们全部合并到一个单独的 YAML 文件中，其中各个对象由 `---` 分隔。确保保持相同的顺序，因为 StatefulSet 定义依赖于存在 ConfigMap 定义。

## 持久卷和 StatefulSets

对于持久存储，你需要将持久卷挂载到 */data/db* 目录中。在 Pod 模板中，你需要更新它以将持久卷声明挂载到该目录：

```
...
        volumeMounts:
        - name: database
          mountPath: /data/db
```

尽管这种方法与我们看到的可靠的单例方法类似，因为 StatefulSet 复制了多个 Pod，你不能简单地引用一个持久卷声明。相反，你需要添加一个 *持久卷声明模板*。你可以将声明模板视为与 Pod 模板相同，但它不是创建 Pod，而是创建卷声明。你需要将以下内容添加到 StatefulSet 定义的底部：

```
  volumeClaimTemplates:
  - metadata:
      name: database
      annotations:
        volume.alpha.kubernetes.io/storage-class: anything
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Gi
```

当你向 StatefulSet 定义添加一个卷声明模板时，每当 StatefulSet 控制器创建 StatefulSet 的一部分的 Pod 时，它将根据该模板创建一个持久卷声明作为该 Pod 的一部分。

###### 注意

要使这些复制的持久卷正常工作，你需要设置持久卷的自动配置或预先填充一个持久卷对象集合供 StatefulSet 控制器使用。如果没有可以创建的声明，StatefulSet 控制器将无法创建相应的 Pod。

## 最后一点：就绪探针

在生产环境中配置我们的 MongoDB 集群的最后一步是为我们的 Mongo 服务容器添加活性检查。正如我们在“健康检查”中学到的那样，活性探针用于确定容器是否正常运行。

对于存活性检查，我们可以通过将以下内容添加到 StatefulSet 对象中的 Pod 模板来使用`mongo`工具本身：

```
...
 livenessProbe:
   exec:
     command:
     - /usr/bin/mongo
     - --eval
     - db.serverStatus()
   initialDelaySeconds: 10
   timeoutSeconds: 10
 ...
```

# 摘要

一旦我们结合了 StatefulSets、持久卷索赔和存活探测，我们就在 Kubernetes 上运行一个经过硬化、可扩展的云原生 MongoDB 安装。虽然这个示例涉及 MongoDB，但创建 StatefulSets 来管理其他存储解决方案的步骤非常类似，可以遵循相似的模式。
