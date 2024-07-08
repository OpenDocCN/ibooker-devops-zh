# 第三章：部署 Kubernetes 集群

现在您已经成功构建了一个应用容器，下一步是学习如何将其转变为一个完整、可靠、可扩展的分布式系统。为了做到这一点，您需要一个工作中的 Kubernetes 集群。在这一点上，大多数公共云都提供了基于云的 Kubernetes 服务，只需通过几条命令行指令即可轻松创建集群。如果您刚开始接触 Kubernetes，我们强烈推荐这种方法。即使最终计划在裸机上运行 Kubernetes，这也是一个快速开始学习 Kubernetes、了解如何在物理机器上安装它的好方法。此外，管理一个 Kubernetes 集群本身就是一项复杂的任务，对于大多数人来说，将这种管理任务推迟到云端是有意义的，特别是当云端的管理服务在大多数云中都是免费的时候。

当然，使用基于云的解决方案需要支付这些基于云的资源的费用，并且需要与云端保持活动的网络连接。出于这些原因，本地开发可能更具吸引力，在这种情况下，`minikube`工具提供了在本地笔记本电脑或台式机的虚拟机上快速启动本地 Kubernetes 集群的简便方法。虽然这是一个不错的选择，但`minikube`只创建一个单节点集群，这并不能完全展示出完整 Kubernetes 集群的所有方面。因此，我们建议人们从基于云的解决方案开始，除非真的不适合他们的情况。一个较新的选择是在单台机器上运行 Docker-in-Docker 集群，这可以在单台机器上快速启动多节点集群。尽管这是一个不错的选择，但请记住，这个项目仍处于测试阶段，可能会遇到意外的问题。

如果您确实坚持要从裸机开始，请参阅本书末尾的附录（app01.xhtml#rpi_cluster），了解如何使用一系列树莓派单板计算机构建集群的说明。这些说明使用`kubeadm`工具，并可以适配树莓派之外的其他机器。

# 在公共云提供商上安装 Kubernetes

本章介绍了在三大主要云提供商（Google Cloud Platform、Microsoft Azure 和 Amazon Web Services）上安装 Kubernetes 的方法。

如果您选择使用云提供商来管理 Kubernetes，您只需要安装其中一种选项；一旦配置好一个集群并准备就绪，您可以跳到“Kubernetes 客户端”，除非您希望在其他地方安装 Kubernetes。

## 使用 Google Kubernetes Engine 安装 Kubernetes

Google Cloud Platform（GCP）提供了一种托管的 Kubernetes 即服务，称为 Google Kubernetes Engine（GKE）。要开始使用 GKE，您需要一个启用计费的 Google Cloud Platform 账户，并安装了[`gcloud`工具](https://oreil.ly/uuUQD)。

一旦安装了`gcloud`，设置默认区域：

```
$ gcloud config set compute/zone us-west1-a
```

然后你可以创建一个集群：

```
$ gcloud container clusters create kuar-cluster --num-nodes=3
```

这将花费几分钟时间。集群就绪后，您可以使用以下命令获取集群的凭据：

```
$ gcloud container clusters get-credentials kuar-cluster
```

如果遇到问题，您可以在[Google Cloud Platform 文档](https://oreil.ly/HMwnD)中找到创建 GKE 集群的完整说明。

## 使用 Azure Kubernetes Service 安装 Kubernetes

Microsoft Azure 作为 Azure 容器服务的一部分提供托管的 Kubernetes 即服务。开始使用 Azure 容器服务的最简单方法是使用 Azure 门户中内置的 Azure Cloud Shell。您可以通过单击右上角工具栏中的 shell 图标来激活 Shell：

![kur3 03in01](img/kur3_03in01.png)

Shell 已经自动安装并配置了`az`工具，可以与您的 Azure 环境一起使用。

或者，您可以在本地计算机上安装[`az` CLI](https://oreil.ly/xpLCa)。

当您的 Shell 准备就绪时，可以运行以下命令：

```
$ az group create --name=kuar --location=westus
```

创建资源组后，您可以使用以下命令创建集群：

```
$ az aks create --resource-group=kuar --name=kuar-cluster
```

这将花费几分钟时间。创建集群后，您可以使用以下命令获取集群的凭据：

```
$ az aks get-credentials --resource-group=kuar --name=kuar-cluster
```

如果您尚未安装`kubectl`工具，可以使用以下命令安装它：

```
$ az aks install-cli
```

您可以在[Azure 文档](https://oreil.ly/hsLWA)中找到在 Azure 上安装 Kubernetes 的完整说明。

## 在 Amazon Web Services 上安装 Kubernetes

亚马逊提供了一个名为弹性 Kubernetes 服务（EKS）的托管 Kubernetes 服务。创建 EKS 集群的最简单方法是使用[开源`eksctl`命令行工具](https://eksctl.io)。

一旦安装并配置了`eksctl`并将其添加到路径中，您可以运行以下命令创建集群：

```
$ eksctl create cluster
```

有关安装选项的更多详细信息（如节点大小等），请使用此命令查看帮助：

```
$ eksctl create cluster --help
```

集群安装包括`kubectl`命令行工具的正确配置。如果您尚未安装`kubectl`，请按照[文档](https://oreil.ly/rorrD)中的说明操作。

# 使用 minikube 本地安装 Kubernetes

如果需要本地开发体验或不想支付云资源费用，可以使用`minikube`安装一个简单的单节点集群。或者，如果已经安装了 Docker Desktop，则它附带了一个单机安装的 Kubernetes。

虽然`minikube`（或 Docker Desktop）是 Kubernetes 集群的良好模拟，但它实际上是为本地开发、学习和实验设计的。因为它只在单节点的虚拟机上运行，所以不提供分布式 Kubernetes 集群的可靠性。此外，本书中描述的某些功能需要与云提供商集成。这些功能在`minikube`中要么不可用，要么在有限的方式下工作。

###### 注意

你需要在机器上安装一个虚拟化软件来使用 `minikube`。对于 Linux 和 macOS，通常是 [VirtualBox](https://virtualbox.org)。在 Windows 上，默认选择是 Hyper-V 虚拟化软件。确保在使用 `minikube` 之前安装虚拟化软件。

你可以在 [GitHub](https://oreil.ly/iHcuV) 上找到 `minikube` 工具。有适用于 Linux、macOS 和 Windows 的二进制文件可供下载。安装了 `minikube` 工具后，你可以使用以下命令创建一个本地集群：

```
$ minikube start
```

这将创建一个本地虚拟机，配置 Kubernetes 并创建一个本地的 `kubectl` 配置，指向该集群。如前所述，该集群只有一个节点，因此虽然它很有用，但与大多数生产部署的 Kubernetes 有一些不同。

当你完成了对集群的使用，可以停止虚拟机：

```
$ minikube stop
```

如果你想删除集群，可以运行：

```
$ minikube delete
```

# 在 Docker 中运行 Kubernetes

一种不同的运行 Kubernetes 集群的方法是最近开发的，它使用 Docker 容器来模拟多个 Kubernetes 节点，而不是在虚拟机中运行所有内容。[kind 项目](https://kind.sigs.k8s.io) 提供了在 Docker 中启动和管理测试集群的良好体验。(*kind* 代表 Kubernetes IN Docker.) kind 目前仍在开发中（预 1.0），但被那些构建用于快速和简单测试的 Kubernetes 工具广泛使用。

你可以在 [kind 网站](https://oreil.ly/EOgJn) 找到适合你平台的安装说明。一旦安装完成，创建集群就像这样简单：

```
$ kind create cluster --wait 5m
$ export KUBECONFIG="$(kind get kubeconfig-path)"
$ kubectl cluster-info
$ kind delete cluster
```

# Kubernetes 客户端

官方的 Kubernetes 客户端是 `kubectl`：一个用于与 Kubernetes API 交互的命令行工具。`kubectl` 可用于管理大多数 Kubernetes 对象，例如 Pods、ReplicaSets 和 Services。`kubectl` 还可以用来探索和验证集群的整体健康状态。

我们将使用 `kubectl` 工具来探索刚刚创建的集群。

## 检查集群状态

第一件事是检查你正在运行的集群的版本：

```
$ kubectl version
```

这将显示两个不同的版本：本地 `kubectl` 工具的版本以及 Kubernetes API 服务器的版本。

###### 注意

如果这些版本不同也不要担心。只要保持工具和集群的两个次要版本号在两个范围内，并且不尝试在较旧的集群上使用更新的功能，Kubernetes 工具与 Kubernetes API 的各个版本都向后兼容和向前兼容。Kubernetes 遵循语义化版本规范，次要版本是中间的数字（例如，在 1.18.2 中是 18）。但是，你需要确保你在支持的版本偏差范围内，该范围为三个版本。如果不是，可能会遇到问题。

现在我们已经确认你可以与 Kubernetes 集群通信，我们将更深入地探索该集群。

首先，您可以为集群获取一个简单的诊断。这是验证您的集群通常是否健康的好方法：

```
$ kubectl get componentstatuses
```

输出应该如下所示：

```
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
```

###### 注意

随着 Kubernetes 的变化和改进，`kubectl` 命令的输出有时会发生变化。如果输出与本书示例中显示的不完全相同，不必担心。

您可以在这里看到组成 Kubernetes 集群的各个组件。`controller-manager` 负责运行各种控制器，以调节集群中的行为，例如确保服务的所有副本可用且健康。`scheduler` 负责将不同的 Pod 放置到集群中的不同节点上。最后，`etcd` 服务器是集群的存储，其中存储着所有的 API 对象。

## 列出 Kubernetes 节点

接下来，您可以列出集群中的所有节点：

```
$ kubectl get nodes
NAME     STATUS   ROLES                  AGE     VERSION
kube0    Ready    control-plane,master   45d     v1.22.4
kube1    Ready    <none>                 45d     v1.22.4
kube2    Ready    <none>                 45d     v1.22.4
kube3    Ready    <none>                 45d     v1.22.4

```

您可以看到这是一个已经运行了 45 天的四节点集群。在 Kubernetes 中，节点被分为包含 API 服务器、调度程序等容器的 `control-plane` 节点，这些节点管理集群，并且 `worker` 节点上运行您的容器。Kubernetes 通常不会将工作调度到 `control-plane` 节点上，以确保用户工作负载不会损害集群的整体运行。

您可以使用 `kubectl describe` 命令获取有关特定节点（如 `kube1`）的更多信息：

```
$ kubectl describe nodes kube1
```

首先，您会看到节点的基本信息：

```
Name:                   kube1
Role:
Labels:                 beta.kubernetes.io/arch=arm
                        beta.kubernetes.io/os=linux
                        kubernetes.io/hostname=node-1
```

您可以看到此节点正在运行 Linux 操作系统，使用的是 ARM 处理器。

接下来，您会看到关于 `kube1` 本身操作的信息（本输出已删除日期以简明起见）：

```
Conditions:
  Type                 Status  ...   Reason                       Message
 -----                 ------        ------                       -------
  NetworkUnavailable   False   ...   FlannelIsUp                  Flannel...
  MemoryPressure       False   ...   KubeletHasSufficientMemory   kubelet...
  DiskPressure         False   ...   KubeletHasNoDiskPressure     kubelet...
  PIDPressure          False   ...   KubeletHasSufficientPID      kubelet...
  Ready                True    ...   KubeletReady                 kubelet...
```

这些状态显示节点具有足够的磁盘和内存空间，并向 Kubernetes 主节点报告其健康状态。接下来，有关机器容量的信息：

```
Capacity:
 alpha.kubernetes.io/nvidia-gpu:        0
 cpu:                                   4
 memory:                                882636Ki
 pods:                                  110
Allocatable:
 alpha.kubernetes.io/nvidia-gpu:        0
 cpu:                                   4
 memory:                                882636Ki
 pods:                                  110
```

然后是有关节点上软件的信息，包括正在运行的 Docker 版本、Kubernetes 和 Linux 内核的版本等：

```
System Info:
  Machine ID:                 44d8f5dd42304af6acde62d233194cc6
  System UUID:                c8ab697e-fc7e-28a2-7621-94c691120fb9
  Boot ID:                    e78d015d-81c2-4876-ba96-106a82da263e
  Kernel Version:             4.19.0-18-amd64
  OS Image:                   Debian GNU/Linux 10 (buster)
  Operating System:           linux
  Architecture:               amd64
  Container Runtime Version:  containerd://1.4.12
  Kubelet Version:            v1.22.4
  Kube-Proxy Version:         v1.22.4
PodCIDR:                      10.244.1.0/24
PodCIDRs:                     10.244.1.0/24
```

最后，有关当前在此节点上运行的 Pods 的信息：

```
Non-terminated Pods:            (3 in total)
  Namespace   Name        CPU Requests CPU Limits Memory Requests Memory Limits
  ---------   ----        ------------ ---------- --------------- -------------
  kube-system kube-dns...  260m (6%)    0 (0%)     140Mi (16%)     220Mi (25%)
  kube-system kube-fla...  0 (0%)       0 (0%)     0 (0%)          0 (0%)
  kube-system kube-pro...  0 (0%)       0 (0%)     0 (0%)          0 (0%)
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.
  CPU Requests  CPU Limits      Memory Requests Memory Limits
  ------------  ----------      --------------- -------------
  260m (6%)     0 (0%)          140Mi (16%)     220Mi (25%)
No events.
```

从此输出中，您可以看到节点上的 Pods（例如为集群提供 DNS 服务的 `kube-dns` Pod）、每个 Pod 从节点请求的 CPU 和内存，以及所请求的总资源。值得注意的是，Kubernetes 跟踪每个 Pod 请求的资源和上限。关于请求和限制之间的差异在 第五章 中有详细描述，但简而言之，Pod 请求的资源保证在节点上存在，而 Pod 的限制是 Pod 可以消耗的给定资源的最大量。如果 Pod 的限制高于其请求，则额外的资源将按最佳努力原则提供。不能保证这些资源在节点上存在。

# 集群组件

Kubernetes 的一个有趣的方面是，构成 Kubernetes 集群的许多组件实际上是使用 Kubernetes 自身部署的。我们将介绍其中一些。这些组件使用了我们后面章节将介绍的多个概念。所有这些组件都在 `kube-system` 命名空间中运行。^(1)

## Kubernetes Proxy

Kubernetes 代理负责将网络流量路由到 Kubernetes 集群中负载均衡的服务。为了完成其工作，代理必须存在于集群中的每个节点上。Kubernetes 有一个名为 DaemonSet 的 API 对象，你将在第十一章中学习，许多集群都使用它来完成这项任务。如果你的集群使用 DaemonSet 运行 Kubernetes 代理，可以通过运行以下命令查看代理：

```
$ kubectl get daemonSets --namespace=kube-system kube-proxy
NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
kube-proxy   5         5         5       5            5           ...   45d
```

根据集群的设置方式，`kube-proxy` 的 DaemonSet 可能会有其他名称，或者可能根本不使用 DaemonSet。无论如何，`kube-proxy` 容器应该在集群中的所有节点上运行。

## Kubernetes DNS

Kubernetes 还运行一个 DNS 服务器，为集群中定义的服务提供命名和发现功能。这个 DNS 服务器也作为集群中的一个复制服务运行。根据集群的大小，你可能会看到一个或多个运行在集群中的 DNS 服务器。DNS 服务作为一个 Kubernetes 部署运行，管理这些副本（可能也被命名为 `coredns` 或其他变种）：

```
$ kubectl get deployments --namespace=kube-system core-dns
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
core-dns   1         1         1            1           45d
```

Kubernetes 还有一个执行 DNS 服务器负载平衡的 Kubernetes 服务：

```
$ kubectl get services --namespace=kube-system core-dns
NAME       CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
core-dns   10.96.0.10   <none>  53/UDP,53/TCP   45d
```

这显示了集群的 DNS 服务地址为 10.96.0.10。如果你登录到集群中的一个容器中，你会看到这个地址已经被填写到了容器的 */etc/resolv.conf* 文件中。

## Kubernetes UI

如果你想在图形用户界面中可视化你的集群，大多数云提供商都会在其云的 GUI 中集成这样的可视化功能。如果你的云提供商没有提供这样的 UI，或者你更喜欢一个集群内的 GUI，可以安装一个由社区支持的 GUI。查看[文档](https://oreil.ly/wKfEx)了解如何为这些集群安装仪表板。你也可以使用像 Visual Studio Code 这样的开发环境扩展来一览你的集群状态。

# 摘要

希望到目前为止你已经建立并运行了一个 Kubernetes 集群（或三个），并且使用了一些命令来探索你创建的集群。接下来，我们将花一些时间探索 CLI，以及教你如何掌握 `kubectl` 工具。在本书的其余部分，你将使用 `kubectl` 和你的测试集群来探索 Kubernetes API 中的各种对象。

^(1) 正如你将在下一章中学到的，Kubernetes 中的命名空间是用来组织 Kubernetes 资源的实体。你可以把它想象成文件系统中的一个文件夹。
