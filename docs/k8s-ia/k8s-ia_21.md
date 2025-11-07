附录 A. 使用 kubectl 管理多个集群

A.1\. 在 Minikube 和 Google Kubernetes Engine 之间切换

本书中的示例可以在使用 Minikube 创建的集群中运行，也可以在使用 Google Kubernetes Engine (GKE)创建的集群中运行。如果你计划使用两者，你需要知道如何在它们之间切换。如何在多个集群中使用`kubectl`的详细说明将在下一节中描述。这里我们来看看如何切换 Minikube 和 GKE。

切换到 Minikube

幸运的是，每次你使用`minikube start`启动 Minikube 集群时，它也会重新配置`kubectl`以使用它：

`$ minikube start` `启动本地 Kubernetes 集群... ... 设置 kubeconfig...` `1` `Kubectl 现在已配置为使用该集群。` `1`

+   1 Minikube 每次启动集群时都会设置 kubectl。

从 Minikube 切换到 GKE 后，可以通过停止 Minikube 并重新启动它来切换回来。此时，`kubectl`将重新配置以再次使用 Minikube 集群。

切换到 GKE

要切换到使用 GKE 集群，可以使用以下命令：

`$ gcloud container clusters get-credentials my-gke-cluster`

这将配置`kubectl`以使用名为`my-gke-cluster`的 GKE 集群。

进一步了解

这两种方法应该足以让你快速入门，但要了解使用`kubectl`管理多个集群的完整情况，请学习下一节。

A.2\. 使用 kubectl 管理多个集群或命名空间

如果你需要在不同 Kubernetes 集群之间切换，或者如果你想在默认命名空间之外工作，并且不想每次运行`kubectl`时都指定`--namespace`选项，以下是操作方法。

A.2.1\. 配置 kubeconfig 文件的位置

`kubectl`使用的配置通常存储在`~/.kube/config`文件中。如果它存储在其他位置，则`KUBECONFIG`环境变量需要指向其位置。

| |
| --- |

注意

你可以使用多个配置文件，并通过在`KUBECONFIG`环境变量中指定所有这些文件来让`kubectl`一次性使用它们（用冒号分隔）。

| |
| --- |

A.2.2\. 理解 kubeconfig 文件的内容

以下列出的是一个示例配置文件。

列表 A.1\. 示例 kubeconfig 文件

`apiVersion: v1 clusters: - cluster:` `1` `certificate-authority: /home/luksa/.minikube/ca.crt` `1` `server: https://192.168.99.100:8443` `1` `name: minikube` `1` `contexts: - context:` `2` `cluster: minikube` `2` `user: minikube` `2` `namespace: default` `2` `name: minikube` `2` `current-context: minikube` `3` `kind: Config preferences: {} users: - name: minikube` `4` `user:` `4` `client-certificate: /home/luksa/.minikube/apiserver.crt` `4` `client-key: /home/luksa/.minikube/apiserver.key` `4`

+   1 包含有关 Kubernetes 集群的信息

+   2 定义 kubectl 上下文

+   3 `kubectl`当前使用的上下文

+   4 包含用户的凭据

kubeconfig 文件包含四个部分：

+   簇列表

+   用户列表

+   上下文名称列表

+   当前上下文名称

每个集群、用户和上下文都有一个名称。该名称用于引用上下文、用户或集群。

集群

集群条目表示一个 Kubernetes 集群，包含 API 服务器 URL、证书颁发机构（CA）文件，以及可能与 API 服务器通信相关的其他一些配置选项。CA 证书可以存储在单独的文件中并在 kubeconfig 文件中引用，或者可以直接包含在 `certificate-authority-data` 字段中。

用户

每个用户定义在访问 API 服务器时使用的凭证。这可以是一对用户名和密码，一个身份验证令牌，或者一个客户端密钥和证书。证书和密钥可以包含在 kubeconfig 文件中（通过 `client-certificate-data` 和 `client-key-data` 属性），或者存储在单独的文件中并在配置文件中引用，如列表 A.1 所示。

上下文

上下文将一个集群、一个用户和 `kubectl` 在执行命令时应使用的默认命名空间 `kubectl` 连接起来。多个上下文可以指向同一个用户或集群。

当前上下文

虽然可以在 kubeconfig 文件中定义多个上下文，但在任何给定时间只有一个上下文是当前上下文。稍后我们将看到如何更改当前上下文。

A.2.3\. 列出、添加和修改 kube 配置条目

您可以手动编辑文件以添加、修改和删除集群、用户或上下文，但您也可以通过 `kubectl config` 命令之一来完成此操作。

添加或修改集群

要添加另一个集群，使用 `kubectl config set-cluster` 命令：

`$ kubectl config set-cluster my-other-cluster`![](img/00006.jpg)`--server=https://k8s.example.com:6443`![](img/00006.jpg)`--certificate-authority=path/to/the/cafile`

这将添加一个名为 `my-other-cluster` 的集群，API 服务器位于 https://k8s.example.com:6443。要查看可以向命令传递的附加选项，请运行 `kubectl config set-cluster` 以打印用法示例。

如果已存在同名集群，则 `set-cluster` 命令将覆盖其配置选项。

添加或修改用户凭证

添加和修改用户与添加或修改集群类似。要添加一个使用用户名和密码通过 API 服务器进行身份验证的用户，运行以下命令：

`$ kubectl config set-credentials foo --username=foo --password=pass`

要使用基于令牌的认证，运行以下命令代替：

`$ kubectl config set-credentials foo --token=mysecrettokenXFDJIQ1234`

这两个示例都将用户凭证存储在名称为 `foo` 下。如果您使用相同的凭证对不同的集群进行身份验证，您可以定义一个单独的用户并使用它与两个集群一起使用。

将集群和用户凭证关联起来

上下文定义了与哪个集群使用哪个用户，但也可以定义`kubectl`在未使用`--namespace`或`-n`选项显式指定命名空间时应使用的命名空间。

以下命令用于创建一个新的上下文，该上下文将您创建的集群和用户关联起来：

`$ kubectl config set-context some-context --cluster=my-other-cluster`![](img/00006.jpg)`--user=foo --namespace=bar`

这创建了一个名为`some-context`的上下文，它使用`my-other-cluster`集群和`foo`用户凭据。在此上下文中，默认命名空间设置为`bar`。

您也可以使用相同的命令更改当前上下文的命名空间，例如。您可以通过以下方式获取当前上下文名称：

`$ kubectl config current-context` `minikube`

然后您可以通过以下方式更改命名空间：

`$ kubectl config set-context minikube --namespace=another-namespace`

只运行一次这个简单的命令，比每次运行`kubectl`时都要包含`--namespace`选项要用户友好得多。


提示

要轻松地在命名空间之间切换，可以定义一个别名，例如：`alias kcd='kubectl config set-context $(kubectl config current-context) --namespace '`。然后您可以使用`kcd some-namespace`在命名空间之间切换。


A.2.4\. 使用 kubectl 与不同的集群、用户和上下文

当您运行`kubectl`命令时，将使用 kubeconfig 当前上下文中定义的集群、用户和命名空间，但您可以使用以下命令行选项来覆盖它们：

+   `--user` 用于使用 kubeconfig 文件中的不同用户。

+   `--username` 和 `--password` 用于使用不同的用户名和/或密码（它们不需要在配置文件中指定）。如果使用其他类型的身份验证，可以使用`--client-key` 和 `--client-certificate` 或 `--token`。

+   `--cluster` 用于使用不同的集群（必须在配置文件中定义）。

+   `--server` 用于指定不同服务器的 URL（该服务器不在配置文件中）。

+   `--namespace` 用于使用不同的命名空间。

A.2.5\. 在上下文之间切换

与之前示例中修改当前上下文不同，您还可以使用`set-context`命令创建一个额外的上下文，然后在这些上下文之间切换。当与多个集群一起工作时，这非常有用（使用`set-cluster`为它们创建集群条目）。

一旦设置了多个上下文，切换它们就变得非常简单：

`$ kubectl config use-context my-other-context`

这会将当前上下文切换到`my-other-context`。

A.2.6\. 列出上下文和集群

要列出在您的 kubeconfig 文件中定义的所有上下文，请运行以下命令：

`$ kubectl config get-contexts` `CURRENT   NAME          CLUSTER       AUTHINFO            NAMESPACE *         minikube      minikube      minikube            default           rpi-cluster   rpi-cluster   admin/rpi-cluster           rpi-foo       rpi-cluster   admin/rpi-cluster   foo`

如您所见，我正在使用三个不同的上下文。`rpi-cluster`和`rpi-foo`上下文使用相同的集群和凭据，但默认使用不同的命名空间。

列出集群的方式类似：

`$ kubectl config get-clusters` `名称 rpi-cluster minikube`

由于安全原因，无法列出凭据。

A.2.7. 删除上下文和集群

要清理上下文或集群列表，您可以手动从 kubeconfig 文件中删除条目，或者使用以下两个命令：

`$ kubectl config delete-context my-unused-context`

和

`$ kubectl config delete-cluster my-old-cluster`
