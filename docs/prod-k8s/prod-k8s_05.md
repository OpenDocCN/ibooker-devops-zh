# 第四章：容器存储

虽然 Kubernetes 最初是在无状态工作负载领域发展起来的，但运行有状态服务变得越来越普遍。甚至像数据库和消息队列这样复杂的有状态工作负载也在 Kubernetes 集群中找到了应用的方式。为了支持这些工作负载，Kubernetes 需要提供超越临时选项的存储能力。也就是说，系统需要在应用崩溃或工作负载被重新调度到不同主机时提供增强的韧性和可用性。

在本章中，我们将探讨我们的平台如何为应用程序提供存储服务。我们将首先讨论应用程序持久性和存储系统期望的关键问题，然后再深入讨论 Kubernetes 中可用的存储原语。随着我们探讨更高级的存储需求，我们将看到 [容器存储接口 (CSI)](https://kubernetes-csi.github.io/docs) 的使用，它使我们能够与各种存储提供者进行集成。最后，我们将探讨使用 CSI 插件为我们的应用程序提供自助存储。

###### 注意

存储本身就是一个广阔的主题。我们的意图是为您提供足够的细节，以便您能够为工作负载做出明智的存储决策。如果存储不是您的背景，强烈建议与您的基础设施/存储团队共同讨论这些概念。Kubernetes 不能取代您组织中对存储专业知识的需求！

# 存储考虑事项

在深入研究 Kubernetes 存储模式和选项之前，我们应该退后一步，分析潜在的存储需求周围的一些关键考虑因素。在基础设施和应用程序级别，思考以下需求是非常重要的。

+   访问模式

+   卷扩展

+   动态供给

+   备份和恢复

+   块、文件和对象存储

+   临时数据

+   选择提供者

## 访问模式

应用程序可以支持三种访问模式：

ReadWriteOnce (RWO)

单个 Pod 可以对卷进行读写操作。

ReadOnlyMany (ROX)

多个 Pod 可以读取卷。

ReadWriteMany (RWX)

多个 Pod 可以对卷进行读写操作。

对于云原生应用程序，RWO 模式是最常见的模式。当利用常见的提供者如 [Amazon Elastic Block Storage (EBS)](https://aws.amazon.com/ebs) 或 [Azure Disk Storage](https://oreil.ly/wAtBg) 时，你只能选择 RWO，因为磁盘只能附加到一个节点。虽然这种限制可能看起来有问题，但大多数云原生应用程序在这种存储环境中表现最佳，因为卷是独占的，提供高性能的读写能力。

我们经常发现遗留应用程序需要 RWX 的需求。通常，它们被设计为假定可以访问[网络文件系统（NFS）](https://oreil.ly/OrsBR)。当服务需要共享状态时，通常有比通过 NFS 共享数据更优雅的解决方案；例如，使用消息队列或数据库。此外，如果应用程序希望共享数据，通常最好通过 API 公开此数据，而不是授予访问其文件系统的权限。这使得许多情况下对于 RWX 的使用都显得可疑。除非 NFS 是正确的设计选择，否则平台团队可能面临是否提供兼容 RWX 的存储或要求其开发人员重新架构应用程序的艰难选择。如果决定支持 ROX 或 RWX 是必需的，那么可以集成几个提供者，如[亚马逊弹性文件系统（EFS）](https://aws.amazon.com/efs)和[Azure 文件共享](https://oreil.ly/u6HiQ)。

## 卷扩展

随着时间的推移，应用程序可能会开始填满其卷。这可能会带来挑战，因为用更大的卷替换卷将需要数据迁移。其中一个解决方案是支持卷扩展。从像 Kubernetes 这样的容器编排器的角度来看，这涉及到几个步骤：

1.  从编排器（例如，通过 PersistentVolumeClaim）请求额外的存储。

1.  通过存储提供者扩展卷的大小。

1.  扩展文件系统以利用更大的卷。

一旦完成，Pod 将能够访问额外的空间。这个功能取决于我们选择的存储后端以及 Kubernetes 中的集成是否能够促成前面的步骤。本章稍后我们将探讨一个卷扩展的示例。

## 卷配置

您可以选择两种配置模型：动态配置和静态配置。静态配置假定卷是在节点上为 Kubernetes 创建的。动态配置是指在集群内运行驱动程序，并可以通过与存储提供者通信来满足工作负载的存储请求。在这两种模型中，如果可能的话，动态配置是首选。通常，选择两者之间的一个取决于您的底层存储系统是否有兼容 Kubernetes 的驱动程序。我们稍后会深入讨论这些驱动程序。

## 备份与恢复

备份是存储的最复杂方面之一，尤其是自动恢复是一个要求。一般来说，备份是数据的副本，用于在数据丢失时使用。通常，我们根据存储系统的可用性保证平衡备份策略。例如，尽管备份始终重要，但在我们的存储系统具有复制保证的情况下，它们的重要性较低，硬件丢失*不*会导致数据丢失。另一个考虑因素是应用程序可能需要不同的程序来促进备份和恢复。通常，我们可以随时备份整个集群并进行恢复的想法是一个天真的观点，或者至少需要大量的工程努力才能实现。

决定由谁负责应用程序的备份和恢复可能是组织内最具挑战性的讨论之一。可以说，将恢复功能作为平台服务提供可能是一个“好处”。然而，当我们涉及到应用程序特定的复杂性时，这可能会引发问题，例如当一个应用程序无法重新启动并且需要进行只有开发人员知道的操作时。

[Project Velero](https://velero.io) 是 Kubernetes 状态和应用状态的最受欢迎的备份解决方案之一。如果您希望在集群之间迁移或恢复它们，Velero 可以备份 Kubernetes 对象。此外，Velero 支持调度卷快照。在本章节深入讨论卷快照时，我们会了解到，安排和管理快照的能力并*不*是自动完成的。更进一步，我们通常提供快照基元，但需要定义围绕它们的编排流程。最后，Velero 支持备份和恢复钩子。这些钩子允许我们在执行备份或恢复之前在容器中运行命令。例如，某些应用可能需要在进行备份之前停止流量或触发刷新。Velero 通过钩子使这成为可能。

## 块设备和文件与对象存储

我们的应用程序期望的存储类型对选择适当的底层存储和 Kubernetes 集成至关重要。应用程序使用的最常见的存储类型是文件存储。文件存储是一个带有文件系统的块设备。这使得应用程序可以按照我们在任何操作系统上熟悉的方式写入文件。

在文件系统的底层是一个块设备。与在其上建立文件系统不同，我们可以提供设备，以便应用程序可以直接与原始块通信。文件系统在写入数据时固有地增加了开销。在现代软件开发中，我们很少会担心文件系统的开销。但是，如果您的用例需要直接与原始块设备进行交互，某些存储系统可以支持这一点。

最后一种存储类型是对象存储。对象存储在文件的意义上有所偏离，因为它没有传统的层次结构。对象存储使开发人员能够获取非结构化数据，为其添加唯一标识符，添加一些元数据，并存储它。云提供商的对象存储，例如[Amazon S3](https://aws.amazon.com/s3)，已成为组织托管图像、二进制文件等的流行位置。这种流行度得益于其功能齐全的 Web API 和访问控制。对象存储通常由应用程序本身进行交互，应用程序使用库进行身份验证和与提供程序的交互。由于对象存储的交互接口标准化程度较低，因此不常见将其集成为应用程序可以透明交互的平台服务。

## 短暂数据

虽然存储可能意味着超出 Pod 生命周期的持久性水平，但支持短暂数据使用的有效用例是存在的。默认情况下，写入自己文件系统的容器将利用短暂存储。如果容器重新启动，这些存储将丢失。[emptyDir](https://oreil.ly/86zjA) 卷类型用于能够抵御重新启动的短暂存储。这不仅对容器重新启动具有韧性，而且可以用于在同一 Pod 中的容器之间共享文件。

短暂数据的最大风险是确保您的 Pod 不会占用主机存储容量过多。尽管每个 Pod 的 4Gi 数字可能看起来并不多，但考虑一个节点可以运行数百个，有时甚至数千个 Pod。Kubernetes 支持限制命名空间中的 Pod 可用短暂存储的累计量。这些问题的配置在第十二章中有详细介绍。

## 选择存储提供者

可供选择的存储提供者众多。选项从您可能自行管理的存储解决方案，例如 Ceph，到像 Google Persistent Disk 或 Amazon Elastic Block Store 这样的完全托管系统。选项的差异远远超出了本书的范围。然而，我们建议了解存储系统的能力以及这些能力与 Kubernetes 容易集成的情况。这将使您能够评估一种解决方案相对于另一种解决方案在满足应用程序需求方面的表现。此外，在可能的情况下，如果您正在管理自己的存储系统，请考虑使用您已经具有操作经验的内容。将 Kubernetes 引入新的存储系统将为您的组织增加大量新的运营复杂性。

# Kubernetes 存储基元

Kubernetes 提供了多个原语来支持工作负载存储。这些原语为我们提供了构建复杂存储解决方案所需的基础。在本节中，我们将使用一个例子来介绍 PersistentVolumes、PersistentVolumeClaims 和 StorageClasses，将快速预配置的存储分配给容器。

## 持久卷和声明

在 Kubernetes 中，卷和声明是存储的基础。这些通过 [PersistentVolume](https://oreil.ly/7_OAz) 和 [PersistentVolumeClaim](https://oreil.ly/PKtAr) API 公开。PersistentVolume 资源表示 Kubernetes 中已知的存储卷。假设管理员已经准备好一个节点，提供了 30Gi 的快速本地存储在 */mnt/fast-disk/pod-0*。为了在 Kubernetes 中表示这个卷，管理员可以创建一个 PersistentVolume 对象：

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0
spec:
  capacity:
    storage: 30Gi ![1](img/1.png)
  volumeMode: Filesystem ![2](img/2.png)
  accessModes:
  - ReadWriteOnce ![3](img/3.png)
  storageClassName: local-storage ![4](img/4.png)
  local:
    path: /mnt/fast-disk/pod-0
  nodeAffinity: ![5](img/5.png)
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - test-w
```

![1](img/#co_container_storage_CO1-1)

此卷中可用的存储量。用于确定声明是否能绑定到此卷。

![2](img/#co_container_storage_CO1-2)

指定卷是 [块设备](https://oreil.ly/mrHwE) 还是文件系统。

![3](img/#co_container_storage_CO1-3)

指定卷的访问模式。包括 `ReadWriteOnce`、`ReadMany` 和 `ReadWriteMany`。

![4](img/#co_container_storage_CO1-4)

将此卷与存储类关联，用于将最终的声明与此卷配对。

![5](img/#co_container_storage_CO1-5)

标识应该将此卷关联到哪个节点。

如您所见，PersistentVolume 包含有关卷实现的详细信息。为了提供更高一层的抽象，引入了 PersistentVolumeClaim，它根据其请求绑定到适当的卷。通常情况下，这将由应用团队定义，并添加到他们的 Namespace，并从他们的 Pod 引用：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc0
spec:
  storageClassName: local-storage ![1](img/1.png)
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 30Gi ![2](img/2.png)
---
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: fast-disk
      persistentVolumeClaim:
        claimName: pvc0 ![3](img/3.png)
  containers:
    - name: ml-processer
      image: ml-processer-image
      volumeMounts:
        - mountPath: "/var/lib/db"
          name: fast-disk
```

![1](img/#co_container_storage_CO2-1)

检查是否有一个类为 `local-storage`，访问模式为 `ReadWriteOnce` 的卷。

![2](img/#co_container_storage_CO2-2)

绑定到一个具有 >= `30Gi` 存储量的卷。

![3](img/#co_container_storage_CO2-3)

声明此 Pod 作为 PersistentVolumeClaim 的消费者。

基于 PersistentVolume 的 `nodeAffinity` 设置，Pod 将自动调度到具有此卷的主机上。开发者无需额外配置亲和性。

这个过程展示了管理员如何将这些存储提供给开发人员的非常手动的流程。我们称之为静态配置。通过适当的自动化，这可以成为向 Pod 暴露快速磁盘的可行方式。例如，可以部署[本地持久卷静态提供程序](https://oreil.ly/YiQ0G)到集群中，以检测预分配的存储并自动将其暴露为持久卷。它还提供一些生命周期管理功能，例如在销毁持久卷索赔时删除数据。

###### 警告

有多种方法可以实现本地存储，这可能导致不良实践。例如，允许开发人员使用[hostPath](https://oreil.ly/PAU8Y)似乎很有吸引力，而不需要预先配置本地存储。`hostPath` 允许您指定绑定到主机的路径，而不必使用 PersistentVolume 和 PersistentVolumeClaim。这可能是一个巨大的安全风险，因为它允许开发人员绑定到主机上的目录，这可能会对主机和其他 Pod 产生负面影响。如果您希望为开发人员提供能够经受 Pod 重启但不能承受 Pod 被删除或移动到不同节点的临时存储，可以使用[EmptyDir](https://oreil.ly/mPwBg)。这将在由 Kube 管理的文件系统中分配存储，并对 Pod 透明。

## 存储类

在许多环境中，期望节点提前准备好磁盘和卷是不现实的。这些情况通常需要动态配置，可以根据我们索赔的需要提供卷。为了促进这种模型，我们可以向开发人员提供存储类。这些使用[StorageClass](https://oreil.ly/MoG_T) API 定义。假设您的集群在 AWS 上运行，并且想要动态地向 Pod 提供 EBS 卷，可以添加以下 StorageClass：

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-standard ![1](img/1.png)
  annotations:
    storageclass.kubernetes.io/is-default-class: true ![2](img/2.png)
provisioner: kubernetes.io/aws-ebs ![3](img/3.png)
parameters: ![4](img/4.png)
  type: io2
  iopsPerGB: "17"
  fsType: ext4
```

![1](img/#co_container_storage_CO3-1)

可以从索赔引用的 StorageClass 名称。

![2](img/#co_container_storage_CO3-2)

将此 StorageClass 设置为默认值。如果索赔未指定类，则将使用此值。

![3](img/#co_container_storage_CO3-3)

使用`aws-ebs`提供程序根据索赔创建卷。

![4](img/#co_container_storage_CO3-4)

用于配置特定于提供程序的卷配置。

通过提供多个 StorageClass，您可以向开发人员提供各种存储选项。这包括在单个集群中支持多个提供者，例如同时运行 Ceph 和 VMware vSAN。或者，您可以通过同一提供者提供不同层次的存储。例如，提供更便宜的存储选项以及更昂贵的选项。不幸的是，Kubernetes 缺乏细粒度的控制来限制开发人员可以请求的类别。可以实现控制作为验证入场控制，这在第八章中有详细介绍。

Kubernetes 提供各种各样的提供者，包括 AWS EBS、Glusterfs、GCE PD、Ceph RBD 等。从历史上看，这些提供者是在内核中实现的。这意味着存储提供者需要在核心 Kubernetes 项目中实现其逻辑。然后，此代码将与相关的 Kubernetes 控制平面组件一起发布。

这种模型存在几个缺点。首先，存储提供者无法独立管理。对提供者的所有更改都必须与 Kubernetes 发布版本绑定。此外，每个 Kubernetes 部署都会包含不必要的代码。例如，运行 AWS 的集群仍包含用于与 GCE PDs 交互的提供者代码。很快就显而易见，将这些提供者集成外部化并弃用内核功能具有高度价值。[FlexVolume 驱动程序](https://oreil.ly/YnnCq)是最初旨在解决这一问题的一种基于外部实现规范。然而，FlexVolumes 已进入维护模式，以支持我们接下来要讨论的容器存储接口（CSI）。

# 容器存储接口（CSI）

容器存储接口是我们如何为工作负载提供块和文件存储的答案。CSI 的实现被称为驱动程序，这些驱动程序具有与存储提供者通信的操作知识。这些提供者涵盖从诸如[Google 持久磁盘](https://cloud.google.com/persistent-disk)等云系统到由您部署和管理的存储系统（例如[Ceph](https://ceph.io)）。这些驱动程序由存储提供者在项目中实现，这些项目独立于集群之外完全管理。

在高层次上，CSI 的实现包括控制器插件和节点插件。CSI 驱动程序开发人员在实现这些组件时具有很大的灵活性。通常，实现会将控制器和节点插件捆绑在同一个二进制文件中，并通过环境变量（例如 `X_CSI_MODE`）启用任一模式。唯一的期望是驱动程序在 kubelet 中注册，并实现 CSI 规范中的端点。

控制器服务负责管理存储提供程序中卷的创建和删除。此功能扩展到（可选）功能，例如获取卷快照和扩展卷。节点服务负责准备卷以供节点上的 Pod 使用。通常意味着设置挂载点并报告有关节点上卷的信息。节点和控制器服务还实现了报告插件信息、能力以及插件是否健康的身份服务。因此，图 4-1 描述了部署了这些组件的集群架构。

![prku 0401](img/prku_0401.png)

###### 图 4-1\. 集群运行 CSI 插件。驱动程序以节点和控制器模式运行。控制器通常作为 Deployment 运行。节点服务部署为 DaemonSet，在每个主机上放置一个 Pod。

让我们更深入地了解这两个组件，即控制器和节点。

## CSI 控制器

CSI 控制器服务提供管理持久存储系统中卷的 API。Kubernetes 控制平面 *不直接* 与 CSI 控制器服务交互。相反，由 Kubernetes 存储社区维护的控制器会对 Kubernetes 事件做出反应，并将其转换为 CSI 指令，例如在创建新的 PersistentVolumeClaim 时的 CreateVolumeRequest。由于 CSI 控制器服务通过 UNIX sockets 公开其 API，控制器通常作为 CSI 控制器服务的 sidecar 部署。有多个外部控制器，每个控制器具有不同的行为：

external-provisioner

创建 PersistentVolumeClaims 时，请求从 CSI 驱动程序创建卷。一旦在存储提供程序中创建卷，此提供程序将在 Kubernetes 中创建 PersistentVolume 对象。

external-attacher

监视 VolumeAttachment 对象，声明应将卷附加到节点或从节点分离。向 CSI 驱动程序发送附加或分离请求。

external-resizer

检测 PersistentVolumeClaims 中存储大小的变化。向 CSI 驱动程序发送扩展请求。

external-snapshotter

创建 VolumeSnapshotContent 对象时，会向驱动程序发送快照请求。

###### 注意

在实现 CSI 插件时，开发者并非必须使用上述控制器。但是，鼓励使用这些控制器，以防止在每个 CSI 插件中重复逻辑。

## CSI 节点

节点插件通常运行与控制器插件相同的驱动程序代码。但是，“节点模式”中的运行意味着专注于任务，例如挂载附加的卷、建立其文件系统以及将卷挂载到 Pod。这些行为的请求通过 kubelet 完成。除了驱动程序外，通常还在 Pod 中包括以下 sidecars：

node-driver-registrar

发送 [注册请求](https://oreil.ly/kmkJh) 到 kubelet 以使其意识到 CSI 驱动程序。

活动探针

报告 CSI 驱动程序的健康状况。

# 实施存储即服务

现在我们已经涵盖了应用程序存储的关键考虑因素，Kubernetes 中可用的存储原语以及使用 CSI 进行驱动程序集成。现在是时候将这些想法结合起来，看一看提供开发人员存储即服务的实现。我们希望以声明方式提供请求存储并使其可用于工作负载的方法。我们还希望动态实现这一点，不需要管理员预先配置和附加卷。相反，我们希望根据工作负载的需求随需提供。

为了开始这个实现，我们将使用亚马逊网络服务（AWS）。此示例集成了 AWS 的 [弹性块](https://oreil.ly/I4VVw) 存储系统。如果您选择的提供商不同，这些内容的大部分仍然相关！我们只是使用这个提供商作为所有部分如何组合在一起的具体示例。

接下来我们将深入探讨集成/驱动程序的安装，向开发人员公开存储选项，使用工作负载消耗存储空间，调整卷大小以及获取卷快照。

## 安装

安装过程相对直接，主要包括两个关键步骤：

1.  配置访问提供商。

1.  将驱动程序组件部署到集群中。

在本例中，供应商 AWS 将要求驱动程序标识自身，确保其具有适当的访问权限。在这种情况下，我们有三个可选项可供选择。其中一种方法是更新 [实例配置文件](https://oreil.ly/fGWYd) 的 Kubernetes 节点。这将使我们不必担心 Kubernetes 级别的凭证，但将为能够访问 AWS API 的工作负载提供通用权限。第二种，也可能是最安全的选择是引入一个身份服务，可以为特定工作负载提供 IAM 权限。一个示例项目是 [kiam](https://github.com/uswitch/kiam)。这种方法在 第十章 中有详细讲解。最后，您可以在挂载到 CSI 驱动程序的密钥中添加凭证。在这种模型中，密钥将如下所示：

```
apiVersion: v1
kind: Secret
metadata:
  name: aws-secret
  namespace: kube-system
stringData:
  key_id: "AKIAWJQHICPELCJVKYNU"
  access_key: "jqWi1ut4KyrAHADIOrhH2Pd/vXpgqA9OZ3bCZ"
```

###### 警告

该帐户将有权访问操作底层存储系统。应仔细管理对此密钥的访问。有关更多信息，请参阅 第七章。

有了这样的配置，CSI 组件可以安装。首先，控制器作为一个 Deployment 安装。在运行多个副本时，它将使用领导选举确定哪个实例应处于活动状态。然后安装节点插件，它作为 DaemonSet 形式在每个节点上运行一个 Pod。初始化后，节点插件的实例将向其 kubelet 注册。kubelet 然后通过为每个 Kubernetes 节点创建一个 CSINode 对象来报告启用 CSI 的节点。一个三节点集群的输出如下：

```
$ kubectl get csinode

NAME                                       DRIVERS   AGE
ip-10-0-0-205.us-west-2.compute.internal   1         97m
ip-10-0-0-224.us-west-2.compute.internal   1         79m
ip-10-0-0-236.us-west-2.compute.internal   1         98m
```

如我们所见，列出了三个节点，每个节点上注册了一个驱动程序。检查一个 CSINode 的 YAML 文件揭示了以下内容：

```
apiVersion: storage.k8s.io/v1
kind: CSINode
metadata:
  name: ip-10-0-0-205.us-west-2.compute.internal
spec:
  drivers:
    - allocatable:
        count: 25 ![1](img/1.png)
      name: ebs.csi.aws.com
      nodeID: i-0284ac0df4da1d584
      topologyKeys:
        - topology.ebs.csi.aws.com/zone ![2](img/2.png)
```

![1](img/#co_container_storage_CO4-1)

最大允许在此节点上的卷数。

![2](img/#co_container_storage_CO4-2)

当选择节点用于工作负载时，此值将传递给 CreateVolumeRequest，以便驱动程序知道*在哪里*创建卷。对于存储系统而言，集群中的节点无法访问相同的存储是很重要的。例如，在 AWS 中，当 Pod 被调度到可用区时，卷必须在相同的区域创建。

另外，驱动程序已在集群中正式注册。详细信息可以在 CSIDriver 对象中找到：

```
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata:
  name: aws-ebs-csi-driver ![1](img/1.png)
  labels:
    app.kubernetes.io/name: aws-ebs-csi-driver
spec:
  attachRequired: true ![2](img/2.png)
  podInfoOnMount: false ![3](img/3.png)
  volumeLifecycleModes:
  - Persistent ![4](img/4.png)
```

![1](img/#co_container_storage_CO5-1)

代表此驱动程序的提供者名称。此名称将绑定到我们为平台用户提供的存储类。

![2](img/#co_container_storage_CO5-2)

指定必须在挂载卷之前完成附加操作。

![3](img/#co_container_storage_CO5-3)

在设置挂载时，不需要将 Pod 元数据传递为上下文。

![4](img/#co_container_storage_CO5-4)

永久卷的默认模型。[内联支持](https://oreil.ly/Z_pDY)可以通过将此选项设置为`Ephemeral`来启用。在短暂模式中，存储预期只持续与 Pod 一样长的时间。

到目前为止，我们探讨的设置和对象都是我们引导过程的产物。CSIDriver 对象使得可以更轻松地发现驱动程序的详细信息，并且包含在驱动程序的部署包中。CSINode 对象由 kubelet 管理。通用的注册器旁路进程包含在节点插件 Pod 中，并从 CSI 驱动程序获取详细信息，并将驱动程序注册到 kubelet 中。然后 kubelet 报告每个主机上可用的 CSI 驱动程序数量。Figure 4-2 展示了这个引导过程。

![prku 0402](img/prku_0402.png)

###### 图 4-2\. CSIDriver 对象已部署并包含在包中，同时节点插件与 kubelet 注册。这反过来创建/管理 CSINode 对象。

## 暴露存储选项

为了为开发者提供存储选项，我们需要创建 StorageClasses。对于这种情况，我们假设有两种类型的存储我们想要暴露。第一个选项是提供便宜的磁盘，可以用于工作负载持久性需求。许多时候，应用程序不需要 SSD，因为它们只是持久化一些不需要快速读写的文件。因此，便宜的磁盘（HDD）将是默认选项。然后，我们希望提供更快的 SSD，每 GB 配置一个定制的[IOPS](https://oreil.ly/qXMcQ)。Table 4-1 展示了我们的提供；价格反映了本文撰写时的 AWS 成本。

Table 4-1\. 存储提供

| 提供名称 | 存储类型 | 每卷最大吞吐量 | AWS 成本 |
| --- | --- | --- | --- |
| default-block | HDD（优化） | 40–90 MB/s | $0.045 每 GB 每月 |
| performance-block | SSD（io1） | ~1000 MB/s | $0.125 每 GB 每月 + $0.065 每预配的 IOPS 每月 |

为了创建这些产品，我们将为每个创建一个存储类。在每个存储类内部有一个`parameters`字段。这是我们可以配置以满足 Table 4-1 中功能的设置的地方。

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: default-block ![1](img/1.png)
  annotations:
    storageclass.kubernetes.io/is-default-class: "true" ![2](img/2.png)
provisioner: ebs.csi.aws.com ![3](img/3.png)
allowVolumeExpansion: true ![4](img/4.png)
volumeBindingMode: WaitForFirstConsumer ![5](img/5.png)
parameters:
  type: st1 ![6](img/6.png)
---
kind: StorageClass ![7](img/7.png)
apiVersion: storage.k8s.io/v1
metadata:
  name: performance-block
provisioner: ebs.csi.aws.com
parameters:
  type: io1
  iopsPerGB: "20"
```

![1](img/#co_container_storage_CO6-1)

这是我们为平台用户提供的存储提供的名称。它将被从 PeristentVolumeClaims 引用。

![2](img/#co_container_storage_CO6-2)

这将设置提供作为默认选项。如果创建 PersistentVolumeClaim 时未指定 StorageClass，则将使用`default-block`。

![3](img/#co_container_storage_CO6-3)

映射到应执行哪个 CSI 驱动程序。

![4](img/#co_container_storage_CO6-4)

允许通过更改 PersistentVolumeClaim 来扩展卷大小。

![5](img/#co_container_storage_CO6-5)

在 Pod 消耗 PersistentVolumeClaim 之前不要预留卷。这将确保在安排的 Pod 的适当可用性区域中创建卷。它还防止了孤立的 PVC 在 AWS 中创建卷，这些卷将会被计费。

![6](img/#co_container_storage_CO6-6)

指定驱动程序应获取什么类型的存储以满足要求。

![7](img/#co_container_storage_CO6-7)

第二类，调整为高性能 SSD。

## 消费存储

在前面的部分准备好之后，我们现在可以让用户使用这些不同类别的存储了。我们将首先查看开发人员请求存储时的体验。然后我们将详细介绍如何满足这些请求。首先，让我们看看当列出可用的 StorageClasses 时开发者会得到什么：

```
$ kubectl get storageclasses.storage.k8s.io

NAME                      PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE
default-block (default)   ebs.csi.aws.com   Delete          Immediate
performance-block         ebs.csi.aws.com   Delete          WaitForFirstConsumer

ALLOWVOLUMEEXPANSION
true
true
```

###### 警告

通过允许开发人员创建 PVCs，我们将允许他们引用*任何*StorageClass。如果这是一个问题，您可能希望考虑实施验证入场控制来评估请求是否合适。这个主题在 Chapter 8 中有详细介绍。

假设开发者想要为一个应用程序提供更便宜的 HDD 和更高性能的 SSD。在这种情况下，将创建两个 PersistentVolumeClaims。我们将分别称之为`pvc0`和`pvc1`：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc0 ![1](img/1.png)
spec:
  resources:
    requests:
      storage: 11Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  resources:
    requests:
      storage: 14Gi
  storageClassName: performance-block ![2](img/2.png)
```

![1](img/#co_container_storage_CO7-1)

这将使用默认存储类(`default-block`)，并假设其他默认设置如 RWO 和文件系统存储类型。

![2](img/#co_container_storage_CO7-2)

确保向驱动程序请求`performance-block`而不是`default-block`。

根据 StorageClass 的设置，这两个将表现出不同的供应行为。高性能存储（来自`pvc1`）在 AWS 中被创建为一个未附加的卷。这种卷可以快速附加并且随时可用。默认存储（来自`pv0`）将处于`Pending`状态，集群会等待直到一个 Pod 使用 PVC 来在 AWS 中配置存储。虽然当一个 Pod 最终使用声明时，这将需要更多的工作来配置，但您不会因为未使用的存储而被计费！Kubernetes 中声明与 AWS 中卷之间的关系可以在 图 4-3 中看到。

![prku 0403](img/prku_0403.png)

###### 图 4-3\. `pv1` 在 AWS 中被配置为一个卷，并且 CSIVolumeName 被传播以便于关联。直到一个 Pod 引用它之前，`pv0` 将不会创建相应的卷。

现在假设开发者创建了两个 Pod。一个 Pod 引用`pv0`，而另一个引用`pv1`。一旦每个 Pod 被调度到一个节点上，卷将附加到该节点以供使用。对于`pv0`，在此之前还需要在 AWS 中创建卷。当 Pod 被调度并且卷被附加后，文件系统就建立起来，存储就被挂载到容器中。由于这些是持久卷，我们现在引入了一种模式，即使 Pod 被重新调度到另一个节点，卷也可以随之移动。我们已经在 图 4-4 中展示了我们如何满足自服务存储请求的端到端流程。

![prku 0404](img/prku_0404.png)

###### 图 4-4\. 驱动程序和 Kubernetes 协作满足存储请求的端到端流程。

###### 注意

事件在调试与 CSI 的存储交互中特别有帮助。因为为了满足 PVC，需要按顺序进行供应、附加和挂载，您应该查看这些对象上的事件，以便不同组件报告它们所做的工作。使用`kubectl describe -n $NAMESPACE pvc $PVC_NAME`可以轻松查看这些事件。

## 调整大小

调整大小是`aws-ebs-csi-driver`中支持的特性。在大多数 CSI 实现中，`external-resizer`控制器用于检测持久卷声明对象的更改。当检测到大小变化时，它会转发给驱动程序，驱动程序将扩展卷。在这种情况下，运行在控制器插件中的驱动程序将利用 AWS EBS API 进行扩展。

扩展 EBS 卷后，新空间*不*立即可用于容器。这是因为文件系统仍然仅占用原始空间。为了扩展文件系统，我们需要等待节点插件驱动实例扩展文件系统。所有这些操作都可以在*不*终止 Pod 的情况下完成。文件系统的扩展可以在节点插件的 CSI 驱动程序的以下日志中看到：

```
mount_linux.go: Attempting to determine if disk "/dev/nvme1n1" is formatted
using blkid with args: ([-p -s TYPE -s PTTYPE -o export /dev/nvme1n1])

mount_linux.go: Output: "DEVNAME=/dev/nvme1n1\nTYPE=ext4\n", err: <nil>

resizefs_linux.go: ResizeFS.Resize - Expanding mounted volume /dev/nvme1n1

resizefs_linux.go: Device /dev/nvme1n1 resized successfully
```

###### 警告

Kubernetes 不支持缩小 PVC 的 size 字段。除非 CSI 驱动程序提供此类解决方法，否则您可能无法在不重新创建卷的情况下缩小卷的大小。在扩展卷时，请牢记这一点。

## 快照

为了方便容器使用的卷数据的定期备份，提供了快照功能。功能通常分为两个控制器，分别负责两个不同的 CRD。CRD 包括 VolumeSnapshot 和 VolumeContentSnapshot。在高层次上，VolumeSnapshot 负责卷的生命周期。基于这些对象，VolumeContentSnapshot 由 external-snapshotter 控制器管理。该控制器通常作为 CSI 控制器插件的 sidecar 运行，并将请求转发给驱动程序。

###### 注意

在撰写本文时，这些对象实现为 CRD，而不是核心 Kubernetes API 对象。这要求 CSI 驱动程序或 Kubernetes 发行版预先部署 CRD 定义。

与通过 StorageClasses 提供存储类似，通过引入 Snapshot 类提供快照功能。以下 YAML 表示了此类：

```
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshotClass
metadata:
  name: default-snapshots
driver: ebs.csi.aws.com ![1](img/1.png)
deletionPolicy: Delete ![2](img/2.png)
```

![1](img/#co_container_storage_CO8-1)

委托快照请求的驱动程序。

![2](img/#co_container_storage_CO8-2)

当删除 VolumeSnapshot 时是否应删除 VolumeSnapshotContent。实际上，可能会删除实际卷（具体取决于供应商的支持）。

在应用程序的命名空间和 PersistentVolumeClaim 中，可能会创建一个 VolumeSnapshot。示例如下：

```
apiVersion: snapshot.storage.k8s.io/v1beta1
kind: VolumeSnapshot
metadata:
  name: snap1
spec:
  volumeSnapshotClassName: default-snapshots ![1](img/1.png)
  source:
    persistentVolumeClaimName: pvc0 ![2](img/2.png)
```

![1](img/#co_container_storage_CO9-1)

指定要使用的类别，以通知驱动程序使用。

![2](img/#co_container_storage_CO9-2)

指定要进行快照的卷索赔。

此对象的存在将通知需要创建 VolumeSnapshotContent 对象。此对象的作用域为整个集群。检测到 VolumeSnapshotContent 对象将导致请求创建快照，并通过与 AWS EBS 通信来满足驱动程序的请求。满足后，VolumeSnapshot 将报告 ReadyToUse。 图 4-5 展示了各种对象之间的关系。

![prku 0405](img/prku_0405.png)

###### 图 4-5\. 组成快照流的各种对象及其关系。

有了快照后，我们可以探索数据丢失的场景。无论原始卷是因为意外删除、故障或因 PersistentVolumeClaim 的意外删除而移除，我们都可以恢复数据。为此，创建一个带有指定`spec.dataSource`的新 PersistentVolumeClaim。`dataSource`支持引用 VolumeSnapshot，可以将数据填充到新声明中。以下清单从先前创建的快照中恢复：

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-reclaim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: default-block
  resources:
    requests:
      storage: 600Gi
  dataSource:
    name: snap1 ![1](img/1.png)
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

![1](img/#co_container_storage_CO10-1)

VolumeSnapshot 实例引用 EBS 快照以填充新的 PVC。

一旦重新创建 Pod 以引用这个新声明，最后一个快照状态将返回到容器中！现在我们可以访问所有用于创建强大的备份和恢复解决方案的基本元素。解决方案可以从通过 CronJob 调度快照、编写自定义控制器，到使用工具如[Velero](https://velero.io)定期备份 Kubernetes 对象及数据卷。

# 摘要

在本章中，我们探讨了各种容器存储主题。首先，我们希望深入了解应用程序的需求，以便最好地指导我们的技术决策。然后，我们希望确保我们的底层存储提供商能够满足这些需求，并且我们有操作经验（在需要时）。最后，我们应确立编排器与存储系统之间的集成，确保开发人员可以获取所需的存储，而无需精通底层存储系统。
