# 第三章：容器运行时

Kubernetes 是一个容器编排器。然而，Kubernetes 本身并不知道如何创建、启动和停止容器。相反，它将这些操作委托给一个可插拔的组件，称为*容器运行时*。容器运行时是一种软件，用于在集群节点上创建和管理容器。在 Linux 中，容器运行时使用一组内核原语，如控制组（cgroups）和命名空间，从容器镜像生成一个进程。实质上，Kubernetes，尤其是 kubelet，与容器运行时共同工作来运行容器。

正如我们在第一章中讨论的那样，构建基于 Kubernetes 平台的组织面临多种选择。选择使用哪种容器运行时就是其中之一。选择是很好的，因为它让您可以根据自己的需求定制平台，从而促进创新和可能原本无法实现的高级用例。然而，考虑到容器运行时的基本性质，为什么 Kubernetes 不提供一个实现？为什么选择提供一个可插拔的接口，并将责任转移给另一个组件？

要回答这些问题，我们将回顾一下容器的历史以及我们如何到达现在的地步。我们将首先讨论容器的出现以及它们如何改变软件开发的格局。毕竟，没有它们，可能 Kubernetes 并不存在。然后，我们将讨论开放容器倡议（OCI），这是因为需要在容器运行时、镜像和其他工具周围进行标准化而产生的。我们将回顾 OCI 的规范及其与 Kubernetes 的关系。在 OCI 之后，我们将讨论专用于 Kubernetes 的容器运行时接口（CRI）。CRI 是 kubelet 与容器运行时之间的桥梁。它规定了容器运行时必须实现的接口，以便与 Kubernetes 兼容。最后，我们将讨论如何为您的平台选择运行时，并回顾 Kubernetes 生态系统中的可用选项。

# 容器的出现

控制组（cgroups）和命名空间是实现容器所需的主要组成部分。Cgroups 对进程可以使用的资源量（如 CPU、内存等）施加限制，而命名空间控制进程可以看到的内容（如挂载点、进程、网络接口等）。这两个基本原语自 2008 年以来就已经存在于 Linux 内核中。在命名空间的情况下，更早些时候已经存在。那么，为什么像今天这样的容器才会在多年后变得流行起来呢？

要回答这个问题，我们首先需要考虑软件和 IT 行业当时周围的环境。首先要考虑的一个因素是应用程序的复杂性。应用程序开发人员使用面向服务的架构构建应用程序，甚至开始接受微服务。这些架构为组织带来了各种好处，如可维护性、可扩展性和生产力。然而，它们也导致应用程序组成部分数量的激增。有意义的应用程序可能涉及数十个服务，可能使用多种语言编写。可以想象，开发和发布这些应用程序是（并且继续是）复杂的。另一个要记住的因素是软件迅速成为企业的差异化因素。您能越快推出新功能，您的竞争力就越强。以可靠方式部署软件对企业至关重要。最后，公共云作为托管环境的出现是另一个重要因素。开发人员和运维团队必须确保应用程序在所有环境中表现一致，从开发者的笔记本电脑到运行在他人数据中心的生产服务器。

记住这些挑战，我们可以看到环境已经成熟，可以进行创新。进入 Docker。Docker 使容器变得普及。他们构建了一个抽象层，使开发人员可以通过易于使用的 CLI 构建和运行容器。开发人员无需了解利用容器技术所需的低级内核构造，他们只需在终端中输入 `docker run`。

虽然并非解决我们所有问题的答案，但容器改善了软件开发生命周期的许多阶段。首先，容器和容器镜像允许开发人员编码应用程序的环境。开发人员不再需要处理缺失或不匹配的应用程序依赖关系。其次，容器通过为测试应用程序提供可复制的环境影响了测试。最后，容器使得将软件部署到生产环境变得更加容易。只要在生产环境中有一个 Docker 引擎，应用程序就可以以最小的摩擦部署。总体而言，容器帮助组织以更加可重复和高效的方式从零到生产部署软件。

容器的出现也催生了一个充满不同工具、容器运行时、容器镜像注册表等多样化生态系统。这个生态系统受到了良好的接受，但也带来了一个新挑战：如何确保所有这些容器解决方案彼此兼容？毕竟，封装性和可移植性保证是容器的主要好处之一。为了解决这一挑战并促进容器的采用，业界汇聚在 Linux 基金会的旗帜下共同合作，推出了一个开放源代码规范：Open Container Initiative。

# 开放容器倡议

随着容器在行业中持续流行，清楚地需要制定标准和规范以确保容器运动的成功。Open Container Initiative（OCI）是一个于 2015 年成立的开源项目，旨在协作制定关于容器的规范。这一倡议的重要创始人包括 Docker（将 runc 捐赠给 OCI）和 CoreOS（通过 rkt 推动容器运行时的发展）。

OCI 包括三个规范：OCI 运行时规范、OCI 镜像规范和 OCI 分发规范。这些规范促进了围绕容器和容器平台（如 Kubernetes）的开发和创新。此外，OCI 的目标是允许最终用户以便捷和互操作的方式使用容器，使他们在必要时更轻松地在产品和解决方案之间进行迁移。

在接下来的章节中，我们将探讨运行时和镜像规范。我们不会深入讨论分发规范，因为它主要涉及容器镜像注册表。

## OCI Runtime 规范

OCI 运行时规范决定如何以兼容 OCI 的方式实例化和运行容器。首先，规范描述了容器配置的模式。模式包括容器的根文件系统、运行命令、环境变量、要使用的用户和用户组、资源限制等信息。以下摘录是从 OCI 运行时规范获取的容器配置文件的简化示例：

```
{
    "ociVersion": "1.0.1",
    "process": {
        "terminal": true,
        "user": {
            "uid": 1,
            "gid": 1,
            "additionalGids": [
                5,
                6
            ]
        },
        "args": [
            "sh"
        ],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "TERM=xterm"
        ],
        "cwd": "/",
        ...
    },
    ...
    "mounts": 
        {
            "destination": "/proc",
            "type": "proc",
            "source": "proc"
        },
    ...
    },
    ...
}
```

运行时规范还确定了容器运行时必须支持的操作。这些操作包括创建、启动、终止、删除和状态（提供容器状态信息）。除了操作外，运行时规范描述了容器的生命周期及其在不同阶段的进展。生命周期阶段包括（1）`creating`，即容器运行时正在创建容器时的活动状态；（2）`created`，即运行时已完成`create`操作时的状态；（3）`running`，即容器进程已启动并正在运行时的状态；以及（4）`stopped`，即容器进程已完成运行时的状态。

OCI 项目还包括 *runc*，这是一个实现 OCI 运行时规范的低级容器运行时。其他高级容器运行时如 Docker、containerd 和 CRI-O 使用 runc 根据 OCI 规范生成容器，如 [图 3-1 所示。利用 runc，容器运行时可以专注于诸如拉取镜像、配置网络、处理存储等高级功能，同时遵循 OCI 运行时规范。

![prku 0301](img/prku_0301.png)

###### 图 3-1\. Docker Engine、containerd 和其他运行时根据 OCI 规范使用 runc 生成容器。

## OCI 镜像规范

OCI 镜像规范关注容器镜像。规范定义了一个清单、一个可选的镜像索引、一组文件系统层和一个配置。镜像清单描述了镜像。它包括指向镜像配置、镜像层列表和可选的注释映射的指针。以下是从 OCI 镜像规范获取的示例清单：

```
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f4..."
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 32654,
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d0..."
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 16724,
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15..."
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 73109,
      "digest": "sha256:ec4b8955958665577945c89419d1af06b5f7636b4ac3da7f12184..."
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

*镜像索引* 是一个顶级清单，用于创建多平台容器镜像。镜像索引包含指向每个平台特定清单的指针。以下是从规范中获取的示例索引。请注意索引如何指向两个不同的清单，一个是 `ppc64le/linux`，另一个是 `amd64/linux`：

```
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 7143,
      "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab...",
      "platform": {
        "architecture": "ppc64le",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 7682,
      "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e9...",
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

每个 OCI 镜像清单都引用一个容器镜像配置。配置包括镜像的入口点、命令、工作目录、环境变量等。容器运行时在实例化镜像时使用此配置。以下摘录显示了容器镜像配置的部分内容，为了简洁起见省略了一些字段：

```
{
  "architecture": "amd64",
  "config": {
    ...
    "ExposedPorts": {
      "53/tcp": {},
      "53/udp": {}
    },
    "Tty": false,
    "OpenStdin": false,
    "StdinOnce": false,
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": null,
    "Image": "sha256:7ccecf40b555e5ef2d8d3514257b69c2f4018c767e7a20dbaf4733...",
    "Volumes": null,
    "WorkingDir": "",
    "Entrypoint": [
      "/coredns"
    ],
    "OnBuild": null,
    "Labels": null
  },
  "created": "2020-01-28T19:16:47.907002703Z",
  ...
```

OCI 镜像规范还描述了如何创建和管理容器镜像层。层实质上是包含文件和目录的 TAR 存档。规范定义了不同的层媒体类型，包括未压缩层、gzip 压缩层和不可分发层。每个层都通过摘要唯一标识，通常是层内容的 SHA256 摘要。如前所述，容器镜像清单引用一个或多个层。这些引用使用 SHA256 摘要指向特定的层。最终容器镜像文件系统是应用每个层后的结果，如清单中所列。

OCI 镜像规范至关重要，因为它确保容器镜像在不同工具和基于容器的平台上是可移植的。该规范支持开发不同的镜像构建工具，如 kaniko 和 Buildah 用于用户空间容器构建，Jib 用于基于 Java 的容器，以及 Cloud Native Buildpacks 用于简化和自动化构建。我们将在 第十五章 中探讨其中一些工具。总体而言，该规范确保 Kubernetes 可以运行不管使用何种构建工具生成的容器镜像。

# 容器运行时接口

正如我们在之前的章节中讨论过的，Kubernetes 提供了许多扩展点，允许您构建一个定制的应用平台。其中一个最关键的扩展点是容器运行时接口 (CRI)。CRI 在 Kubernetes v1.5 中引入，旨在支持包括 CoreOS 的 rkt 和基于虚拟化的运行时，如 Intel 的 Clear Containers（后来成为 Kata Containers）在内的不断增长的容器运行时生态系统。

在 CRI 出现之前，为新的容器运行时添加支持需要发布新版本的 Kubernetes 并对 Kubernetes 代码库有深入了解。一旦建立了 CRI，容器运行时开发人员只需遵循接口即可确保与 Kubernetes 的兼容性。

总体而言，CRI 的目标是将容器运行时的实现细节与 Kubernetes 分离，更具体地说是 kubelet。这是依赖反转原则的一个典型示例。kubelet 从具有分散在各处的容器运行时特定代码和 if 语句的实现逐步演变为依赖接口的更精简实现。因此，CRI 减少了 kubelet 实现的复杂性，同时使其更具可扩展性和可测试性。这些都是设计良好的软件的重要特性。

CRI 使用 gRPC 和 Protocol Buffers 实现。该接口定义了两个服务：*RuntimeService* 和 *ImageService*。kubelet 利用这些服务与容器运行时交互。RuntimeService 负责所有与 Pod 相关的操作，包括创建 Pod、启动和停止容器、删除 Pod 等。ImageService 则涉及容器镜像操作，包括在节点上列出、拉取和删除容器镜像等。

虽然我们可以在本章节详细描述 RuntimeService 和 ImageService 的 API，但更有用的是理解 Kubernetes 中可能最重要操作的流程：在节点上启动一个 Pod。因此，让我们在接下来的部分探索 kubelet 与容器运行时通过 CRI 的交互。

## 启动一个 Pod

###### 注：

以下描述基于 Kubernetes v1.18.2 和 containerd v1.3.4。这些组件使用 CRI 的 v1alpha2 版本。

一旦 Pod 被调度到一个节点上，kubelet 会与容器运行时一起工作，启动 Pod。正如前文所述，kubelet 通过 CRI 与容器运行时进行交互。在这种情况下，我们将探讨 kubelet 与 containerd CRI 插件之间的交互。

containerd CRI 插件启动一个 gRPC 服务器，监听一个 Unix 套接字。默认情况下，此套接字位于 */run/containerd/containerd.sock*。kubelet 配置为通过此套接字与 containerd 进行交互，使用 `container-runtime` 和 `container-runtime-endpoint` 命令行标志：

```
/usr/bin/kubelet
  --container-runtime=remote
  --container-runtime-endpoint=/run/containerd/containerd.sock
  ... other flags here ...
```

要启动 Pod，kubelet 首先使用 RuntimeService 的 `RunPodSandbox` 方法创建一个 Pod 沙箱。因为一个 Pod 由一个或多个容器组成，所以必须首先创建沙箱，以建立 Linux 网络命名空间（等等）供所有容器共享。调用此方法时，kubelet 向 containerd 发送元数据和配置，包括 Pod 的名称、唯一 ID、Kubernetes 命名空间、DNS 配置等。一旦容器运行时创建了沙箱，运行时会返回一个 Pod 沙箱 ID，kubelet 使用该 ID 在沙箱中创建容器。

一旦沙箱可用，kubelet 通过 ImageService 的 `ImageStatus` 方法检查节点上是否存在容器镜像。`ImageStatus` 方法返回关于镜像的信息。当镜像不存在时，该方法返回 null，并且 kubelet 继续拉取镜像。当需要时，kubelet 使用 ImageService 的 `PullImage` 方法拉取镜像。一旦运行时拉取了镜像，它会返回镜像的 SHA256 摘要，kubelet 然后使用该摘要创建容器。

在创建沙箱和拉取镜像后，kubelet 使用 RuntimeService 的 `CreateContainer` 方法在沙箱中创建容器。kubelet 向容器运行时提供沙箱 ID 和容器配置。容器配置包括您可能期望的所有信息，包括容器镜像摘要、命令和参数、环境变量、卷挂载等。在创建过程中，容器运行时生成一个容器 ID，并将其传递回 kubelet。这个 ID 是您在 Pod 的状态字段下容器状态中看到的那个：

```
containerStatuses:
  - containerID: containerd://0018556b01e1662c5e7e2dcddb2bb09d0edff6cf6933...
    image: docker.io/library/nginx:latest
```

然后，kubelet 继续使用 RuntimeService 的 `StartContainer` 方法启动容器。在调用此方法时，它使用容器运行时传递的容器 ID。

就是这样！在本节中，我们学习了 kubelet 如何使用 CRI 与容器运行时交互。我们特别关注了启动 Pod 时调用的 gRPC 方法，其中包括 ImageService 和 RuntimeService 上的方法。这两个 CRI 服务除了 Pod 和容器管理（即 CRUD）方法外，还提供了 kubelet 用于完成其他任务的其他方法。除了执行容器内命令（`Exec` 和 `ExecSync`）、附加到容器（`Attach`）、转发特定容器端口（`PortForward`）等任务外，CRI 还定义了其他方法。

# 选择运行时

考虑到 CRI 的可用性，平台团队在选择容器运行时时有了灵活性。然而，事实上，在过去几年中，容器运行时已经成为一个实施细节。如果您使用 Kubernetes 发行版或利用托管的 Kubernetes 服务，则容器运行时很可能会为您选择。即使是像 Cluster API 这样的社区项目也是如此，它提供预先构建的包含容器运行时的节点镜像。

话虽如此，如果您确实有选择运行时的选项或者有专用运行时的用例（例如基于 VM 的运行时），您应该具备足够的信息来做出决策。在本节中，我们将讨论选择容器运行时时应考虑的因素。

在帮助现场组织时，我们通常会问的第一个问题是他们对容器运行时有何经验。在大多数情况下，有长期容器历史的组织使用 Docker，并熟悉 Docker 的工具链和用户体验。虽然 Kubernetes 支持 Docker，但我们不鼓励使用它，因为它具有一些 Kubernetes 不需要的扩展功能，例如构建镜像、创建容器网络等。换句话说，完整的 Docker 守护程序对于 Kubernetes 来说过于臃肿。好消息是 Docker 在幕后使用 containerd，这是社区中最普遍的容器运行时之一。不利之处是平台操作员必须学习 containerd 的 CLI。

另一个要考虑的因素是支持的可用性。根据您获取 Kubernetes 的位置，您可能会获得容器运行时的支持。诸如 VMware 的 Tanzu Kubernetes Grid、RedHat 的 OpenShift 等 Kubernetes 发行版通常会预装特定的容器运行时。除非您有极其强烈的理由选择其他选项，否则应坚持这种选择。在这种情况下，请确保您理解使用不同容器运行时的支持影响。

与支持密切相关的是容器运行时的一致性测试。Kubernetes 项目，特别是节点特别兴趣小组（sig-node），定义了一组 CRI 验证测试和节点一致性测试，以确保容器运行时兼容并如预期般行为。这些测试是每个 Kubernetes 发布的一部分，某些运行时可能比其他运行时有更多的覆盖。可以想象，测试覆盖越多越好，因为运行时的任何问题都会在 Kubernetes 发布过程中被捕捉到。社区通过 [Kubernetes 测试网格](https://k8s-testgrid.appspot.com) 提供所有测试和结果。在选择运行时时，应考虑容器运行时的一致性测试以及其与整体 Kubernetes 项目的关系。

最后，你应确定你的工作负载是否需要比 Linux 容器提供的更强的隔离保证。虽然不常见，但有些用例需要工作负载的 VM 级别隔离，比如执行不受信任的代码或运行需要强大的多租户保证的应用程序。在这些情况下，你可以利用专用运行时，比如 Kata Containers。

现在我们已经讨论了在选择运行时时应考虑的因素，让我们回顾一下最常见的容器运行时：Docker、containerd 和 CRI-O。我们还将探讨 Kata Containers，以了解如何在虚拟机中运行 Pod，而不是 Linux 容器。最后，虽然 Virtual Kubelet 不是容器运行时或实现 CRI 的组件，但我们将了解它，因为它提供了在 Kubernetes 上运行工作负载的另一种方式。

## Docker

Kubernetes 通过称为 *dockershim* 的 CRI 适配器支持 Docker Engine 作为容器运行时。这个适配器是内置到 kubelet 中的一个组件。本质上，它是一个实现我们在本章前面描述的 CRI 服务的 gRPC 服务器。dockershim 的存在是因为 Docker Engine 没有实现 CRI。为了不特别处理所有 kubelet 代码路径以与 CRI 和 Docker Engine 一起工作，dockershim 作为一个外观，kubelet 可以通过它与 Docker 通信，通过 CRI。dockershim 处理 CRI 调用到 Docker Engine API 调用的转换。图 3-2 描述了 kubelet 如何通过 shim 与 Docker 交互。

![prku 0302](img/prku_0302.png)

###### 图 3-2\. kubelet 与 Docker Engine 通过 dockershim 之间的交互。

正如我们在本章前面提到的，Docker 在底层利用 containerd。因此，kubelet 的传入 API 调用最终会被转发到 containerd，containerd 开始容器。最终，生成的容器结束在 containerd 而不是 Docker 守护程序下：

```
systemd
  └─containerd
      └─containerd-shim -namespace moby -workdir ...
          └─nginx
              └─nginx
```

从故障排除的角度来看，您可以使用 Docker CLI 列出并检查在给定节点上运行的容器。虽然 Docker 没有 Pod 的概念，但 dockershim 将 Kubernetes 命名空间、Pod 名称和 Pod ID 编码到容器的名称中。例如，以下列表显示属于`default`命名空间中名为`nginx`的 Pod 的容器。Pod 基础设施容器（即暂停容器）在名称中具有`k8s_POD_`前缀：

```
$ docker ps --format='{{.ID}}\t{{.Names}}' | grep nginx_default
3c8c01f47424	k8s_nginx_nginx_default_6470b3d3-87a3-499c-8562-d59ba27bced5_3
c34ad8d80c4d	k8s_POD_nginx_default_6470b3d3-87a3-499c-8562-d59ba27bced5_3
```

您还可以使用 containerd CLI `ctr`来检查容器，尽管其输出不如 Docker CLI 输出友好。Docker Engine 使用名为`moby`的 containerd 命名空间：

```
$ ctr --namespace moby containers list
CONTAINER                    IMAGE    RUNTIME
07ba23a409f31bec7f163a...    -        io.containerd.runtime.v1.linux
0bfc5a735c213b9b296dad...    -        io.containerd.runtime.v1.linux
2d1c9cb39c674f75caf595...    -        io.containerd.runtime.v1.linux
...
```

最后，如果节点上有`crictl`可用，您可以使用它。`crictl`实用程序是由 Kubernetes 社区开发的命令行工具。它是用于通过 CRI 与容器运行时进行交互的 CLI 客户端。尽管 Docker 不实现 CRI，您仍然可以使用`crictl`与 dockershim Unix 套接字一起使用：

```
$ crictl --runtime-endpoint unix:///var/run/dockershim.sock ps --name nginx
CONTAINER ID   IMAGE                CREATED        STATE    NAME   POD ID
07ba23a409f31  nginx@sha256:b0a...  3 seconds ago  Running  nginx  ea179944...
```

## containerd

在构建基于 Kubernetes 的平台时，containerd 可能是我们在现场遇到的最常见的容器运行时。撰写本文时，containerd 是基于 Cluster API 的节点映像中的默认容器运行时，并且在各种托管的 Kubernetes 提供中可用（例如，AKS、EKS 和 GKE）。

containerd 容器运行时通过 containerd CRI 插件实现 CRI。CRI 插件是自 containerd v1.1 起可用的本地 containerd 插件，并且默认启用。containerd 通过 Unix 套接字`/run/containerd/containerd.sock`公开其 gRPC API。kubelet 在运行 Pod 时使用此套接字与 containerd 进行交互，如图 3-3 所示。

![prku 0303](img/prku_0303.png)

###### 图 3-3\. kubelet 与 containerd 通过 containerd CRI 插件之间的交互。

生成的容器的进程树与使用 Docker Engine 时的进程树完全相同。这是预期的，因为 Docker Engine 使用 containerd 来管理容器：

```
systemd
  └─containerd
      └─containerd-shim -namespace k8s.io -workdir ...
          └─nginx
              └─nginx
```

要检查节点上的容器，您可以使用 containerd CLI `ctr`。与 Docker 相反，由 Kubernetes 管理的容器位于名为`k8s.io`的 containerd 命名空间中，而不是`moby`：

```
$ ctr --namespace k8s.io containers ls | grep nginx
c85e47fa...    docker.io/library/nginx:latest    io.containerd.runtime.v1.linux
```

使用`crictl` CLI 与 containerd 通过 containerd 的 Unix 套接字进行交互：

```
$ crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
  --name nginx
CONTAINER ID    IMAGE          CREATED         STATE      NAME   POD ID
c85e47faf3616   4bb46517cac39  39 seconds ago  Running    nginx  73caea404b92a
```

## CRI-O

CRI-O 是专为 Kubernetes 设计的容器运行时。从名称中您可能能够看出，它是 CRI 的一个实现。因此，与 Docker 和 containerd 相比，它不适用于 Kubernetes 以外的用途。在撰写本文时，CRI-O 容器运行时的主要使用者之一是 RedHat OpenShift 平台。

与 containerd 类似，CRI-O 通过 Unix 套接字暴露 CRI。 kubelet 使用套接字与 CRI-O 进行交互，通常位于 */var/run/crio/crio.sock*。 图 3-4 展示了 kubelet 直接通过 CRI 与 CRI-O 交互的过程。

![prku 0304](img/prku_0304.png)

###### 图 3-4\. kubelet 与 CRI-O 使用 CRI API 的交互。

在生成容器时，CRI-O 实例化了一个名为 *conmon* 的进程。 Conmon 是容器监视器。 它是容器进程的父进程，并处理多个问题，例如提供连接到容器的方式，将容器的 STDOUT 和 STDERR 流存储到日志文件中，并处理容器的终止：

```
systemd
  └─conmon -s -c ed779... -n k8s_nginx_nginx_default_e9115... -u8cdf0c...
      └─nginx
          └─nginx
```

因为 CRI-O 被设计为 Kubernetes 的低级组件，所以 CRI-O 项目不提供 CLI。 话虽如此，您可以像对待其他实现 CRI 的容器运行时一样使用 `crictl` 与 CRI-O 进行交互：

```
$ crictl --runtime-endpoint unix:///var/run/crio/crio.sock ps --name nginx
CONTAINER   IMAGE                 CREATED         STATE     NAME    POD ID
8cdf0c...   nginx@sha256:179...   2 minutes ago   Running   nginx   eabf15237...
```

## Kata Containers

Kata Containers 是一个开源的专用运行时，使用轻量级虚拟机而不是容器来运行工作负载。 该项目源于两个先前基于 VM 的运行时的合并：Intel 的 Clear Containers 和 Hyper.sh 的 RunV。

由于使用了虚拟机，Kata 提供比 Linux 容器更强的隔离保证。 如果您有安全要求，禁止工作负载共享 Linux 内核，或者有 cgroup 隔离无法满足的资源保证要求，那么 Kata Containers 可能是一个很好的选择。 例如，Kata 容器的常见用例是运行运行不受信任代码的多租户 Kubernetes 集群。 云提供商如 [百度云](https://oreil.ly/btDL9) 和 [华为云](https://oreil.ly/Mzarh) 在其云基础设施中使用 Kata Containers。

要在 Kubernetes 中使用 Kata Containers，仍然需要一个可插拔的容器运行时放置在 kubelet 和 Kata 运行时之间，如 图 3-5 所示。 原因在于 Kata Containers 没有实现 CRI。 取而代之，它利用现有的容器运行时（如 containerd）来处理与 Kubernetes 的交互。 为了与 containerd 集成，Kata Containers 项目实现了 containerd 运行时 API，特别是 [v2 containerd-shim API](https://oreil.ly/DxGyZ)。

![prku 0305](img/prku_0305.png)

###### 图 3-5\. kubelet 与 Kata Containers 之间通过 containerd 的交互。

因为节点上需要并且可用 containerd，因此可以在同一节点上运行 Linux 容器 Pod 和基于 VM 的 Pod。 Kubernetes 提供了一种称为 Runtime Class 的机制，用于配置和运行多个容器运行时。 使用 RuntimeClass API，可以在同一 Kubernetes 平台上提供不同的运行时，使开发人员可以选择更适合其需求的运行时。 下面的片段是 Kata Containers 运行时的示例 RuntimeClass：

```
apiVersion: node.k8s.io/v1beta1
kind: RuntimeClass
metadata:
    name: kata-containers
handler: kata
```

要在`kata-containers`运行时下运行一个 Pod，开发者必须在他们 Pod 的规格中指定运行时类名：

```
apiVersion: v1
kind: Pod
metadata:
  name: kata-example
spec:
  containers:
  - image: nginx
    name: nginx
  runtimeClassName: kata-containers
```

Kata Containers 支持不同的虚拟化程序来运行工作负载，包括 [QEMU](https://www.qemu.org)，[NEMU](https://github.com/intel/nemu)，和 [AWS Firecracker](https://firecracker-microvm.github.io)。例如，当使用 QEMU 时，我们可以在使用 `kata-containers` 运行时类的 Pod 启动后看到一个 QEMU 进程：

```
$ ps -ef | grep qemu
root     38290     1  0 16:02 ?        00:00:17
   /snap/kata-containers/690/usr/bin/qemu-system-x86_64
   -name sandbox-c136a9addde4f26457901ccef9de49f02556cc8c5135b091f6d36cfc97...
   -uuid aaae32b3-9916-4d13-b385-dd8390d0daf4
   -machine pc,accel=kvm,kernel_irqchip
   -cpu host
   -m 2048M,slots=10,maxmem=65005M
   ...
```

虽然 Kata Containers 提供了一些有趣的功能，但我们认为它是一个小众产品，并且没有看到它在实际场景中的应用。话虽如此，如果你在 Kubernetes 集群中需要 VM 级别的隔离保证，那么 Kata Containers 值得一试。

## 虚拟 Kubelet

[虚拟 Kubelet](https://github.com/virtual-kubelet/virtual-kubelet) 是一个开源项目，行为类似于 kubelet，但在后端提供可插拔 API。虽然它本身不是一个容器运行时，但其主要目的是公开替代运行时以运行 Kubernetes Pods。由于虚拟 Kubelet 的可扩展架构，这些替代运行时本质上可以是任何能运行应用程序的系统，如无服务器框架、边缘框架等。例如，如图 3-6 所示，虚拟 Kubelet 可以在 Azure 容器实例或 AWS Fargate 等云服务上启动 Pods。

![prku 0306](img/prku_0306.png)

###### 图 3-6\. 虚拟 Kubelet 在云服务（如 Azure 容器实例、AWS Fargate 等）上运行 Pods。

虚拟 Kubelet 社区提供了多种提供者，如果符合你的需求，你可以利用它们，包括 AWS Fargate、Azure 容器实例、HashiCorp Nomad 等。如果你有更具体的用例，你也可以实现自己的提供者。实现一个提供者涉及使用虚拟 Kubelet 库编写一个 Go 程序，用于处理与 Kubernetes 的集成，包括节点注册、运行 Pods 和导出 Kubernetes 期望的 API。

虽然虚拟 Kubelet 可以启用有趣的场景，但我们还没有在实际场景中遇到需要它的用例。话虽如此，了解其存在是有益的，你应该将其纳入你的 Kubernetes 工具箱。

# 摘要

容器运行时是基于 Kubernetes 平台的基础组件。毕竟，在没有容器运行时，是不可能运行容器化工作负载的。正如我们在本章学到的那样，Kubernetes 使用容器运行时接口（CRI）与容器运行时进行交互。CRI 的主要优点之一是其可插拔性，这使您可以选择最适合您需求的容器运行时。为了让您了解生态系统中的不同容器运行时选项，我们讨论了一些在实际场景中常见的选项，如 Docker、containerd 等。了解这些不同的选择并进一步探索它们的能力，应该有助于您选择满足应用平台需求的容器运行时。
