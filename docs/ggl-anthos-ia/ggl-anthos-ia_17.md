# 17 在裸金属上运行的计算环境

乔治·加洛罗

本章涵盖

+   裸金属 Anthos 简介

+   部署选项

+   网络架构

+   存储架构

+   在裸金属上安装和配置 Anthos

Anthos 的原始发布版本要求您在 vSphere 基础设施上部署您的集群，并且不提供在不同的虚拟机管理程序或物理服务器上部署的选项。对于初始发布版本来说，这样做是有道理的，因为 vSphere 被众多企业使用，这允许企业使用他们现有的基础设施和技能集。然而，随着容器和 Kubernetes 用例的增长，组织越来越希望并且需要更灵活的部署选项。

为了解决这些额外的用例，Google 将 Anthos 扩展到包括裸金属部署模型。需要强调的一点是，您不必在裸金属上实际部署 Anthos 到物理服务器。裸金属模型允许您将部署到任何支持的操作系统，无论是物理服务器还是虚拟机，甚至是运行在 Hyper-V 或 KVM 上的虚拟机。

您可以将裸金属选项视为“自带 Linux”的部署模型。与 vSphere 部署模型不同，您需要提供可用的服务器，然后才能在裸金属上部署 Anthos。现在让我们向您介绍裸金属 Anthos。

## 17.1 裸金属 Anthos 简介

如前几章所述，Anthos 是一个为多个部署环境设计的平台，如图 17.1 所总结。

![17-01](img/17-01.png)

图 17.1 Anthos 部署环境

裸金属上的 Anthos 是一种部署选项，可以在客户提供的操作系统上运行的物理服务器上运行 Anthos。它内置了网络、生命周期管理、诊断、健康检查、日志记录和监控功能。此外，它支持 CentOS、Red Hat Enterprise Linux (RHEL) 和 Ubuntu，所有这些都经过 Google 验证。使用裸金属上的 Anthos，您可以使用组织的标准硬件和操作系统镜像，利用现有投资，这些投资将自动检查并验证是否符合 Anthos 基础设施要求。

### 17.1.1 比较本地 Anthos 部署选项

现在您有选择在本地部署 Anthos 的选项，那么您如何决定哪种最适合您的组织？这两种选项都有其自身的优缺点，为了决定哪种部署最适合您，您需要考虑您个人的需求。在表 17.1 中，我们概述了每个选项的一些优缺点。

表 17.1 Anthos 在 VMware 和裸金属上的优缺点

| VMware 上的 Anthos | 裸金属上的 Anthos |
| --- | --- |

| 在 VMware 上运行最佳适用于

+   vSphere 作为企业标准

+   涉及多个团队或集群（开发/测试）共享硬件

+   集成操作系统生命周期管理

+   集群的自愈/自动扩展

| 在裸金属或本地 IaaS 上运行，最适合希望

+   降低成本和复杂性（由于消除了 vSphere 许可证）

+   低延迟工作负载（电信和高性能计算）

+   为了通过简化的软件堆栈解锁边缘计算的新的用例

+   为了更接近硬件以获得更好的性能

|

| 部署优点 |
| --- |

|

+   部署多个集群更容易

+   提供的节点设备需要很少的维护

+   包括两个用于持久磁盘的 vSphere 存储提供程序

+   节点自动修复

+   容易扩展集群节点

|

+   可以部署到任何支持的 Linux 节点，本地或云服务提供商

+   没有工作负载调度冲突

+   扩展 GPU 兼容性

+   允许节点定制以满足组织的需求

+   更好的节点性能

+   使用现有的企业标准（例如，日志记录和监控标准）

|

| 部署缺点 |
| --- |

|

+   需要额外的 VMware 许可证。

+   不支持节点设备的定制。

+   通过直通支持有限的 GPU。

+   需要对虚拟机团队或 Kubernetes 支持团队进行额外的培训。

+   vSphere 调度程序和 Kubernetes 调度程序互不认识。

+   Storage DRS 可能会破坏您的集群。

|

+   需要规划以正确调整节点大小，以避免资源浪费

+   除了本地主机存储外，不包括其他存储分配器

+   在大多数企业中难以扩展

+   没有节点自动修复

+   管理和更新操作系统底层的节点

|

## 17.2 Anthos 裸金属架构

与 Anthos on VMware 集群节点从 Google 提供的预配置 VM 映像部署不同，Anthos on bare metal 依赖于客户提供他们自己管理和打补丁的受支持的操作系统版本。如图 17.2 所示，操作系统可以直接安装在物理服务器上，或者安装在支持与 Anthos on bare metal 兼容的 Linux 分发的任何虚拟化平台（KVM、OpenStack）上运行的 VM 上。

![17-02](img/17-02.png)

图 17.2 Anthos 本地部署选项，共享责任模型

### 17.2.1 集群架构

在本节中，我们将讨论裸金属集群的架构。您从 Anthos on VMware 部署中了解到的许多架构对于裸金属也是相同的；然而，裸金属选项包括一些与 VMware 模型不同的架构差异。

集群角色

Anthos 裸金属安装有以下两种类型的集群：

+   *用户集群*—应用程序部署的地方，它包括控制平面节点和运行容器化应用程序实例的工作节点。

+   *管理集群*—管理一个或多个用户集群的集群。它通过配置在两个自定义资源（集群和 NodePool）中的 Anthos 在裸金属上特定的操作员来安装、更新、升级和删除用户集群。

管理员集群仅包括控制平面节点，其中运行用于管理安装的组件。它还托管一些安全敏感的数据，包括访问节点 OS 的 SSH 密钥和 GCP 服务账户密钥。与在 VMware 上运行的 Anthos 不同，用户集群控制平面节点与管理员集群解耦。

**高可用性**

你可以在高可用性（HA）模式下运行用户或管理员集群控制平面，这样控制平面节点故障就不会影响集群操作。此模式需要三个或更多控制平面节点。如果不需要高可用性，你可以在每个集群中运行单个控制平面节点，但这种方法仅适用于非生产工作负载。

除了控制平面外，你还需要考虑工作节点的可用性。对于需要高可用性的应用程序，你需要至少两个工作节点。与控制平面一样，你不应该在没有为工作节点启用高可用性的情况下运行生产工作负载。

**部署模型**

这是你开始看到 Anthos 在 VMware 和裸金属之间部署差异的地方。在裸金属上运行的 Anthos 提供几种不同的部署模型，以提供灵活性以满足不同的组织需求。

**独立集群部署**

在独立集群部署模型中，如图 17.3 所示，单个集群同时作为管理员集群和用户集群。由于此模型不需要单独的管理员集群，你可以在高可用性设置中节省三个节点。这种情况在需要独立管理每个集群或对单个集群每个部署位置都有要求的情况下可能很有用，例如边缘用例或隔离网络。

![17-03](img/17-03.png)

图 17.3 独立集群部署

从安全角度来看，你需要考虑用户工作负载将在与控制平面相同的集群上运行。你需要仔细考虑保护你的集群，以保护诸如节点 SSH 密钥和 GCP 服务账户密钥等信息。实施 RBAC 策略、OPA 策略、网络策略和适当的审计将有助于保护集群。

为了为这些类型的部署提供更多灵活性，从 Anthos 版本 1.8 开始，Google 将每个集群支持的最小节点数从两个减少到一个，并为独立集群引入了新的边缘配置文件，这进一步降低了硬件要求。

**多集群部署**

这是 Anthos 在 VMware 上使用的相同部署模型。在多集群部署模型中，如图 17.4 所示，你有一个单个管理员集群管理多个用户集群。如果需要集中管理在同一数据中心部署的集群系列，则此模型很有用。

![17-04](img/17-04.png)

图 17.4 多集群部署

**混合集群部署**

混合集群部署模型，如图 17.5 所示，与多集群部署类似，只有一个区别：你可以使用管理集群来运行用户工作负载以及标准工作节点。

![17-05](img/17-05.png)

图 17.5 混合集群部署

如你所见，与 VMware 部署模型相比，Anthos 在裸金属上增加了更大的部署灵活性。然而，增加的灵活性并不止于此。在下一小节中，我们将讨论网络架构的更新。

网络架构

对于裸金属选项的网络架构与 VMware 模型的不同是有道理的。在 VMware 部署中，你可以使用外部负载均衡器，或者使用捆绑的负载均衡器，Seesaw。如果你选择 Seesaw 选项，Anthos 将在你的 VMware 环境中部署一个预配置的虚拟机，或者以 HA 模式运行的机器。

在裸金属部署中，Google 不提供任何组件的设备或虚拟机镜像。不过，不用担心——Google 通过其他组件如 HAProxy 和 MetalLB 来解决这个问题。

负载均衡

Anthos 在裸金属上需要使用层 4（L4）负载均衡来暴露控制平面端点、入口端点和应用程序，使用 LoadBalancer 类型的服务。负载均衡器的责任是路由和平衡流量到适当的节点。

无论你选择哪种集群部署模型，Anthos 在裸金属上可以通过捆绑的 L4 负载均衡器（捆绑负载均衡器模式）提供所需的负载均衡功能，或者，你也可以使用客户提供的和配置的外部负载均衡解决方案（手动负载均衡器模式）。

无论你选择哪种 L4 负载均衡选项，在安装过程中都会部署基于 Envoy 的 Istio Ingress 网关，并通过 L4 负载均衡器以 LoadBalancer 类型的服务公开。这个 Envoy 部署用于为应用程序提供代理功能，通过标准的 Kubernetes 入口对象暴露它们。

捆绑式负载均衡器模式

首先，让我们讨论控制平面的负载均衡。如果你选择捆绑的负载均衡，Anthos 在裸金属上部署 L4 负载均衡器，在集群安装期间，无需提供外部负载均衡器。负载均衡器可以运行在专用的工作节点池上，或者位于与控制平面相同的节点上。在任一情况下，负载均衡器节点必须在同一子网中。

从 Anthos 1.9 版本开始，Google 改变了 L4 负载均衡器的部署方式。之前，HAProxy 容器镜像作为标准 Docker 容器（即非 Kubernetes 控制）以 systemd 服务的形式部署到节点（s）。从版本 1.9 开始，Keepalived 和 HAProxy 容器已更新为在负载均衡器节点上作为静态 Kubernetes Pod 运行。

HAProxy 仅用于控制平面的负载均衡。为了向数据平面提供 L4 服务，Anthos 部署了一个流行的开源解决方案，称为 MetalLB，该解决方案为集群中任何使用 LoadBalancer 类型部署的服务提供服务。

为了回顾捆绑的负载均衡器组件：

+   *控制平面负载均衡*——控制平面虚拟 IP 地址（VIP），将流量路由到运行在控制平面节点上的 Kubernetes API 服务器，通过在负载均衡器节点上作为 Kubernetes Pod 运行的 HAProxy 负载均衡器公开，同时还有一个容器化的 Keepalived 服务来管理 HAProxy 的高可用性。

+   *数据平面负载均衡*——为应用程序创建的 LoadBalancer 类型服务对象以及与裸金属上的 Anthos 一起部署的 Istio Ingress 网关，通过在负载均衡器节点上运行的 Anthos 管理的 MetalLB 部署公开，IP 地址可以从预定义的池中自动分配，并且是负载均衡器节点部署的同一子网的一部分。

控制平面负载均衡组件（HAProxy 和 Keepalived）以及数据平面负载均衡组件（MetalLB）都在指定的节点上一起运行（集群控制平面节点或专用负载均衡器工作节点）。

图 17.6 显示了在单个子网中部署的用户集群的架构示例，其中捆绑的负载均衡器在控制平面节点上运行。图 17.7 显示了在单个子网中部署的用户集群的架构示例，其中捆绑的负载均衡器在专用工作节点上运行。

![17-06](img/17-06.png)

图 17.6 在控制平面节点上运行的负载均衡器

![17-07](img/17-07.png)

图 17.7 在专用工作节点上运行的负载均衡器

手动负载均衡器模式

如果你选择手动负载均衡器模式，Anthos 安装不会部署捆绑的负载均衡器，你需要负责部署外部负载均衡解决方案。

图 17.8 显示了在单个子网中部署的用户集群的示例，其中配置了手动负载均衡模式的外部负载均衡器。

![17-08](img/17-08.png)

图 17.8 手动负载均衡架构

内部集群网络

在裸金属上部署的 Anthos 使用基于 GENEVE 隧道的覆盖网络，这要求集群中的节点之间具有第 3 层（L3）连接性，除了需要位于同一第 2 层（L2）域中的负载均衡器节点。

类似于 Anthos 在 VMware 上的工作方式，Pod IP 寻址在孤岛模式下工作，这意味着分配给 Pod 的 IP 地址只能从同一集群访问，Pod CIDR 范围可以在集群之间重复使用。

存储架构

为在裸金属上运行的 Anthos 工作负载提供持久存储的主要方法是通过来自 Anthos 准备存储合作伙伴的 Container Storage Interface 驱动程序。您可以在 [`mng.bz/1MdX`](http://mng.bz/1MdX) 找到合作伙伴列表和经过验证的存储解决方案。

裸金属上的 Anthos 还捆绑了 sig-storage-local-static-provisioner，它为每个节点提供挂载点，并为每个挂载点创建一个本地持久卷 (PV)。由于其局限性，您应仅将本地 PV 用于非生产环境或特定的先进用例。

可观测性

您可以使用 Google Cloud 操作收集裸金属上 Anthos 的日志和监控指标，就像在其他 Anthos 部署环境中一样。默认情况下，系统组件日志被发送到 Cloud Logging，系统组件指标被发送到 Cloud Monitoring。通过在集群配置中启用它，Cloud 操作还可以收集应用程序日志和指标（使用 Google Managed Service for Prometheus）。

作为云操作的一种替代方案，如果您更倾向于使用其他解决方案，可以选择 Prometheus/Grafana、Elastic Stack、Sysdig、Datadog 或 Splunk。

身份集成

通过 Anthos Identity Service 认证代理，裸金属上的 Anthos 可以与支持 OpenID Connect (OIDC) 或 LDAP 的任何身份提供者集成，以使用现有的用户身份和凭据管理对集群的用户和组身份验证。如果您已经使用或希望使用 Google IDs 登录到您的 Anthos 集群而不是 OIDC 或 LDAP 提供商，建议您使用 Connect 网关进行身份验证。

通过这种集成，您可以使用组织内部创建、启用和禁用账户的标准程序来管理对裸金属上 Anthos 集群的访问。您可以使用 Kubernetes RBAC 将特定角色绑定到身份提供者中定义的用户和组，以授权他们在特定资源上执行特定操作。

## 17.3 安装和配置概述

在本节中，我们将概述部署集群所需的要求。在尝试部署集群之前，了解这些要求是一个重要的步骤，我们将在下一节中进行讨论。

Google 已经使在裸金属上部署 Anthos 变得简单。与 VMware 部署类似，您在配置文件中配置集群选项，并使用名为 bmctl 的单个二进制可执行文件执行部署。

Anthos 要求您满足软件和硬件的要求。大多数组织很容易满足这些要求，但在部署之前，您应始终验证您的基础设施是否满足所有要求。

### 17.3.1 操作系统和软件要求

如介绍和架构部分所述，Anthos on bare metal 是安装在客户提供的并由客户配置的服务器上。服务器可以是物理的或虚拟的，只要它们具有支持的操作系统之一，并配置以满足 Anthos 的要求。用作 Anthos on bare metal 节点的服务器需要以下操作系统之一：

+   CentOS 8.2/8.3/8.4/8.5

+   Red Hat Enterprise Linux (RHEL) 8.2/8.3/8.4/8.5/8.6

+   Ubuntu 18.04 和 20.04

每个支持的版本都需要稍有不同的配置。如果您使用 RHEL 或 CentOS，必须配置 firewalld 服务以允许流量通过 TCP 和 UDP 端口——这些将在 17.3.4 节中的内部连接性要求中介绍。在这些操作系统上，如果 SELinux 以强制模式启用，则在 Anthos on bare metal 设置期间将配置容器隔离和安全策略。如果您在 Ubuntu 上运行节点，则必须禁用 Uncomplicated Firewall 服务。

时间对于集群来说非常重要。为确保所有节点的时钟同步，所有服务器都需要配置并启用 NTP 服务。最后，因为安装会在节点上建立 SSH 连接，所以您需要一对 SSH 密钥来以 root 权限访问每个节点。

### 17.3.2 硬件容量要求

Anthos on bare metal 可以在任何与支持的操作系统兼容的硬件上运行。安装所需的节点数量取决于选择的部署和负载均衡模型，如前所述。所需的 worker 节点数量将取决于集群（们）将要托管的应用程序的能力要求。

表 17.2 描述了每个节点的最小和推荐硬件要求，无论其角色如何，使用默认配置文件，不包括应添加的应用程序容量要求。

表 17.2 Anthos on bare metal 的硬件要求

| 资源 | 最小 | 推荐 |
| --- | --- | --- |
| CPU/vCPU | 4 核 | 8 核 |
| 内存 | 16 GiB | 32 GiB |
| 存储 | 128 GiB | 256 GiB |

表 17.3 描述了 Anthos on bare metal 版本 1.8 中引入的边缘配置文件的硬件要求。

表 17.3 边缘配置文件的硬件要求

| 资源 | 最小 | 推荐 |
| --- | --- | --- |
| CPU/vCPU | 2 核 | 4 核 |
| 内存 | Ubuntu: 4 GBCentOS/RHEL: 6 GiB | Ubuntu: 8 GBCentOS/RHEL: 12 GiB |
| 存储 | 128 GiB | 256 GiB |

### 17.3.3 管理工作站

除了节点之外，建议再有一个工作站来运行安装工具。此工作站必须运行与集群节点相同的 Linux 操作系统，并配置 Docker 19.03 或更高版本，以便由非 root 用户管理。除了 Docker 之外，该机器还必须具备以下条件：

+   已安装带有 anthos-auth 和 kubectl 的 gcloud。

+   需要 50 GB 以上的空闲磁盘空间。

+   到所有集群节点机器的 L3 连接性。

+   通过私钥和密码无根访问权限通过 SSH 访问所有集群节点机器。访问可以是直接或通过 sudo。

+   访问控制平面 VIP。

### 17.3.4 网络需求

Anthos 对外部和内部网络连接有不同的需求。

外部连接需求

所有在裸金属节点上的 Anthos 都需要出站 HTTPS 连接到互联网以执行以下操作：

+   在 GCP 控制台中注册并从那里通过 GKE Connect 进行管理

+   将指标和日志发送到云操作端点

+   从 Google 容器注册库拉取镜像

这种连接可以使用公共互联网、HTTP 代理或类似 Google Cloud VPN 或专用互连的私有连接。

内部连接需求

本节将详细说明您集群的内部网络需求。集群的每个组件都有不同的需求，表格 17.4-17.7 列出了用于集群节点流量的特定连接端口。

表 17.4 控制平面节点

| 协议 | 方向 | 端口范围 | 目的 | 使用者 |
| --- | --- | --- | --- | --- |
| UDP | Inbound | 6081 | GENEVE 封装 | Self |
| TCP | Inbound | 22 | 管理集群节点的配置和更新 | 管理工作站 |
| TCP | Inbound | 6444 | Kubernetes API 服务器 | All |
| TCP | Inbound | 2379-2380 | etcd 服务器客户端 API | kube-apiserver 和 etcd |
| TCP | Inbound | 10250 | kubelet API | Self 和控制平面 |
| TCP | Inbound | 10251 | kube-scheduler | Self |
| TCP | Inbound | 10252 | kube-controller-manager | Self |
| TCP | Inbound | 10256 | 节点健康检查 | All |
| TCP | Both | 4240 | CNI 健康检查 | All |

表 17.5 工作节点

| 协议 | 方向 | 端口范围 | 目的 | 使用者 |
| --- | --- | --- | --- | --- |
| TCP | Inbound | 22 | 用户集群节点的配置和更新 | 管理集群节点 |
| UDP | Inbound | 6081 | GENEVE 封装 | Self |
| TCP | Inbound | 10250 | kubelet API | Self 和控制平面 |
| TCP | Inbound | 10256 | 节点健康检查 | All |
| TCP | Inbound | 30000-32767 | NodePort 服务 | Self |
| TCP | Both | 4240 | CNI 健康检查 | All |

表 17.6 负载均衡节点

| 协议 | 方向 | 端口范围 | 目的 | 使用者 |
| --- | --- | --- | --- | --- |
| TCP | Inbound | 22 | 用户集群节点的配置和更新 | 管理集群节点 |
| UDP | Inbound | 6081 | GENEVE 封装 | Self |
| TCP | Inbound | 443 | 集群管理 | All |
| TCP | Both | 4240 | CNI health check | All |
| TCP | Inbound | 7946 | Metal LB 健康检查 | 负载均衡节点 |
| TCP | Inbound | 10256 | 节点健康检查 | All |
| UDP | Inbound | 7946 | Metal LB 健康检查 | 负载均衡节点 |

表 17.7 多集群端口需求

| 协议 | 方向 | 端口范围 | 目的 | 使用者 |
| --- | --- | --- | --- | --- |
| TCP | Inbound | 22 | 集群节点的配置和更新 | 所有节点 |
| TCP | Inbound | 443 | 添加集群的 Kubernetes API 服务器 | 控制平面和负载均衡节点 |

在网络要求得到解决后，让我们继续讨论配置集群负载均衡器的附加要求。

负载均衡要求

在裸金属上安装 Anthos 之前，您需要选择一个负载均衡的架构（手动或捆绑）以及在捆绑的情况下，决定您的负载均衡器是否将安装在控制平面节点或专用工作节点上。无论您选择哪种解决方案，以下 VIP 地址都必须预留：

+   *每个集群的控制平面一个 VIP*——如果您使用捆绑的负载均衡器，这将根据您在安装过程中定义的配置自动创建。如果您使用手动负载均衡器，则需要手动将其关联到一个包含集群控制平面节点所有 IP 地址的后端服务器组。控制平面监听的后端端口是 6444。

+   *每个用户集群的 Ingress 服务一个 VIP*——如果您使用捆绑的负载均衡器，这将根据您在安装过程中定义的配置自动创建。如果您使用手动负载均衡器，则需要手动配置，使用分配给在集群 gke-system 命名空间中创建的 istio-ingress Service 的相同 IP 地址，并关联到一个包含集群节点 IP 地址的后端服务器组。后端端口将是 istio-ingress Service 的 NodePort。如果您想同时使用 Ingress 网关处理 HTTP 和 HTTPS 流量，可能需要为每个端口配置一个 VIP（和后端池）。

+   *集群中每个 LoadBalancer 类型 Service 创建一个 VIP*——如果您使用捆绑的负载均衡器，这些将根据您在安装过程中定义的池自动分配。如果您使用手动负载均衡器，则需要手动配置，使用分配给 Service 对象的相同 IP 地址，并关联到一个包含集群工作节点所有 IP 地址的后端服务器组。后端端口将是 Service 对象的 NodePort。

如果集群部署将使用捆绑的负载均衡器，以下项目必须进行配置：

+   负载均衡节点需要位于同一 L2 网络中，而其他连接，包括工作节点，只需要 L3 连接性。

+   所有 VIP 必须位于负载均衡器机器子网中，并且完全可路由。

+   负载均衡器子网的网关必须监听免费 ARP 消息并将 ARP 数据包转发到负载均衡器节点。

接下来，下一节将介绍 Google Cloud Platform 的要求。

### 17.3.5 Google Cloud Platform 要求

Anthos 在裸金属上的安装有几个 GCP 项目要求，包括必需的 API、服务帐户和必需的角色。

必需的 GCP API

为了成功部署，将连接到集群的项目必须启用几个 API。您可以手动完成此操作，或者您可以在使用 bmctl 执行部署时自动启用它们作为选项。在用于安装的 GCP 项目中必须启用以下 API：

+   anthos.googleapis.com

+   anthosaudit.googleapis.com

+   anthosgke.googleapis.com

+   cloudresourcemanager.googleapis.com

+   container.googleapis.com

+   gkeconnect.googleapis.com

+   gkehub.googleapis.com

+   iam.googleapis.com

+   serviceusage.googleapis.com

+   stackdriver.googleapis.com

+   monitoring.googleapis.com

+   logging.googleapis.com

+   opsconfigmonitoring.googleapis.com

如果在运行部署之前未启用任何 API，预检检查将捕获缺失的 API 并停止部署继续进行。

必需的服务帐户和角色

在裸金属上部署 Anthos 之前的一个要求是创建所需的服务帐户和所需的角色。尽管您可以使用具有所有角色的单个帐户，但这被认为是一种不良的安全实践。您的组织将有其自己的安全要求，但建议您创建所有帐户作为不同的服务帐户。

您可以选择手动创建服务帐户，或者您可以在安装过程中创建它们，使用 bmctl 安装工具的参数。在裸金属上运行的 Anthos 需要以下具有指定角色的 Google Cloud 服务帐户：

+   没有特殊角色的服务帐户 Container Registry (gcr.io)

+   用于将集群注册到 GCP 控制台并具有 GKE hub 管理员 IAM 角色的服务帐户

+   用于维护集群与 Google Cloud 之间连接并具有 GKE Connect 代理 IAM 角色的服务帐户

+   用于将日志和指标发送到 Google Cloud 操作套件并具有以下 IAM 角色的服务帐户：

    +   日志记录器

    +   监控指标写入器

    +   Stackdriver 资源元数据写入器

    +   监控仪表板配置编辑器

    +   Ops 配置监控资源元数据写入器

如果您想通过 bmctl 工具启用这些 API 并在安装过程中创建所需的 GCP 服务帐户，用于安装的帐户必须具有项目所有者/编辑角色，或者至少分配以下角色：

+   服务帐户管理员

+   服务帐户密钥管理员

+   项目 IAM 管理员

+   计算机查看器

+   服务使用管理员

最后，我们将涵盖一个 Anthos 将用于集群指标的要求。

云指标要求

要将指标发送到 Google Cloud 的操作套件，除了上一节中列出的服务帐户外，您必须在 GCP 项目中创建一个云监控工作区。

## 17.4 创建集群

满足所有要求后，您可以继续创建集群。以下章节假设所有安装任务都是在满足第 17.3.3 节所述要求的机器上执行的。

### 17.4.1 创建管理员、混合或独立集群

正如我们详细说明的，您可以使用几种不同的集群模型在裸金属上部署 Anthos，包括独立的管理员/用户集群、混合集群或独立集群。在本节中，我们将讨论部署每种模型的过程。

安装流程摘要

您可以使用 bmctl 工具在特定的部署环境中安装第一个集群，无论选择哪种模型。在相同环境中创建附加用户集群可以通过应用 Anthos on bare metal 用户集群配置文件实现，该文件与第一个集群配置类似，只有一些细微的更改。创建第一个集群的高级步骤如下：

1.  下载 bmctl 工具。

1.  使用 bmctl 创建集群配置模板文件。

1.  使用所需的设置修改配置文件。

1.  运行 bmctl 创建集群。

图 17.9 展示了使用 bmctl 创建第一个集群的流程。

![图片](img/17-09.png)

图 17.9 使用 bmctl 创建集群的流程

前四个步骤由用户启动，当您执行第 4 步，即 bmctl create cluster 命令时，将执行以下步骤：

+   *验证配置*—检查集群配置文件，以验证规范是否格式正确，没有 IP 地址冲突发生，服务账户密钥可用，并且集群尚未在 GCP 控制台中注册。

+   *创建 kind 集群*—作为设置的一部分，bmctl 首先在管理员工作站上创建一个临时的 kind（Docker 中的 Kubernetes）集群，其中一些用于管理员集群的资源，如集群和 NodePool 对象或包含静态站点生成器和服务账户密钥的 Secrets，是从配置文件规范创建的。

+   *预检查*—对集群机器和网络要求进行检查，例如操作系统版本和配置、文件系统可用空间以及 GCP API 端点的可达性。

+   *配置裸金属集群*—二进制文件被复制到目标节点，并执行安装，包括节点初始化和加入。

+   *安装附加组件*—安装附加组件，如 GKE Connect Agent、日志和监控组件、裸金属操作员和 MetalLB。

+   *转换*—将裸金属资源从 kind 集群移动到已配置集群的过程。之后将从 kind 集群中删除 Kubernetes 资源。

最后，让我们深入探讨如何创建一个集群！

登录 GCP 并下载 bmctl 工具

记住，Google 提供了一个名为 bmctl 的工具，用于在裸金属上部署 Anthos。在您将用于部署集群的工作站上，按照以下步骤下载 bmctl 工具（在我们的示例中，我们将假设工作目录是 ~/baremetal）：

1.  使用具有“安装账户角色要求”部分中描述的角色之一的用户，通过 gcloud auth application-default login 登录 gcloud。

1.  从文档中找到的 URL 或存储桶下载 bmctl 工具，如下所示：

    ```
    gsutil cp gs://anthos-baremetal-release/bmctl/<VERSION>/linux-amd64/bmctl bmctl)
    ```

1.  使 bmctl 可执行：

    ```
    chmod a+x bmctl
    ```

现在我们有了 bmctl 可执行文件，我们可以继续创建集群配置。

创建集群配置

要部署集群，我们需要有一个包含部署所需所有参数和选项的集群配置文件。bmctl 工具可以通过使用 create config 选项为我们创建一个新的配置文件：

```
bmctl create config -c CLUSTER_NAME --project-id=CLOUD_PROJECT_ID
```

其中 CLUSTER_NAME 是你想要给集群的名字，CLOUD_PROJECT_ID 是你想要与裸金属上的 Anthos 一起使用的项目的项目 ID。

如果你还没有启用所需的 API，你可以在之前的命令中添加--enable-apis 选项来启用它们，如果你还没有创建所需的服务帐户，你可以添加选项--create-service-accounts 以在创建所需角色时同时创建它们。

填充集群配置文件

在创建集群之前，我们需要正确准备由 bmctl 创建的配置文件。该文件默认保存在名为*bmctl-workspace*的文件夹内，其名称与集群名称相同。在本节中，我们将解释配置文件中的选项。

SSH 私钥

这是将在集群部署期间用于连接到节点的 SSH 私钥。将 sshPrivateKeyPath:规范添加到完整的路径，以便访问授权访问所有目标节点作为 root 的 SSH 私钥，例如：

```
sshPrivateKeyPath: /root/.ssh/anthoskey
```

GCP 服务帐户密钥

如果你已经在运行 bmctl 之前手动创建了 GCP 服务帐户，你需要使用以下示例中的完整路径填充相关字段，以服务帐户密钥的完整路径。如果你使用了 bmctl --create-service-accounts 参数，它们将已经填充：

```
gkeConnectAgentServiceAccountKeyPath: /root/bmctl-workspace/.sa-keys/anthos-demos-anthos-baremetal-connect.json
gkeConnectRegisterServiceAccountKeyPath: /root/bmctl-workspace/.sa-keys/anthos-demos-anthos-baremetal-register.json
cloudOperationsServiceAccountKeyPath: /root/bmctl-workspace/.sa-keys/anthos-demos-anthos-baremetal-cloud-ops.json
gcrKeyPath: /root/bmctl-workspace/.sa-keys/anthos-demos-anthos-baremetal-gcr.json
```

集群类型

根据所选的集群部署模型，在 Cluster 自定义资源中设置 type spec 的值，选择 admin、hybrid*或 standalone，如下例所示：

```
---
.gke.io/v1
kind: Cluster
metadata:
 name: admin-cluster
 namespace: cluster-admin-cluster
spec:
 type: hybrid
```

控制平面配置

根据所选的控制平面架构，在 controlPlane:部分的 nodePoolSpec:规范中添加目标控制平面节点的 IP 地址。以下是一个基于三个控制平面节点的 HA 架构的示例。记住，如果你想启用高度可用的控制平面，你需要提供至少三个 IP 地址：

```
 controlPlane:
   nodePoolSpec:
     nodes:
     # Control plane node pools. Typically, this is either a single machine
     # or three machines if using a high availability deployment.
     - address: 172.16.0.3
     - address: 172.16.0.4
     - address: 172.16.0.5
```

Pod 和服务的 CIDR 块

clusterNetwork:部分包括分配给集群内 Pods 和 Kubernetes Service 对象的 CIDR 范围；这些范围仅在集群内部可见，永远不会用于外部。只有当与你的网络中现有服务的任何重叠时，才更改默认值，这些服务是集群中运行的 Pod 可能需要联系的服务。

负载均衡器配置

您需要根据所选的负载均衡器模式（捆绑式或手动）以及该模式的期望配置选项来填写 loadBalancer: 部分。以下是对各种规范的描述：

+   mode—负载均衡器模式；您需要在捆绑式或手动之间进行选择。

+   ports.controlPlaneLBPort—负载均衡器服务 Kubernetes API 服务器所使用的端口。

+   vips.controlPlaneVIP—分配给集群中 Kubernetes API 服务器上的 VIP。

+   vips.ingressVIP—分配给集群中第 7 层（L7）Istio Ingress 网关的 VIP；这必须是稍后定义的地址池的一部分。如果集群是混合型、独立型或用户型，则需要此 VIP；在管理集群中不需要，可以保持注释状态。

+   addressPools—数据平面负载均衡器使用的池，用于将 VIP 分配给 Ingress 网关和 LoadBalancer 类型的 Kubernetes 服务对象；它必须包括之前定义的 Ingress VIP，但在管理集群中不需要，可以保持注释状态。

+   nodePoolSpec—列出您想要部署捆绑式负载均衡器的节点地址。只有在您想为捆绑式负载均衡器指定专用工作节点时才需要使用它。如果未注释，负载均衡器将部署在控制平面节点上。

请记住，如果正在部署捆绑式负载均衡器，所有 VIP（包括控制平面和地址池以及 Ingress 网关）都必须位于负载均衡器节点的同一子网中。以下代码显示了一个配置示例，用于在两个专用工作节点（IP 地址为 172.16.0.7 和 172.16.0.7）上部署捆绑式负载均衡器的混合集群：

```
# Load balancer configuration
  loadBalancer:
    mode: bundled
    ports:
      controlPlaneLBPort: 443
    vips:
      controlPlaneVIP: 172.16.0.16
      ingressVIP: 172.16.0.17
    addressPools:
    - name: pool1
      addresses:
      - 172.16.0.17-172.16.0.26
    nodePoolSpec:
     nodes:
     - address: 172.16.0.7
     - address: 172.16.0.8
```

代理配置

如果节点需要通过 HTTP 代理连接到互联网，请在 proxy: 部分填写所需信息：

+   url—代理服务器可访问的 URL，格式为 http://username:password@fqdn:port

+   noProxy—不应代理的 IP 地址、主机名或域名列表

以下示例配置了一个可访问于 http://172.16.0.101:3128 的代理服务器条目，无需身份验证，并为 172.16.0.0/16 范围添加了 noProxy 条目，该条目告诉系统不要将此范围内的 IP 地址发送到代理服务器：

```
proxy:
   noProxy: // specifies a list of IPs, hostnames, and domains that should skip the proxy.
   - 172.16.0.0/16
   url: http://172.16.0.101:3128 // address of the proxy server.
```

日志和监控的云操作

要配置日志和监控的选项，您需要在 clusterOperations: 部分中添加 projectID 和 location，如下所述：

+   projectID—您想要托管指标和日志的项目 ID。

+   location—您想要存储日志和指标的 Google Cloud 区域。选择靠近您的本地数据中心所在的区域是个好主意。

默认情况下，云操作仅收集管理集群和工作负载以及系统命名空间（如 kube-system、gke-system、gke-connect、istio-system 和 config-management-system）中的工作负载的日志和指标。系统组件的日志和指标也由 Google 支持用于解决支持案例中的问题。除了系统命名空间的指标外，云操作还收集来自所有 Pod 的节点资源使用情况的指标。

您还可以配置云操作收集应用程序日志并使用 Prometheus 托管服务收集应用程序指标。您可以通过修改 stackdriver 自定义资源来在安装后启用这两种功能，就像在其他 Anthos 部署选项中一样。

存储配置

storage：部分包括用于使用本地节点上的挂载点提供持久卷的本地卷提供程序（LVP）的配置。使用本地持久卷将 Pod 绑定到特定的磁盘和节点。如果该磁盘或节点变得不可用，则 Pod 也变得不可用。因此，使用本地 PV 的工作负载需要能够抵御此类故障。因此，通常使用本地持久卷适用于数据持久性不是关键或数据已复制到其他卷且在节点或磁盘不可用的情况下可恢复的证明概念或高级用例。

Anthos 裸金属集群中本地 PV 的存储类有三种类型如下：

+   *LVP 节点挂载*—此存储类为指定目录中每个挂载的磁盘创建一个本地 PV。每个 PV 映射到与底层磁盘容量相等的磁盘。集群中创建的本地 PV 总数是所有节点下路径挂载的磁盘数量。

+   *LVP 共享*—此存储类为集群中每个节点的本地共享文件系统上的子目录创建一个由本地 PV 支持的存储类。这些子目录在集群创建期间自动创建。使用此存储类的工作负载将共享容量和每秒输入/输出操作，因为 PV 由相同的共享文件系统支持。

+   *Anthos 系统*—此存储类在集群创建期间创建预配置的本地 PV，供 Anthos 系统 Pod 使用。不要更改或删除此存储类，也不要为此存储类使用有状态应用。

lvpNodeMounts：部分包含以下参数，用于配置 LVP 节点挂载：

+   path—用作本地持久卷的磁盘挂载的本地节点目录路径。

+   storageClassName—将创建 PV 的 StorageClass。StorageClass 是在集群创建期间创建的。

lvpShare：部分包含以下参数以配置 LVP 共享：

+   path*—在每个主机上创建子目录的本地节点目录路径。将为每个子目录创建一个本地 PV。

+   storageClassName—将创建 PV 的 StorageClass。StorageClass 在集群创建期间创建。

+   numPVUnderSharedPath—在路径下创建的子目录数量。在集群中共享持久卷的 LVP 总数将是此数量乘以节点数。

下一个示例配置使用混合集群的默认参数：

```
 storage:
   lvpNodeMounts:
     path: /mnt/localpv-disk
     storageClassName: local-disks
   lvpShare:
     path: /mnt/localpv-share
     storageClassName: local-shared
     numPVUnderSharedPath: 5
```

认证

如前所述，Anthos 在裸金属上使用 Anthos Identity Service 认证代理通过 OpenID Connect (OIDC) 或 LDAP 与现有的身份提供程序集成。Anthos Identity Service 允许用户通过 GCP 控制台和 kubectl（在这种情况下，使用 gcloud CLI 进行身份验证并创建一个包含用于 kubectl 的 ID 令牌的 kubeconfig 文件）使用现有的企业凭据进行身份验证。

集群配置文件中的 authentication：部分可用于在创建集群时配置身份验证。在集群创建后，也可以使用 ClientConfig 对象或集群级别的 Anthos Identity Service 配置身份验证。

下一个示例配置设置创建集群时 OIDC 认证的参数：

```
 authentication:
    oidc:
      issuerURL: "https://infra.example.dev/adfs"
      clientID: "be654652-2c45-49ff-9d7c-3663cee9ba51"
      clientSecret: "clientSecret"
      kubectlRedirectURL: "http://localhost:44320/callback"
      username: "unique_name"
      usernamePrefix: "oidc:"
      group: "groups"
      groupPrefix: "oidc:"
      scopes: "allatclaims"
      extraParams: "resource=token-groups-claim"
      deployCloudConsoleProxy: true
      certificateAuthorityData: base64EncodedCACertificate
      proxy: http://10.194.2.140:3128
```

下一个示例使用 LDAP：

```
authentication:
  - name: ldap
    ldap:
      connectionType: ldaps
      group:
        baseDN: ou=Groups,dc=onpremidp,dc=example,dc=net
        filter: (objectClass=*)
        identifierAttribute: dn
      host: ldap.google.com:636
      user:
        baseDN: ou=Users,dc=onpremidp,dc=example,dc=net
        filter: (objectClass=*)
        identifierAttribute: uid
        loginAttribute: uid
      serviceAccountSecret:
        name: google-ldap-client-secret
        namespace: anthos-identity-service
        type: tls
```

工作节点节点池

如果安装中的第一个集群是混合或独立集群，旨在托管用户工作负载，您需要在集群配置文件中配置工作节点 NodePool 资源，提供目标工作节点的 IP 地址。以下是一个使用节点池中的三个工作节点的示例：

```
# Node pools for worker nodes
apiVersion: baremetal.cluster.gke.io/v1
kind: NodePool
metadata:
 name: node-pool-1
 namespace: cluster-hybrid-cluster
spec:
 clusterName: hybrid-cluster
 nodes:
 - address: 172.16.0.8
 - address: 172.16.0.9
 - address: 172.16.0.10
```

注意：如果集群是管理集群，本节内容不需要。

创建集群

现在我们已经有一个完整的配置文件，我们可以使用 bmctl create cluster 选项部署集群，如下所示：

```
bmctl create cluster -c CLUSTER_NAME
```

将 CLUSTER_NAME 替换为您在创建集群配置文件时定义的集群名称。这个过程需要一些时间，一旦集群成功创建，您就可以通过使用生成的 kubeconfig 文件来连接到它。

连接到集群

集群创建完成后，您可以使用安装工具在 bmctl-workspace/CLUSTER_NAME 文件夹中创建的 kubeconfig 文件来使用 kubectl 连接到它。

您也可以使用承载令牌从 GCP 控制台连接。支持多种类型的承载令牌。最简单的方法是在集群中创建一个 Kubernetes 服务帐户，并使用其承载令牌进行登录。

如果您已将集群配置为与身份提供程序进行身份集成，您可以通过创建 RoleBindings 或 ClusterRoleBindings 来授权现有用户和组对特定资源执行特定操作，将它们分配到具有所需权限的角色。在创建所需的绑定后，您可以通过选择 GCP 控制台中的“使用为集群配置的身份提供程序进行身份验证”选项从 GCP 控制台登录到集群。

要通过 kubectl 对集群进行认证以执行操作，您需要在创建了所需的 RoleBindings 和/或 ClusterRoleBindings 之后执行以下步骤：

1.  *创建和分发认证配置文件。* 您需要创建一个认证配置文件，该文件将被分发给需要使用 kubectl 访问集群的客户端。此文件包含从 gcloud CLI 获取的 OIDC 配置，以从客户端启动认证和令牌请求。

    从管理员工作站或任何可以访问安装过程中创建的 kubeconfig 文件的机器执行以下命令：

    ```
    gcloud anthos create-login-config --kubeconfig CLUSTER_KUBECONFIG
    ```

    将 CLUSTER_KUBECONFIG 替换为安装过程中创建的 kubeconfig 文件。如果命令成功完成，将在当前目录中创建名为 kubectl-anthos-config.yaml 的认证配置文件。此 kubeconfig 文件提供了对集群的 admin 访问权限，并且仅应提供给需要使用 kubectl 进行管理任务的人员。大多数组织应使用现有的安全标准来保护此文件，这些标准是“破窗”过程的一部分。

1.  *使用集群进行认证。* 用于访问集群的客户端机器需要安装 kubectl 和 gcloud CLI，包括 anthos-auth 组件。从客户端机器执行以下命令以从 OIDC 提供者获取 ID 令牌，并相应地配置本地 kubeconfig 以成功认证集群：

    ```
    gcloud anthos auth login \
     --cluster CLUSTER_NAME \
     --user USER_NAME \
     --login-config AUTH_CONFIG_FILE_PATH \
     --login-config-cert CA_CERT_PEM_FILE \
     --kubeconfig CLUSTER_KUBECONFIG
    ```

    登录选项在此处描述：

    +   CLUSTER_NAME—可选。这是您希望在目标 kubeconfig 文件中定义的集群名称。如果省略此标志，系统将提示您从您的认证配置文件中指定的集群中选择。

    +   USER_NAME—可选。这是在 kubeconfig 文件中使用的用户名；如果省略，则默认为 CLUSTER_NAME-anthos-default-user。

    +   AUTH_CONFIG_FILE_PATH—指定认证配置文件的路径。

    +   CA_CERT_PEM_FILE—指定从您的 CA 获取的 PEM 证书文件的路径，如果认证配置文件存储在 HTTPS 服务器上，则需要此证书。

    +   CLUSTER_KUBECONFIG—写入 OIDC ID 令牌的目标 kubeconfig 文件；如果省略，则默认为 kubectl 默认位置。

命令将在 OIDC 提供者的同意登录页面打开浏览器，您需要在此处输入凭据。您的 kubeconfig 文件现在包含一个 ID 令牌，您的 kubectl 命令将使用此令牌与集群上的 Kubernetes API 服务器进行认证。

### 17.4.2 创建用户集群

一旦您创建了一个管理员或混合集群，您就可以向其中添加用户集群。您可以通过应用一个只包含新集群的 Cluster 和 NodePool 自定义资源清单的新配置文件来完成此操作。创建第一个集群的高级步骤如下：

1.  使用 bmctl 创建集群配置模板文件。

1.  使用所需的设置修改配置文件。

1.  使用 bmctl 应用配置文件。

这些任务是你已经为创建第一个集群所做任务的一个子集。在下一节中，我们将详细说明部署用户集群的配置文件。

创建集群配置

与创建第一个集群时相同，使用以下命令启动集群配置文件：

```
bmctl create config -c CLUSTER_NAME 
```

其中 CLUSTER_NAME 是您希望赋予用户集群的名称。

填充集群配置文件

与创建第一个集群时相同，您需要准备由 create config 命令创建的配置文件。该文件默认保存在名为 bmctl-workspace 的文件夹内，以集群名称命名的文件夹中。

对于配置文件中的许多部分，已经为第一个集群创建提供的相同说明也适用于用户集群，因此请遵循第 17.4.1 节中“填充集群配置文件”给出的说明。

注意：确保控制平面和负载均衡器部分以及 NodePool 资源中使用的 IP 地址与您为第一个集群已使用的地址不冲突。

在下一节中，您将找到针对用户集群特定的任务。

删除凭据部分

用户集群将使用在管理员/混合集群创建期间提供的凭据，因此我们不需要为 GCP 提供凭据。因为这些不是必需的，我们需要从文件中删除，例如包含指向密钥的节：

```
gcrKeyPath: <path to GCR service account key>
sshPrivateKeyPath: <path to SSH private key, used for node access>
gkeConnectAgentServiceAccountKeyPath: <path to Connect agent service account key>
gkeConnectRegisterServiceAccountKeyPath: <path to Hub registration service account key>
cloudOperationsServiceAccountKeyPath: <path to Cloud Operations service account key>
```

集群类型

在集群中将类型规范值设置为用户：

```
---
apiVersion: baremetal.cluster.gke.io/v1
kind: Cluster
metadata:
 name: user-cluster
 namespace: cluster-user-cluster
spec:
 # Cluster type. This can be:
 type: user
```

负载均衡器配置

接下来，您需要提供用户集群负载均衡器的配置，如下所示。这里使用的 IP 地址不能与分配给第一个集群的负载均衡器重叠：

```
# Sample user cluster config for load balancer and address pools
loadBalancer:
    vips:
      controlPlaneVIP: 10.200.0.71
      ingressVIP: 10.200.0.72
    addressPools:
    - name: pool1
      addresses:
      - 10.200.0.72-10.200.0.90
```

创建集群

完成集群配置文件后，您可以使用以下命令创建第一个集群：

```
bmctl create cluster -c CLUSTER_NAME --kubeconfig ADMIN_KUBECONFIG
```

用户集群部署完成后，您可以使用生成的 kubeconfig 连接到新集群。

连接到集群

集群创建后，您可以使用 kubectl 从用户集群命名空间中安装过程创建的 Secret 中提取 kubeconfig 来连接到它。以下是一个提取 kubeconfig 的示例命令：

```
kubectl --kubeconfig ADMIN_KUBECONFIG get secret USER_CLUSTER_NAME-kubeconfig -n USER_CLUSTER_NAMESPACE -o ‘jsonpath={.data.value}’ | base64 -d > bmctl-workspace/user-cluster/USER_CLUSTER_NAME-kubeconfig
```

您也可以使用承载令牌从 GCP 控制台连接，或者如果您已为集群配置了身份集成，用户可以按照第 17.4.1 节中“连接到集群”所述的步骤从 GCP 控制台和 gcloud CLI 验证集群身份。

## 17.5 升级集群

当 Anthos on bare metal 发布新版本时，您可以升级您的集群。在非独立集群安装中，您需要首先升级管理员/混合集群，然后升级用户集群。

### 17.5.1 升级管理员、独立或混合集群

执行升级到管理员、独立或混合集群的步骤如下：

1.  修改在集群创建过程中使用的集群配置文件，以将 Anthos 在裸金属集群的版本从现有版本更改为要升级到的版本。以下是一个升级到版本 1.13 的示例配置：

    ```
    apiVersion: baremetal.cluster.gke.io/v1
    kind: Cluster
    metadata:
     name: admin-cluster
     namespace: cluster-admin-cluster
    spec:
     type: hybrid
     # Anthos cluster version.
     anthosBareMetalVersion: 1.13.0
    ```

1.  下载所需的 bmctl 工具版本（您要将集群升级到的版本）：

    ```
     gs://anthos-baremetal-release/bmctl/<VERSION>/linux-amd64/bmctl
    ```

1.  执行以下命令以升级集群

    ```
    bmctl upgrade cluster -c CLUSTER_NAME --kubeconfig ADMIN_KUBECONFIG
    ```

其中，CLUSTER_NAME 是集群名称，ADMIN_KUBECONFIG 是由安装创建的 kubeconfig 文件。

### 17.5.2 升级用户集群

在您升级了管理员或混合集群之后，可以按照以下步骤升级用户集群：

1.  如同管理员/混合集群配置文件的做法，修改用户集群配置文件，以将 Anthos 在裸金属集群的版本从现有版本更改为要升级到的版本。

1.  执行以下命令以升级集群版本

    ```
    bmctl upgrade cluster -c CLUSTER_NAME --kubeconfig ADMIN_KUBECONFIG
    ```

    其中，CLUSTER_NAME 是要升级的用户集群名称，而 ADMIN_KUBECONFIG 是由第一个管理员/混合集群安装创建的 kubeconfig 文件。

## 摘要

+   Anthos 在裸金属上允许组织在非 VMware 平台上部署 Anthos，包括裸金属或替代虚拟机管理程序。

+   在使用裸金属安装时，提供了不同的部署选项，包括管理员/用户、混合和独立集群。

+   Anthos 在裸金属上提供了多种负载均衡选择，包括使用外部负载均衡器（称为手动模式）或包含的选项（称为捆绑模式）。捆绑模式将为控制平面部署 HAProxy 解决方案，并为工作负载部署 MetalLB。

+   裸金属安装提供的默认存储选项仅限于本地主机存储，并且应仅用于开发集群。

+   在裸金属上安装 Anthos 提供了一个简单的部署和升级过程，使用几个自文档化的配置文件，这些文件通过单个可执行文件 bmctl 部署。
