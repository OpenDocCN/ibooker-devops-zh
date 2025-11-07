# 14 节点和 Kubernetes 安全

本章涵盖

+   节点加固和 Pod 清单

+   API 服务器安全，包括 RBAC

+   用户身份验证和授权

+   开放策略代理（OPA）

+   Kubernetes 中的多租户

在上一章中，我们刚刚完成了 Pod 的安全设置；现在我们将讨论 Kubernetes 节点的安全。在本章中，我们将包括更多关于节点安全的信息，这些信息与对节点和 Pod 的可能攻击有关，并提供带有多种配置的完整示例。

## 14.1 节点安全

在 Kubernetes 中保护节点类似于保护任何其他虚拟机或数据中心服务器。我们将从传输层安全（TLS）证书开始介绍。这些证书允许保护节点，但我们还将探讨与镜像不可变性、工作负载、网络策略等相关的问题。将本章视为一个按需菜单，其中包含你至少应该考虑在生产中运行 Kubernetes 的重要安全主题。

### 14.1.1 TLS 证书

Kubernetes 中的所有外部通信通常都通过 TLS 进行，尽管这可以配置。然而，TLS 有许多不同的版本。因此，你可以为 Kubernetes API 服务器选择一个加密套件。大多数安装程序或自托管的 Kubernetes 版本将为你处理 TLS 证书的创建。*加密套件*是一系列算法的集合，总体上允许 TLS 安全地进行。定义 TLS 算法包括

+   *密钥交换*——设置一个同意的密钥交换方式，用于加密/解密

+   *身份验证*——确认消息发送者的身份

+   *加密*——使消息在外部人员无法阅读的情况下隐藏

+   *消息认证*——确认消息来自有效源

在 Kubernetes 中，你可能会找到以下加密套件：`TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384`。让我们来分解一下。在这个字符串中，每个下划线（`_`）将一个算法与下一个算法分开。例如，如果我们设置 API 服务器中的`--tls-cipher-suites`为类似

```
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
```

我们可以在[`mng.bz/nYZ8`](http://mng.bz/nYZ8)上查找这个特定的协议，然后确定通信协议的工作方式。例如：

+   *协议*——传输层安全（TLS）

+   *密钥交换*——椭圆曲线迪菲-赫尔曼临时（ECDHE）

+   *身份验证*——椭圆曲线数字签名算法（ECDSA）

+   *加密*——通过加密块链模式（AES 256 CBC）使用 256 位密钥的高级加密标准

这些协议的具体细节超出了本书的范围，但重要的是要注意，你需要监控你的 TLS 安全状态，特别是如果它是由你组织中的更大标准机构设置的，以确认你的 Kubernetes 安全模型与组织中的 TLS 标准相一致。例如，要更新任何 Kubernetes 服务使用的加密套件，在启动时发送`tls-cipher-suites`参数：

```
--tls-cipher-suites=TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA,
➥ TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
```

将此添加到你的 API 服务器中可以确保它只通过此密钥套件连接到其他服务。如图所示，你可以通过添加逗号来分隔值以支持多个密钥套件。任何服务的帮助页面都提供了一个全面的套件列表（例如，[`mng.bz/voZq`](http://mng.bz/voZq)显示了 Kubernetes 调度器`kube-scheduler`的帮助）。还应注意，

+   如果一个 TLS 密钥被发现存在漏洞，你将希望更新 Kubernetes API 服务器、调度器、控制器管理器和 kubelet 中的密钥套件。这些组件以某种方式通过 TLS 提供服务。

+   如果你的组织不允许某些密钥套件，你应该明确删除这些套件。

注意：如果你过于简化允许进入你的 API 服务器的密钥套件，你可能会面临某些类型的客户端无法连接到它的风险。例如，Amazon ELB 有时会使用 HTTPS 健康检查来确保在转发流量之前端点是可用的，但它们不支持 Kubernetes API 服务器中使用的某些常见 TLS 密钥。AWS 负载均衡器 API 的版本 1 仅支持非椭圆曲线密钥算法，例如`TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256`。这里的结果可能是灾难性的；你的整个集群将完全无法工作！因为通常会在单个 ELB 后面放置多个 API 服务器，所以请记住，随着时间的推移，TCP 健康检查（而不是 HTTPS）可能更容易管理，特别是如果你需要在 API 服务器上使用特殊的安全密钥的话。

### 14.1.2 不可变操作系统与修补节点

不可变性是你无法更改的东西。一个*不可变*的操作系统由只读组件和二进制文件组成，你不能对其进行修补。与其修补和更新软件，不如通过从服务器或云中擦除操作系统来替换整个操作系统，然后删除虚拟机并创建一个新的。Kubernetes 通过允许管理员更容易地将工作负载从节点上移除（因为这是一个内置功能）来简化了不可变操作系统的运行。

与使用你的发行版的包管理器自动应用补丁的系统相比，使用不可变操作系统。拥有一个 Kubernetes 集群消除了定制*雪花服务器*的概念。这些服务器运行特定的应用程序，并用一个标准化的节点替换它们。作为下一个逻辑步骤，运行不可变操作系统。将漏洞注入可变系统的一种最简单的方法是替换一个常见的 Linux 库。

不可变操作系统是只读的，由于它们是只读的，因此无法进行特定的更改，因为你不能将这些更改写入磁盘，这减少了我们的暴露。使用不可变的发行版消除了这些机会中的许多。一般来说，Kubernetes 控制平面（API 服务器、控制器管理器和调度器）节点将具有

+   kubelet 二进制文件

+   `kube-proxy`二进制文件

+   containerd 或其他容器运行时可执行文件

+   etcd 的镜像

所有这些功能都是为了快速启动而内置的。同时，Kubernetes 工作节点将拥有相同的组件，除了 etcd。这是一个重要的区别，因为 etcd 在 Windows 环境中不支持运行，但一些用户可能希望运行 Windows 工作节点来运行 Windows Pod。

为 Windows 工作节点构建自定义镜像非常重要，因为 Windows 操作系统不可分发，因此最终用户必须构建 Windows kubelet 镜像，如果他们想使用不可变部署模型。要了解更多关于不可变镜像的信息，您可以评估 Tanzu Community Edition 项目（[`tanzu.vmware.com/tanzu/community`](https://tanzu.vmware.com/tanzu/community)）。该项目旨在为更广泛的社区提供一个“包含电池”的方法来使用不可变镜像，并配合 Cluster API 启用可用的、生产级别的集群。许多其他托管 Kubernetes 服务，包括 Google 的 GKE，也使用不可变操作系统。

### 14.1.3 隔离容器运行时

容器非常出色，但它们并不能完全隔离进程与操作系统。Docker 引擎（以及其他容器引擎）并没有完全将运行中的容器与 Linux 内核沙箱化。容器与宿主机之间没有强大的安全边界，因此如果宿主机的内核存在漏洞，容器可能能够访问 Linux 内核漏洞并利用它。Docker 引擎利用 Linux 命名空间来隔离进程，例如直接访问 Linux 网络堆栈，但仍然存在漏洞。例如，宿主机的 /sys 和 /proc 文件系统仍然被容器内运行的过程读取。

类似于 gVisor、IBM Nabla、Amazon Firecracker 和 Kata 这样的项目提供了一种虚拟 Linux 内核，它将容器进程与宿主机的内核隔离开来，从而提供了一个更真实的沙箱环境。这些项目在开源领域相对较新，至少目前还没有在 Kubernetes 环境中得到广泛使用。这些只是其中一些相当成熟的项目，因为 gVisor 被用作 Google Cloud Platform 的一部分，而 Firecracker 被用作 Amazon Web Services 平台的一部分。也许在你阅读这篇文章的时候，更多的 Kubernetes 集群容器将运行在虚拟内核之上！我们甚至可以思考启动微虚拟机作为 Pod。我们正生活在有趣的时代！

### 14.1.4 资源攻击

Kubernetes 节点拥有有限的资源，包括 CPU、内存和磁盘。我们在集群上运行了许多 Pod、`kube-proxy`、kubelets 以及其他 Linux 进程。节点通常有一个 CNI 提供商、一个日志守护进程以及其他支持集群的进程。您需要确保 Pod 中的容器不会耗尽节点资源。如果您不提供限制，那么一个容器可能会耗尽节点资源，影响所有其他系统。本质上，一个 *失控的容器进程* 可以对一个节点执行拒绝服务（DoS）攻击。这就是资源限制的用武之地……

资源限制通过实现三个不同的 API 级对象和配置来控制。Pod API 对象可以具有控制每个限制的设置。例如，以下 YAML 段落提供了 CPU、内存和磁盘空间使用限制：

```
apiVersion: v1
kind: Pod
metadata:
  name: core-kube-limited
spec:
  containers:
  - name: app
    image:
    resources:
      requests:                   ❶
       memory: "42Mi"
        cpu: "42m"
        ephemeral-storage: "21Gi"
      limits:                     ❷
       memory: "128Mi"
        cpu: "84m"
        ephemeral-storage: "42Gi"
```

❶ 提供初始的 CPU、内存或存储量

❷ 设置允许的最大 CPU、内存和存储量

在安全方面，如果这些值中的任何一个超过了限制，Pod 将会被重启。而且，如果再次超过限制，那么 Pod 将被终止，并且不会再次启动。

另一个有趣的事情是，资源 `requests` 和 `limits` 也会影响 Pod 在节点上的调度。托管 Pod 的节点必须具有满足调度器选择托管 Pod 的节点的初始请求的资源。您可能会注意到我们正在使用单位来表示请求和限制内存、CPU 和临时存储值。

### 14.1.5 CPU 单位

要测量 CPU，Kubernetes 使用的基准单位是 1，这相当于裸金属上的一个超线程或在云中的单个核心/vCPU。您也可以用小数表示 CPU 单位；例如，您可以有 0.25 个 CPU 单位。此外，API 还允许您将 0.25 个十进制 CPU 单位转换为 250 m。所有这些段落都适用于 CPU：

```
resources:
  requests:
    cpu: "42"    ❶
resources:
  requests:
  cpu: "0.42"    ❷
resources:
  requests:
    cpu: "420m"  ❸
```

❶ 设置 42 个 CPU（这是一个大服务器！）

❷ 0.42 个 CPU，以 1 为单位进行测量

❸ 这与上一个代码块中的 0.42 相同。

### 14.1.6 内存单位

内存以字节、整数和固定点数进行测量，使用以下后缀：*E*、*P*、*T*、*G*、*M* 或 *K*。或者您可以使用 *Ei*、*Pi*、*Ti*、*Gi*、*Mi* 或 *Ki*，它们代表二进制的等效值。以下段落大致都是相同的值：

```
resources:
  requests:
    memory: "128974848"    ❶
resources:
  requests:
    memory: "129e6"        ❷
```

❶ 字节纯数字表示（128,974,848 字节）

❷ 129e6 有时写作 129e+6 的科学记数法：129e+6 == 129000000。这个段落代表 129,000,000 字节。

下一个段落处理的是典型的兆比特与兆字节的转换：

```
resources:
  requests:
    memory: "129M"      ❶
```

❶ 129 兆比特 == 1.613e+7 字节，接近 129e+6 的值。

接下来是兆字节：

```
resources:
  requests:
    memory: "123Mi"      ❶
```

❶ 123 兆字节 == 1.613e+7 字节，接近 129e+6 的值。

### 14.1.7 存储单位

最新的 API 配置是临时存储请求和限制。临时存储限制适用于三个存储组件：

+   emptyDir 卷，除了 tmpfs

+   存放节点级日志的目录

+   可写容器层

当超过限制时，kubelet 会驱逐 Pod。每个节点都配置了最大临时存储量，这再次影响了 Pod 到节点的调度。还有一个用户可以指定特定节点限制的“扩展资源”限制。你可以在 Kubernetes 文档中找到有关扩展资源的更多详细信息。

### 14.1.8 主机网络与 Pod 网络比较

在第 14.4 节中，我们将介绍 NetworkPolicies。这些政策允许你使用 CNI 提供者锁定 Pod 通信，通常，这些政策为你实现。然而，你应该考虑一种更基本的网络安全类型：不要将 Pod 运行在与你的主机相同的网络上。这立即

+   限制外部世界对您的 Pod 的访问

+   限制您的 Pod 对主机网络端口的访问

如果一个 Pod 加入主机网络，将允许 Pod 更容易地访问节点，从而在攻击时增加爆炸半径。如果一个 Pod 不需要在主机网络上运行，请不要在主机网络上运行 Pod！如果 Pod 需要在主机网络上运行，那么请不要将该 Pod 暴露给互联网。以下代码片段是一个在主机网络上运行的 Pod 的部分 YAML 定义。如果 Pod 正在执行管理任务，如日志记录或网络（一个 CNI 提供者），你通常会看到在主机网络上运行的 Pod：

```
apiVersion: v1
kind: Pod
metadata:
  name: host-Pod
spec:
  hostNetwork: true
```

### 14.1.9 Pod 示例

我们已经介绍了不同的 Pod API 配置：服务账户令牌、CPU 和其他资源设置、安全上下文等。以下是一个包含所有配置的示例：

```
apiVersion: v1
kind: Pod
metadata:
  name: example-Pod
spec:
  automountServiceAccountToken: false   ❶
 securityContext:                       ❷
   runAsUser: 3042
    runAsGroup: 4042
    fsGroup: 5042
    capabilities:
      add: ["NET_ADMIN"]
  hostNetwork: true                     ❸
 volumes:
  - name: sc-vol
     emptyDir: {}
  containers:
  - name: sc-container
    image: my-container
    resources:                          ❹
     requests:
        memory: "42Mi"
        cpu: "42m"
        ephemeral-storage: "1Gi"
      limits:
        memory: "128Mi"
        cpu: "84m"
        ephemeral-storage: "2Gi"
    volumeMounts:
    - name: sc-vol
      mountPath: /data/foo
  serviceAccountName: network-sa        ❺
```

❶ 禁用服务账户令牌的自动挂载

❷ 设置安全上下文并赋予 NET_ADMIN 能力

❸ 在主机网络上运行

❹ 设置资源限制

❺ 给 Pod 分配一个特定的服务账户

## 14.2 API 服务器安全

像二进制认证这样的组件使用准入控制器提供的 webhooks。各种控制器是 Kubernetes API 服务器的一部分，并创建一个 webhook 作为事件的入口点。例如，ImagePolicyWebhook 是允许系统响应 webhook 并对容器做出准入决定的插件之一。如果一个 Pod 没有通过准入标准，它将保持挂起状态——它不会被部署到集群中。准入控制器可以验证集群中正在创建的 API 对象，修改或更改这些对象，或者两者都做。从安全角度来看，这为集群提供了大量的控制和审计能力。

### 14.2.1 基于角色的访问控制（RBAC）

首先，在您的集群上启用基于角色的访问控制（RBAC）是必要的。目前，大多数安装程序和云托管提供商都启用了 RBAC。Kubernetes API 服务器使用 `--authorization-mode=RBAC` 标志来启用 RBAC。如果您使用的是 Kubernetes 的自托管版本，如 GKE，则 RBAC 已启用。作者确信存在一种边缘情况，即运行 RBAC 无法满足业务需求。然而，在其余 99% 的情况下，您需要启用 RBAC。

RBAC 是一种基于角色的安全机制，它控制用户和系统对资源的访问。它通过角色和权限仅限制授权用户和服务账户对资源的访问。这如何应用于 Kubernetes？您想用 Kubernetes 保护的最关键组件之一是 API 服务器。当系统用户通过 API 服务器对集群具有管理员访问权限时，该用户可以清理节点、删除对象，并造成极大的破坏。在 Kubernetes 中，管理员是在集群上下文中的 root 用户。

RBAC 是一个强大的安全组件，它提供了在集群内限制 API 访问的巨大灵活性。因为它是一个强大的机制，所以它也有通常的副作用，即有时相当复杂且难以调试。

注意：在 Kubernetes 中运行的平均 Pod 不应具有访问 API 服务器的权限，因此您应该禁用服务账户令牌的挂载。

### 14.2.2 RBAC API 定义

RBAC API 定义了以下类型：

+   *Role*—包含一组权限，限制在命名空间内

+   *ClusterRole*—包含一组集群范围内的权限

+   *RoleBinding*—将角色授予用户或组

+   *ClusterRole*—将 ClusterRole 授予用户或组

在 Role 和 ClusterRole 定义中，有几个定义的组件。我们将首先介绍的是 *动词*，它包括 API 和 HTTP 动词。API 服务器内的对象可以有一个 get 请求；因此，有 get 请求动词定义。我们经常从创建 REST 服务时定义的创建、读取、更新和删除（CRUD）动词的角度来考虑这个问题。您可以使用的动词包括

+   *资源请求的 API 请求动词*—get、list、create、update、patch、watch、proxy、redirect、delete 和 deletecollection

+   *非资源请求的 HTTP 请求动词*—get、post、put 和 delete

例如，如果您想使操作员能够监视和更新 Pods，您可以

1.  定义资源（在这种情况下，是一个 Pod）

1.  定义角色可以访问的动词（最可能是 list 和 patch）

1.  定义 API 组（使用空字符串表示核心 API 组）

你已经熟悉 API 组了，因为它们是出现在 Kubernetes 清单中的 `apiVersion` 和 `kind`。API 组遵循 API 服务器中的 REST 路径（/apis/$GROUP_NAME/$VERSION）并使用 `apiVersion $GROUP_NAME/$VERSION`（例如，`batch/v1`）。不过，让我们保持简单，暂时不处理 API 组。我们将从核心 API 组开始。以下是一个特定命名空间的角色的示例。由于角色仅限于命名空间，这提供了对 Pod 资源执行列表和补丁动词的访问权限：

```
# Create a custom role in the default namespace that grants access to
# list, and patch Pods
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: Pod-labeler
  namespace: rbac-example
rules:
- apiGroups: [""] # "" refers to the core API group
  resources: ["Pods"] # the
  verbs: ["list", "patch"] # authorization to use list and patch verbs
```

对于前面的示例，我们可以定义一个服务账户来使用该片段中的角色，如下所示：

```
# Create a ServiceAccount that will be bound to the role above
apiVersion: v1
kind: ServiceAccount
  metadata:
    name: Pod-labeler
    namespace: rbac-example
```

之前的 YAML 创建了一个服务账户，该账户可以被 Pod 使用。接下来，我们将创建一个角色绑定，将之前定义的服务账户与之前定义的角色连接起来：

```
# Binds the Pod-labeler ServiceAccount to the Pod-labeler Role
# Any Pod using the Pod-labeler ServiceAccount will be granted
# API permissions based on the Pod-labeler role.
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: Pod-labeler
  namespace: rbac-example
subjects:
  # List of service accounts to bind
- kind: ServiceAccount
  name: Pod-labeler
roleRef:
  # The role to bind
  kind: Role
  name: Pod-labeler
  apiGroup: rbac.authorization.k8s.io
```

现在，你可以在分配给该 Pod 的服务账户的部署中启动 Pod：

```
# Deploys a single Pod to run the Pod-labeler code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: Pod-labeler
  namespace: rbac-example
spec:
  replicas: 1

  # Control any Pod labeled with app=Pod-labeler
  selector:
    matchLabels:
      app: Pod-labeler

  template:
    # Ensure created Pods are labeled with app=Pod-labeler
    # to match the deployment selector
    metadata:
      labels:
        app: Pod-labeler

    spec:
      # define the service account the Pod uses
      serviceAccount: Pod-labeler

      # Another security improvement, set the UID and GID the Pod runs with
      # Pod-level security context to define the default UID and GIDs
      # under which to run all container processes. We use 9999 for
      # all IDs because it is unprivileged and known to be unallocated
      # on the node instances.
      securityContext:
        runAsUser: 9999
        runAsGroup: 9999
        fsGroup: 9999

      containers:
      - image: gcr.io/pso-examples/Pod-labeler:0.1.5
        name: Pod-labeler
```

让我们回顾一下。我们创建了一个具有修补和列出 Pod 权限的角色。然后，我们创建了一个服务账户，以便我们可以创建 Pod 并让该 Pod 使用定义的用户。接下来，我们定义了一个角色绑定，将服务账户添加到之前定义的角色中。最后，我们启动了一个包含定义 Pod 的部署，该 Pod 使用之前定义的服务账户。

RBAC（基于角色的访问控制）虽然不复杂，但对 Kubernetes 集群的安全性至关重要。前面的 YAML 是从位于 [`mng.bz/ZzMa`](http://mng.bz/ZzMa) 的 Helmsman RBAC 演示中提取的。

### 14.2.3 资源和子资源

大多数 RBAC 资源使用单个名称，如 Pod 或 Deployment。一些资源有子资源，如下面的代码片段所示：

```
GET /api/v1/namespaces/rbac-example/Pods/Pod-labeler/log
```

此 API 端点表示 `rbac-example` 命名空间中名为 Pod-labeler 的 Pod 的子资源日志路径。定义如下：

```
GET /api/v1/namespaces/{namespace}/Pods/{name}/log
```

为了使用日志的子资源，你需要定义一个角色。以下是一个示例：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-example
  name: Pod-and-Pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["Pods", "Pods/log"]
  verbs: ["get", "list"]
```

你还可以通过命名 Pod 来进一步限制对 Pod 日志的访问。例如：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-example
  name: Pod-labeler-logs
rules:
- apiGroups: [""]
  resourceNames: ["Pod-labeler"]
  resources: ["Pods/log"]
  verbs: ["get"]
```

注意到之前的 YAML 中的 `rules` 元素是一个数组。以下代码片段显示了如何向 YAML 添加多个权限。`resources`、`resourceNames` 和 `verbs` 可以是任何组合：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-example
  name: Pod-labeler-logs
rules:
- apiGroups: [""]
  resourceNames: ["Pod-labeler"]
  resources: ["Pods/log"]
  verbs: ["get"]
- apiGroups: [""]
  resourceNames: ["another-Pod"]
  resources: ["Pods/log"]
  verbs: ["get"]
```

资源类似于 Pods 和节点，但 API 服务器还包括不是资源的元素。这些元素由 API REST 端点的实际 URI 组件定义；例如，给 `Pod-labeler-logs` RBAC 角色访问 /healthz API 端点，如下面的代码片段所示：

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: rbac-example
  name: Pod-labeler-logs
rules:
- apiGroups: [""]
  resourceNames: ["Pod-labeler"]
  resources: ["Pods/log"]
  verbs: ["get"]
- apiGroups: [""]
  resourceNames: ["another-Pod"]
  resources: ["Pods/log"]
  verbs: ["get"]
- nonResourceURLs: ["/healthz", "/healthz/*"]    ❶
  verbs: ["get", "post"]
```

❶ 非资源 URL 中的星号 (*) 是后缀通配符匹配。

### 14.2.4 主题和 RBAC

角色绑定可以包括 Kubernetes 中的用户、服务账户和组。在以下示例中，我们将创建另一个名为 `log-reader` 的服务账户，并将该服务账户添加到上一节中的角色绑定定义中。在示例中，我们还有一个名为 `james-bond` 的用户和一个名为 `MI-6` 的组：

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: Pod-labeler
  namespace: rbac-example
subjects:
  # List of service accounts to bind
- kind: ServiceAccount
  name: Pod-labeler
- kind: SeviceAccount
  name: log-reader
- kind: User
  name: james-bond
- kind: Group
  name: MI-6
roleRef:
  # The role to bind
  kind: Role
  name: Pod-labeler
  apiGroup: rbac.authorization.k8s.io
```

注意：用户和组是由为集群设置的认证策略创建的。

### 14.2.5 调试 RBAC

虽然 RBAC 很复杂，使用起来也很痛苦，但我们始终有审计日志。当启用时，Kubernetes 会记录影响集群的安全事件的审计跟踪。这些事件包括用户操作、管理员操作和/或集群内的其他组件。基本上，如果你使用了 RBAC 和其他安全组件，你会得到“谁”、“什么”、“从哪里”和“如何”的信息。审计日志是通过一个审计策略文件配置的，该文件通过`kube-apiserver` `--audit-policy-file`传递到 API 服务器。

因此，我们有了所有事件的日志——太棒了！但是等等……现在你有一个拥有数百个角色、许多用户和大量角色绑定的集群。现在你必须将所有这些数据结合起来。为此，有几个工具可以帮助我们。这些工具的共同主题是将用于定义 RBAC 访问的不同对象结合起来。为了在集群角色、RoleBindings 和主体中内省基于 RBAC 的权限，需要将主体（可以是用户、组或服务账户）结合起来。

+   *ReactiveOps*创建了一个工具，允许用户查找用户、组或服务账户当前是成员的角色。`rbac-lookup`可在[`github.com/reactiveops/rbac-lookup`](https://github.com/reactiveops/rbac-lookup)找到。

+   *另一个可以找到用户或服务账户在集群内具有哪些权限的工具是* `kubectl-rbac`*。此工具位于[`github.com/octarinesec/kubectl-rbac`](https://github.com/octarinesec/kubectl-rbac)。

+   *Jordan Liggit 有一个名为* `audit2rbac`*的开源工具。该工具接受审计日志和用户名，创建与请求访问权限相匹配的角色和角色绑定。对 API 服务器进行调用，并可以捕获日志。从那里，你可以运行`audit2rbac`来生成所需的 RBAC（换句话说，RBAC 反向工程）。

## 14.3 认证、授权和秘密

*Authn*（用户认证）和*Authz*（授权）是认证用户拥有的组和权限。我们在这里也谈论秘密可能看起来有些奇怪，但一些用于认证和授权的工具也用于秘密，并且你通常需要认证和授权来访问秘密。

首先，不要使用在安装集群时生成的默认集群管理证书。你需要一个 IAM（身份和访问管理）服务提供商来认证和授权用户。此外，不要在 API 服务器上启用用户名和密码认证；使用 TLS 证书内置的用户认证功能。

### 14.3.1 IAM 服务账户：保护你的云 API

Kubernetes 容器具有*原生云*身份（这些身份了解云）。这既美丽又可怕。如果没有你的云威胁模型，就无法为你的 Kubernetes 集群制定威胁模型。

云 IAM 服务账户构成了安全的基础，包括对人和系统的授权和认证。在数据中心内，Kubernetes 安全配置仅限于 Linux 系统、Kubernetes、网络以及部署的容器。当在云中运行 Kubernetes 时，出现了一个新的问题——节点和 Pods 的 IAM 角色：

+   IAM 是特定用户或服务账户的角色，该角色随后成为某个组的成员。

+   Kubernetes 集群中的每个节点都有一个 IAM 角色。

+   Pods 通常继承该角色。

你集群中的节点，特别是那些运行在控制平面上的节点，需要在云中运行时具有 IAM 角色。这是因为 Kubernetes 中许多原生功能来自于 Kubernetes 本身有一个如何与其云提供商通信的概念。例如，让我们从 GKE 的官方文档中汲取一些启示：Google Cloud Platform 自动创建一个名为计算引擎默认服务账户的服务账户，GKE 将其与它创建的节点关联起来。根据你的项目配置，默认服务账户可能或可能没有权限使用其他云平台 API。GKE 还向计算实例分配了一些有限的访问范围。更新默认服务账户的权限或将更多访问范围分配给计算实例不是从运行在 GKE 上的 Pods 中认证到其他云平台服务的推荐方法。

因此，你的容器在很多情况下具有与节点本身相当的权限。随着云服务逐渐为容器提供更细粒度的权限模型，这种默认设置很可能会在未来得到改善。然而，仍然需要确保 IAM 角色或角色具有最少的适用权限，并且始终存在更改这些 IAM 角色的方法。例如，当在 Google Cloud Platform 中使用 GKE 时，你必须为集群在项目中创建一个新的 IAM 角色。如果不这样做，集群通常会使用计算引擎默认服务账户，该账户具有编辑权限。

*Google Cloud 编辑器* 允许给定账户（在这种情况下，你的集群中的一个节点，这相当于可能任何 Pod）编辑该项目中任何资源。例如，你只需破坏集群中的某个 Pod，就可以删除整个数据库集群、TCP 负载均衡器或云 DNS 条目。此外，你应该删除你在 GCP 中创建的任何项目的默认服务账户。AWS、Azure 等也存在相同的问题。总之，每个集群都是使用其独特的服务账户创建的，并且该服务账户具有尽可能少的权限。使用 `kops`（Kubernetes Operations）等工具，我们可以检查 Kubernetes 集群所需的每个权限，然后 `kops` 为控制平面创建一个特定的 IAM 角色，以及为节点创建另一个角色。

### 14.3.2 云资源访问

假设你已经以最低权限配置了你的 Kubernetes 节点，现在你已经阅读了这篇文章，你可能感觉安全。实际上，如果你正在运行 AKS（Azure Kubernetes Service）这样的解决方案，你不需要担心配置控制平面，只需关注节点级别的 IAM 即可，但这还不是全部。例如，一个开发者创建了一个需要与托管云服务通信的服务——比如文件存储服务。现在运行的 Pod 需要一个具有正确角色的服务账户。有各种实现这一点的途径。

注意：AKS 可能是解决方案中最简单的，但它确实带来了一些挑战。你需要限制节点上的 Pod，只允许访问云资源，或者接受所有 Pod 现在都将具有文件存储访问的风险。

提示：使用 `kube2iam` ([`github.com/jtblin/kube2iam`](https://github.com/jtblin/kube2iam)) 或 `kiam` ([`github.com/uswitch/kiam`](https://github.com/uswitch/kiam)) 工具来实现此方法。

一些新创建的操作员可以将特定的服务账户分配给特定的 Pod。每个节点上的组件会拦截对云 API 的调用。它不是使用节点的 IAM 角色，而是将一个角色分配给集群中的 Pod，这通过注解来表示。一些托管云提供商有类似的解决方案。一些云提供商，如 Google，有可以运行并连接到云 SQL 服务的边车。边车被分配一个角色，然后代理连接到数据库的应用程序。

可能最复杂但更稳健的解决方案是使用集中式的密钥库服务器。使用这种方式，应用程序可以检索短期有效的 IAM 令牌，从而允许访问云系统。通常，会使用边车来自动刷新所使用的令牌。我们还可以使用 HashiCorp Vault 来保护非 IAM 凭证的机密信息。如果你的用例需要稳健的机密和 IAM 管理，Vault 是一个绝佳的解决方案，但正如所有关键任务解决方案一样，你需要维护和支持它。

提示：使用 HashiCorp Vault ([`www.vaultproject.io/`](https://www.vaultproject.io/))来存储密钥。

### 14.3.3 私有 API 服务器

在本节中，我们将要讨论的最后一件事是 API 服务器的网络访问。您可以选择使 API 服务器通过互联网不可访问，或者将 API 服务器放在私有网络上。如果您将 API 服务器负载均衡器放在私有网络上，您将需要利用堡垒主机、VPN 或其他形式的连接到 API 服务器，因此这种解决方案并不那么方便。

API 服务器是一个极其敏感的安全点，必须像这样进行保护。DoS 攻击或一般入侵可能会使集群瘫痪。此外，当 Kubernetes 社区发现安全问题时，它们偶尔会存在于 API 服务器中。如果可能，请将您的 API 服务器放在私有网络上，或者至少将能够连接到您的 API 服务器前端负载均衡器的 IP 地址列入白名单。

## 14.4 网络安全

再次强调，这是一个很少得到适当解决的问题的安全领域。默认情况下，Pod 网络上的 Pod 可以访问集群中任何地方的任何 Pod，这包括 API 服务器。这种能力存在是为了允许 Pod 访问像 DNS 这样的系统进行服务查找。在主机网络上运行的 Pod 几乎可以访问任何东西：所有 Pod、所有节点和 API 服务器。如果启用了端口，主机网络上的 Pod 甚至可以访问 kubelet API 端口。

*网络策略*是您可以定义以控制 Pod 之间网络流量的对象。NetworkPolicy 对象允许您配置 Pod 的进出访问。*进入*是指进入 Pod 的流量，而*流出*是指离开 Pod 的网络流量。

### 14.4.1 网络策略

您可以在任何 Kubernetes 集群上创建 NetworkPolicy 对象，但您需要一个正在运行的安全提供程序，如 Calico。Calico 是一个 CNI 提供程序，同时也提供了一个单独的应用程序来实现网络策略。如果您在没有提供程序的情况下创建网络策略，该策略将不起作用。网络策略有以下约束和功能。它们是

+   应用到 Pods

+   通过标签选择器与特定 Pod 匹配

+   控制进出网络流量

+   控制由 CIDR 范围、特定命名空间或匹配的 Pod 或 Pods 定义的网络流量

+   专门设计用于处理 TCP、UDP 和 SCTP 流量

+   能够处理命名端口或特定的 Pod 编号

让我们试试这个。为了设置一个`kind`集群并在其上安装 Calico，首先运行以下命令来创建`kind`集群，并且不要启动默认的 CNI。Calico 将在之后安装：

```
$ cat <<EOF | kind create cluster --config -
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  # the default CNI will not be installed
  disableDefaultCNI: true
  PodSubnet: "192.168.0.0/16"
EOF
```

接下来，我们安装 Calico Operator 及其自定义资源。使用以下命令来完成此操作：

```
$ kubectl create -f \
https://docs.projectcalico.org/manifests/tigera-operator.yaml
$ kubectl create -f \
https://docs.projectcalico.org/manifests/custom-resources.yaml
```

现在我们可以观察 Pod 启动。使用以下命令：

```
$ kubectl get Pods --watch -n calico-system
```

接下来，设置几个命名空间、一个用于提供测试网页的 NGINX 服务器和一个可以运行`wget`的 BusyBox 容器。为此，请使用以下命令：

```
$ kubectl create ns web
$ kubectl create ns test-bed
$ kubectl create deployment -n web nginx --image=nginx
$ kubectl expose -n web deployment nginx --port=80
$ kubectl run --namespace=test-bed testing --rm -ti --image busybox /bin/sh
```

从 BusyBox 容器的命令提示符访问在 `web` 命名空间中安装的 NGINX 服务器。以下是此命令：

```
$ wget -q nginx.web -O
```

现在，安装一个拒绝所有进入 NGINX Pod 的流量的网络策略。使用以下命令：

```
$ kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-ingress
  namespace: web
spec:
  PodSelector:
    matchLabels: {}
  policyTypes:
  - Ingress
EOF
```

此命令创建了一个策略，使得您无法从测试 Pod 访问 NGINX 网页。在测试 Pod 的命令行上运行以下命令。命令将超时并失败：

```
$ wget -q --timeout=5 nginx.web.svc.cluster.local -O -
```

接下来，打开从 `test-bed` 命名空间到 `web` 命名空间的 Pod 入口。使用以下代码片段：

```
$ kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: access-web
  namespace: web
spec:
  PodSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Ingress
  ingress:
    - from:
      - namespaceSelector:
          matchLabels:
            name: test-bed
EOF
```

在测试 Pod 的命令行上输入

```
$ wget -q --timeout=5 nginx.web.svc.cluster.local -O -
```

您会注意到命令失败了。原因是网络策略匹配标签，而 `test-bed` 命名空间没有标签。以下命令添加了标签：

```
$ kubectl label namespaces test-bed name=test-bed
```

在测试 Pod 的命令行上，检查网络策略现在是否生效。以下是命令：

```
$ wget -q --timeout=5 nginx.web.svc.cluster.local -O -
```

对于所有防火墙配置的第一项建议是创建一个拒绝所有规则的规则。此策略拒绝命名空间内的所有流量。运行以下命令并禁用 `test-bed` 命名空间的所有入口和出口 Pod：

```
$ kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-all
  namespace: test-bed
spec:
  PodSelector:
    matchLabels: {}    ❶
 policyTypes:          ❷
 - Ingress
  - Egress
EOF
```

❶ 匹配命名空间中的所有 Pod

❷ 定义了两种策略类型：入口和出口

现在，实施此策略会产生一些有趣的副作用。不仅 Pod 无法与其他任何东西（除了它们自己的命名空间）通信，而且现在它们无法与 `kube-system` 命名空间中的 DNS 提供商通信。如果 Pod 不需要 DNS 功能，请不要启用它！让我们应用以下网络策略以启用 DNS 的出口：

```
$ kubectl label namespaces kube-system name=kube-system
$ kubectl create -f - <<EOF
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: dns-egress
  namespace: test-bed
spec:
  PodSelector:
    matchLabels: {}          ❶
 policyTypes:                ❷
 - Egress
  egress:
  - to:
    - namespaceSelector:     ❸
       matchLabels:
          name: kube-system
    ports:                   ❹
   - protocol: UDP
      port: 53
EOF
```

❶ 匹配核心 Kubernetes 命名空间中的所有 Pod

❷ 只允许出口

❸ 匹配带有标签的 kube-system 的出口规则

❹ 只允许通过端口 53 的 UDP，这是 DNS 的协议和端口

如果您运行 `wget` 命令，您会注意到命令仍然失败。我们在 `web` 命名空间允许了入口，但没有在 `test-bed` 命名空间到 `web` 命名空间的出口上启用。运行以下命令以从 `test-bed` Pod 启用到 `web` 命名空间的出口：

```
$ kubectl label namespaces web name=web
$ kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-bed-egress
  namespace: test-bed
spec:
  PodSelector:
    matchLabels: {}
  policyTypes:
    - Egress
  egress:
    - to:
      - namespaceSelector:
          matchLabels:
            name: web
EOF
```

您可能已经注意到 NetworkPolicy 规则可能会变得复杂。如果您在一个信任模型为 *高信任* 的集群上运行，实施网络策略可能 *不会* 帮助您的安全态势。使用 80/20 规则，如果您的组织没有更新其镜像，请不要从 NetworkPolicies 开始。是的，网络策略很复杂，这也是为什么在您的组织中使用服务网格可能有助于安全的原因之一。

服务网格

*服务网格* 是在 Kubernetes 集群之上运行的应用程序，它提供了各种能力，通常可以改善可观察性、监控和可靠性。常见的服务网格包括 Istio、Linkerd、Consul 以及其他一些。我们在安全章节中提到服务网格，因为它们可以帮助您的组织完成两个关键任务：相互 TLS 和高级网络流量策略。我们对此主题的介绍非常简短，因为关于这个主题有整本书的篇幅。

服务网格在集群中运行的每个应用程序之上添加了一个复杂的层，但也提供了许多良好的安全组件。再次强调，您需要判断是否需要添加服务网格，但不要从第一天开始就使用服务网格。如果您想知道您的集群是否符合 CNCF 对 NetworkPolicy API 的规范，您可以使用 Sonobuoy（我们在前面的章节中已介绍）运行 NetworkPolicy 测试套件：

```
$ sonobuoy run --e2e-focus=NetworkPolicy
# wait about 30 minutes
$ sonobuoy status
```

这会输出一系列表格测试，显示您的集群上网络策略的确切工作方式。要了解更多关于 CNI 提供程序 NetworkPolicy API 兼容性概念的信息，请查看[`mng.bz/XW7M`](http://mng.bz/XW7M)。我们强烈建议在评估 CNI 提供程序与 Kubernetes 网络安全规范的兼容性时运行 NetworkPolicy 兼容性测试。

### 14.4.2 负载均衡器

需要注意的是，Kubernetes 可以创建外部负载均衡器，将您的应用程序暴露给世界，并且它是自动完成的。这看起来可能像常识，但将错误的服务放入生产环境可能会将服务（如管理用户界面）暴露给数据库。在 CI（持续集成）期间使用工具或像 Open Policy Agent（OPA）这样的工具来确保不会意外创建外部负载均衡器。此外，当可能时，请使用内部负载均衡器。

### 14.4.3 开放策略代理（OPA）

我们之前提到操作员可以帮助组织进一步保护集群。OPA，一个 CNCF 项目，致力于允许通过准入控制器运行的声明性策略。

OPA 是一个轻量级的通用策略引擎，可以与您的服务一起部署。您可以将 OPA 集成为一个 sidecar、主机级守护进程或库。

服务通过执行查询将策略决策卸载给 OPA。OPA 评估策略和数据以生成查询结果（这些结果会发送回客户端）。策略是用高级声明性语言编写的，可以通过文件系统或定义良好的 API 加载到 OPA 中。

—Open Policy Agent ([`mng.bz/RE6O`](http://mng.bz/RE6O))

OPA 维护了两个不同的组件：OPA 准入控制器和 OPA Gatekeeper。Gatekeeper 不使用 sidecar，使用 CRDs（自定义资源定义），可扩展，并执行审计功能。下一节将介绍如何在`kind` Kubernetes 集群上安装 Gatekeeper。

安装 OPA

首先，清理运行 Calico 的集群。然后，让我们启动另一个集群：

```
$ kind delete cluster
$ kind create cluster
```

接下来，使用以下命令安装 OPA Gatekeeper：

```
$ kubectl apply -f \
https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.7.0/
➥ deploy/gatekeeper.yaml
```

以下命令打印已安装 Pod 的名称：

```
$ kubectl -n gatekeeper-system get po
NAME                                           READY  STATUS   RESTARTS  AGE
gatekeeper-audit-7d99d9d87d-rb4qh              1/1    Running  0         40s
gatekeeper-controller-manager-f94cc7dfc-j6zjv  1/1    Running  0         39s
gatekeeper-controller-manager-f94cc7dfc-mxz6d  1/1    Running  0         39s
gatekeeper-controller-manager-f94cc7dfc-rvqvj  1/1    Running  0         39s
```

注意：您也可以使用 Helm 安装 OPA Gatekeeper。

Gatekeeper CRDs

OPA 的一个复杂性是学习一种新的语言（称为 Rego）来编写策略。有关 Rego 的更多信息，请参阅 [`mng.bz/2jdm`](http://mng.bz/2jdm)。使用 Gatekeeper，你将把用 Rego 编写的策略放入支持的 CRD 中。你需要创建两个不同的 CRD 来添加策略：

+   一个约束模板用于定义策略及其目标

+   一个约束用于启用约束模板并定义如何启用策略

以下是一个约束模板及其相关约束的示例。源块包含在 YAML 中定义的两个 CRD。在这个例子中，`match` 段支持

+   `kinds`—定义 Kubernetes API 对象

+   `namespaces`—指定命名空间列表

+   `excludedNamespaces`—指定要排除的命名空间列表

+   `scope`— *, 集群或命名空间

+   `labelSelector`—设置标准的 Kubernetes 标签选择器

+   `namespaceSelector`—设置标准的 Kubernetes 命名空间选择器

```
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: enforcespecificcontainerregistry
spec:
  crd:
    spec:
      names:
        kind: EnforceSpecificContainerRegistry        ❶
       # Schema for the `parameters` field
        openAPIV3Schema:
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |                                         ❶
       package enforcespecificcontainerregistry

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          satisfied := [good | repo = input.parameters.repos[_] ;
➥ good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("container ‘%v' has an invalid image repo
➥ ‘%v', allowed repos are %v",
➥ [container.name, container.image, input.parameters.repos])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          satisfied := [good | repo = input.parameters.repos[_] ;
➥ good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("container ‘%v' has an invalid image repo ‘%v',
➥ allowed repos are %v",
➥ [container.name, container.image, input.parameters.repos])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: EnforceSpecificContainerRegistry
metadata:
  name: enforcespecificcontainerregistrytestns
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "test-ns"
  parameters:
    repos:
      - "myrepo.com"
```

❶ 定义了 EnforceSpecific-ContainerRegistry，这是一个用于约束的 CRD

现在，让我们将之前的 YAML 文件保存为两个文件（一个包含模板，另一个包含约束）。在集群上，首先安装模板文件，然后安装约束文件。（为了简洁，我们不提供命令。）现在我们可以通过运行以下命令来测试策略：

```
$ kubectl create ns test-ns
$ kubectl create deployment -n test-ns nginx --image=nginx
```

你可以通过运行以下命令来检查部署的状态（我们预计 Pod 不会启动）：

```
$ kubectl get -n test-ns deployments.apps
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   0/1     0            0           37s
```

如果你执行 `kubectl -n test-ns get Pods`，你会注意到没有 Pod 在运行。事件日志中包含显示 Pod 创建失败的消息。你可以使用以下命令查看日志：

```
$ kubectl -n test-ns get events
7s          Warning   FailedCreate        replicaset/nginx-6799fc88d8
➥ Error creating: admission webhook "validation.gatekeeper.sh"
➥ denied the request: [denied by
➥ enforcespecificcontainerregistrytestns] container <nginx>
➥ has an invalid image repo <nginx>, allowed repos are
➥ ["myrepo.com"]
```

### 14.4.4 多租户

要对多租户进行分类，查看租户之间的信任水平，然后开发该模型。有三个基本类别或安全模型用于将多租户分类：

+   *高信任（同一家公司）*—同一家公司的不同部门在同一个集群上运行工作负载。

+   *中低信任（不同公司）*—外部客户在不同的命名空间中运行应用程序。

+   *零信任（受法律管辖的数据）*—不同的应用程序运行受法律管辖的数据，允许不同数据存储之间的访问可能导致公司面临法律诉讼。

Kubernetes 社区已经为解决这些用例工作了多年。Jessie Frazelle 在她的博客文章“Kubernetes 中的硬多租户”中很好地总结了这些内容：

多租户模型在社区的多租户工作组中已经进行了广泛的讨论。……还提出了一些提案来解决每个模型。Kubernetes 中当前的租户模型假设集群是安全边界。你可以在 Kubernetes 之上构建 SaaS，但你将需要带来自己的可信 API，而不仅仅是使用 Kubernetes API。当然，这伴随着在为 SaaS 安全构建集群时你必须考虑的许多考虑因素。

——杰西·弗拉泽尔，[`mng.bz/1jdn`](http://mng.bz/1jdn)

Kubernetes API 并没有考虑到在同一个集群内存在多个隔离的客户的概念。Docker Engine 和其他容器运行时在运行恶意或不信任的工作负载时也存在问题。像 gVisor 这样的软件组件在正确沙箱化容器方面取得了进展，但在撰写本书时，我们还没有达到可以运行完全不受信任的容器的地步。

那么，我们现在在哪里？安全人员会说这取决于你的信任和安全模型。我们之前列出了三种安全模型：高度信任（同一公司）、低信任到无信任（不同公司）和零信任（数据受法律管辖）。Kubernetes 可以支持高度信任的多租户，并且根据模型，可以在高度信任和低信任模型之间支持信任模型。当你与租户之间或租户之间有零信任或低信任时，你需要为每个客户端使用单独的集群。一些公司运行数百个集群，以便每个小型应用程序组都能获得自己的集群，但诚然，这需要管理很多集群。

即使客户属于同一公司，由于数据敏感性，可能还需要在特定节点上隔离 Pods。通过 RBAC、命名空间、网络策略和节点隔离，可以获得相当程度的隔离。诚然，存在不同公司运行的工作负载托管在同一个 Kubernetes 集群中的风险。多租户的支持将随着时间的推移而增长。

注意：多租户也适用于在生产集群中运行其他环境，如开发或测试。然而，通过混合环境，你可能会将不良行为者的代码引入集群中。

使用单个 Kubernetes 来托管多个客户主要面临两个挑战：API 服务器和节点安全。在建立身份验证、授权和 RBAC 之后，为什么 API 服务器和多个租户之间会出现问题？其中一个问题是 API 服务器 URI 布局的问题。对于多个租户使用相同 API 服务器的常见模式是让用户 ID、项目 ID 或某些唯一 ID 作为 URI 的开头。

具有以唯一 ID 开始的 URI 允许租户调用以获取*所有*命名空间。因为 Kubernetes 没有这种隔离，你需要运行`kubectl get namespaces`来获取集群中的所有命名空间。你还需要在 Kubernetes API 之上运行一个 API 层，以提供这种形式的隔离。

允许多租户的另一种模式是资源嵌套的能力，以及 Kubernetes 中的基本资源边界是命名空间。Kubernetes 命名空间不允许嵌套。许多资源是跨命名空间的，包括默认的服务账户令牌。通常，租户希望拥有细粒度的 RBAC 能力，而授予用户在 Kubernetes 中创建 RBAC 对象权限可能会赋予用户超出其共享租户的能力。

关于节点安全，挑战在于内部。如果你在同一 Kubernetes 集群上有多个租户，请记住他们共享以下项目（这只是一个简短的列表）：

+   控制平面和 API 服务器

+   添加 DNS 服务器、日志记录或 TLS 证书生成等附加组件

+   自定义资源定义（CRDs）

+   网络

+   主机资源

信任多租户概述

许多公司希望多租户可以降低成本和管理开销。不运行三个集群：一个用于开发、一个用于测试、一个用于生产环境是有价值的。只需运行一个 Kubernetes 集群用于所有三个环境。此外，一些公司不希望为不同的产品或软件部门拥有单独的集群。这同样是一个业务和安全决策，我们合作的组织通常有预算和人力资源的限制。

我们不会给你提供如何进行多租户的逐步指导。我们只是提供你需要实施的步骤的指南。这些步骤会随着时间的推移而变化，并且在不同安全模型的组织之间会有所不同：

1.  记录并设计一个安全模型。这可能看起来很明显，但我们看到一些组织没有使用安全模型。安全模型需要包括不同的用户角色，包括集群管理员和命名空间管理员，以及一个或多个租户角色。为组织创建的所有 API 对象、用户和其他组件的标准命名约定也是至关重要的。

1.  利用各种 API 对象：

    +   命名空间

    +   网络策略

    +   ResourceQuotas

    +   服务账户和 RBAC 规则

1.  使用具有相互 TLS 和网络策略管理等功能的服务网格可以提供另一层安全性。使用服务网格确实会增加显著的复杂性，因此只有在需要时才使用它。

1.  考虑使用 OPA 来帮助将基于策略的控制应用于 Kubernetes 集群。

提示：如果你打算在单个集群中结合多个环境，不仅存在安全问题，还有测试 Kubernetes 升级的挑战。最好先在另一个集群上测试升级。

## 14.5 Kubernetes 技巧

这里是一份简短的各种配置和设置要求的列表：

+   拥有私有 API 服务器端点，并且如果可能的话，不要将你的 API 服务器暴露给互联网。

+   使用 RBAC。

+   使用网络策略。

+   不要在 API 服务器上启用用户名和密码授权。

+   创建 Pod 时使用特定用户，不要使用默认管理员账户。

+   很少允许 Pod 在主机网络上运行。

+   如果 Pod 需要访问 API 服务器，请使用`serviceAccountName`；否则，将`automountServiceAccountToken`设置为 false。

+   在命名空间上使用资源配额，并在所有 Pod 中定义限制。

## 摘要

+   节点安全性依赖于 TLS 证书来保护节点和控制平面之间的通信。

+   使用不可变操作系统可以进一步增强节点安全性。

+   资源限制可以防止资源级别的攻击。

+   使用 Pod 网络，除非你不得不使用主机网络。主机网络允许 Pod 与节点操作系统通信。

+   RBAC 是保护 API 服务器的关键。虽然非同寻常，但这是必要的。

+   IAM 服务账户允许正确隔离 Pod 权限。

+   网络策略是隔离网络流量的关键。否则，一切都可以与一切通信。

+   Open Policy Agent (OPA) 允许用户编写安全策略，并在 Kubernetes 集群上强制执行这些策略。

+   Kubernetes 最初并非以零信任多租户模式构建。你可以找到多租户的形式，但它们伴随着权衡。
