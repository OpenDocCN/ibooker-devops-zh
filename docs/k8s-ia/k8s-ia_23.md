附录 C. 使用其他容器运行时

C.1\. 用 rkt 替换 Docker

我们在这本书中多次提到了 rkt（发音为 rock-it）。与 Docker 一样，它使用与 Docker 相同的 Linux 技术在隔离的容器中运行应用程序。让我们看看 rkt 与 Docker 的区别以及如何在 Minikube 中尝试它。

rkt 的第一个优点是它直接支持 Pod 的概念（运行多个相关容器），这与仅运行单个容器的 Docker 不同。rkt 基于开放标准，并且从一开始就考虑了安全性（例如，镜像已签名，因此您可以确信它们没有被篡改）。与最初基于客户端-服务器架构且与 systemd 等初始化系统不兼容的 Docker 不同，rkt 是一个直接运行容器的 CLI 工具，而不是告诉守护进程运行它。rkt 的一个好处是它可以运行现有的 Docker 格式容器镜像，因此您无需重新打包应用程序即可开始使用 rkt。

C.1.1\. 配置 Kubernetes 以使用 rkt

如您从第十一章中可能记得的，Kubelet 是唯一与容器运行时通信的 Kubernetes 组件。要使 Kubernetes 使用 rkt 而不是 Docker，您需要通过运行带有 `--container-runtime=rkt` 命令行选项的 Kubelet 来配置它。但请注意，rkt 的支持并不像 Docker 那样成熟。

请参考 Kubernetes 文档以获取有关如何使用 rkt 以及哪些功能受支持的更多信息。在此，我们将快速浏览一个示例，以帮助您入门。

C.1.2\. 尝试使用 Minikube 配置 rkt

幸运的是，要开始在 Kubernetes 上使用 rkt，您只需要与您已经使用的相同的 Minikube 可执行文件。要在 Minikube 中使用 rkt 作为容器运行时，您只需使用以下两个选项启动 Minikube 即可：

`$ minikube start --container-runtime=rkt --network-plugin=cni`


注意

您可能需要先运行 `minikube delete` 来删除现有的 Minikube VM。


`--container-runtime=rkt` 选项显然配置了 Kubelet 以使用 rkt 作为容器运行时，而 `--network-plugin=cni` 则使其使用容器网络接口作为网络插件。没有此选项，Pod 将无法运行，因此您必须使用它。

运行 Pod

一旦 Minikube VM 启动，您就可以像以前一样与 Kubernetes 交互。例如，您可以使用 `kubectl run` 命令部署 kubia 应用程序：

`$ kubectl run kubia --image=luksa/kubia --port 8080` `deployment "kubia" created`

当 Pod 启动时，您可以通过使用 `kubectl describe` 检查其容器来看到它是通过 rkt 运行的，如下所示。

列表 C.1\. 使用 rkt 运行的 Pod

`$ kubectl describe pods` `Name:           kubia-3604679414-l1nn3 ... Status:         Running IP:             10.1.0.2 Controllers:    ReplicaSet/kubia-3604679414 Containers:   kubia:     Container ID:       rkt://87a138ce-...-96e375852997:kubia` `1` `Image:              luksa/kubia     Image ID:           rkt://sha512-5bbc5c7df6148d30d74e0...` `1` `...`

+   1 容器和镜像 ID 提到 rkt 而不是 Docker。

你也可以尝试访问 pod 的 HTTP 端口，看看它是否正确响应 HTTP 请求。你可以通过创建一个 `NodePort` 服务或使用 `kubectl port-forward` 等方式来实现。

检查 Minikube VM 中运行的容器

要更熟悉 rkt，你可以尝试使用以下命令登录到 Minikube 虚拟机：

`$ minikube ssh`

然后，你可以使用 `rkt list` 来查看正在运行的 pods 和容器，如下所示。

列出 C.2\. 使用 rkt list 列出正在运行的容器

`$ rkt list` `UUID      APP                 IMAGE NAME                       STATE   ... 4900e0a5  k8s-dashboard       gcr.io/google_containers/kun...  running ... 564a6234  nginx-ingr-ctrlr    gcr.io/google_containers/ngi...  running ... 5dcafffd  dflt-http-backend   gcr.io/google_containers/def...  running ... 707a306c  kube-addon-manager  gcr.io/google-containers/kub...  running ... 87a138ce` `kubia``               registry-1.docker.io/luksa/k...  running ... d97f5c29  kubedns             gcr.io/google_containers/k8s...  running ...           dnsmasq             gcr.io/google_containers/k8...           sidecar             gcr.io/google_containers/k8...`

你可以看到 `kubia` 容器，以及其他正在运行的系统容器（在 `kube-system` 命名空间中部署的 pod）。注意底部两个容器在 `UUID` 或 `STATE` 列中没有任何内容？这是因为它们属于与上面列出的 `kubedns` 容器相同的 pod。

Rkt 将属于同一 pod 的容器分组在一起打印。每个 pod（而不是每个容器）都有自己的 UUID 和状态。如果你在以前使用 Docker 作为容器运行时尝试过这样做，你会欣赏使用 rkt 查看所有 pods 和它们的容器有多么容易。你会注意到每个 pod 都没有基础设施容器（我们在第十一章章节 11 中解释了它们）。这是因为 rkt 对 pods 的原生支持。

列出容器镜像

如果你玩过 Docker CLI 命令，你会很快熟悉 rkt 的命令。运行 `rkt` 不带任何参数，你会看到你可以运行的所有命令。例如，要列出容器镜像，你运行以下列表中的命令。

列出 C.3\. 使用 rkt image list 列出镜像

`$ rkt image list` `ID           NAME                          SIZE    IMPORT TIME  LAST USED sha512-a9c3  ...addon-manager:v6.4-beta.1  245MiB  24 min ago   24 min ago sha512-a078  .../rkt/stage1-coreos:1.24.0  224MiB  24 min ago   24 min ago sha512-5bbc  ...ker.io/luksa/kubia:latest  1.3GiB  23 min ago   23 min ago sha512-3931  ...es-dashboard-amd64:v1.6.1  257MiB  22 min ago   22 min ago sha512-2826  ...ainers/defaultbackend:1.0  15MiB   22 min ago   22 min ago sha512-8b59  ...s-controller:0.9.0-beta.4  233MiB  22 min ago   22 min ago sha512-7b59  ...dns-kube-dns-amd64:1.14.2  100MiB  21 min ago   21 min ago sha512-39c6  ...nsmasq-nanny-amd64:1.14.2  86MiB   21 min ago   21 min ago sha512-89fe  ...-dns-sidecar-amd64:1.14.2  85MiB   21 min ago   21 min ago`

这些都是 Docker 格式的容器镜像。您也可以尝试使用 acbuild 工具（可在[`github.com/containers/build`](https://github.com/containers/build)找到）构建 OCI 镜像格式（OCI 代表开放容器倡议）的镜像，并使用 rkt 运行它们。这样做超出了本书的范围，所以我会让您自己尝试。

到目前为止，本附录中解释的信息应该足以让您开始使用 rkt 与 Kubernetes 一起使用。有关 rkt 的更多信息，请参阅[`coreos.com/rkt`](https://coreos.com/rkt)文档，有关 Kubernetes 的更多信息，请参阅[`kubernetes.io/docs`](https://kubernetes.io/docs)文档。

C.2\. 通过 CRI 使用其他容器运行时

Kubernetes 对其他容器运行时的支持不仅限于 Docker 和 rkt。这两个运行时最初是直接集成到 Kubernetes 中的，但在 Kubernetes 版本 1.5 中，引入了容器运行时接口（CRI）。CRI 是一个插件 API，它使得其他容器运行时能够轻松地与 Kubernetes 集成。现在，人们可以自由地将其他容器运行时插入到 Kubernetes 中，而无需深入挖掘 Kubernetes 的代码。他们只需要实现几个接口方法即可。

从 Kubernetes 版本 1.6 开始，CRI 是 Kubelet 使用的默认接口。现在，Docker 和 rkt 都是通过 CRI 使用的（不再是直接使用）。

C.2.1\. 介绍 CRI-O 容器运行时

除了 Docker 和 rkt 之外，一个新的 CRI 实现 CRI-O 允许 Kubernetes 直接启动和管理符合 OCI 规范的容器，而无需您部署任何额外的容器运行时。

您可以通过使用`--container-runtime=crio`启动 Minikube 来尝试 CRI-O。

C.2.2\. 在虚拟机中而不是在容器中运行应用

Kubernetes 是一个容器编排系统，对吧？在本书中，我们探讨了众多特性，表明它远不止是一个编排系统，但归根结底，当您使用 Kubernetes 运行应用时，应用总是运行在容器内部，对吧？您可能会惊讶地发现，这种情况已经不再适用了。

正在开发新的 CRI 实现，使得 Kubernetes 能够在虚拟机中而不是在容器中运行应用程序。其中一个名为 Frakti 的实现允许你通过虚拟机管理程序直接运行基于 Docker 的常规容器镜像，这意味着每个容器运行自己的内核。这比它们使用相同内核时提供了更好的容器间隔离。

还有更多。另一个 CRI 实现是 Mirantis Virtlet，它使得运行实际的虚拟机镜像（在 QCOW2 镜像文件格式下，这是 QEMU 虚拟机工具使用的格式之一）成为可能，而不是容器镜像。当你使用 Virtlet 作为 CRI 插件时，Kubernetes 为每个 pod 启动一个虚拟机。这有多么酷？
