# 第四章：常用 kubectl 命令

`kubectl`命令行实用程序是一个强大的工具，在接下来的章节中，您将使用它来创建对象并与 Kubernetes API 进行交互。然而，在此之前，先了解适用于所有 Kubernetes 对象的基本`kubectl`命令是有意义的。

# 命名空间

Kubernetes 使用*命名空间*来组织集群中的对象。您可以将每个命名空间视为一个包含一组对象的文件夹。默认情况下，`kubectl`命令行工具与`default`命名空间交互。如果您想使用不同的命名空间，可以传递`kubectl``--namespace`标志。例如，`kubectl --namespace=mystuff` 引用 `mystuff` 命名空间中的对象。如果您感到简洁，还可以使用缩写 `-n` 标志。如果您想与所有命名空间交互——例如，列出集群中所有 Pod——可以传递 `--all-namespaces` 标志。

# 上下文

如果您想更长期地更改默认命名空间，可以使用*上下文*。这将记录在一个`kubectl`配置文件中，通常位于*$HOME/.kube/config*。这个配置文件还存储了如何找到和认证您的集群。例如，您可以使用以下命令为您的`kubectl`命令创建一个具有不同默认命名空间的上下文：

```
$ kubectl config set-context my-context --namespace=mystuff
```

这将创建一个新的上下文，但实际上并没有开始使用它。要使用这个新创建的上下文，您可以运行：

```
$ kubectl config use-context my-context
```

上下文还可以用于管理不同的集群或不同用户，通过使用`set-context`命令的`--users`或`--clusters`标志进行认证。

# 查看 Kubernetes API 对象

Kubernetes 中的所有内容都由 RESTful 资源表示。在本书中，我们将这些资源称为*Kubernetes 对象*。每个 Kubernetes 对象存在于唯一的 HTTP 路径；例如，*https://your-k8s.com/api/v1/namespaces/default/pods/my-pod* 指向默认命名空间中名为`my-pod`的 Pod 的表示。`kubectl`命令通过这些 URL 向这些路径发出 HTTP 请求，以访问驻留在这些路径上的 Kubernetes 对象。

通过`kubectl`查看 Kubernetes 对象的最基本命令是`get`。如果您运行`kubectl get <*resource-name*>`，您将获得当前命名空间中所有资源的列表。如果您想获取特定资源，可以使用`kubectl get <*resource-name*> <*obj-name*>`。

默认情况下，`kubectl`使用人类可读的打印机来查看来自 API 服务器的响应，但这种人类可读的打印机会移除对象的许多细节以适应每个终端行。要获取稍多一些信息的一种方法是添加`-o wide`标志，这会在较长的行上提供更多细节。如果您想查看完整的对象，还可以使用`-o json`或`-o yaml`标志查看对象的原始 JSON 或 YAML 格式。

用于操作`kubectl`输出的常见选项是移除标题，这在将`kubectl`与 Unix 管道结合使用时通常很有用（例如，`kubectl ... | awk ...`）。如果指定`--no-headers`标志，`kubectl`将跳过人类可读表格顶部的标题。

另一个常见的任务是从对象中提取特定字段。`kubectl`使用 JSONPath 查询语言来选择返回对象中的字段。 JSONPath 的完整细节超出了本章的范围，但作为示例，此命令将提取并打印指定 Pod 的 IP 地址：

```
$ kubectl get pods my-pod -o jsonpath --template={.status.podIP}
```

您还可以通过使用逗号分隔的类型列表查看不同类型的多个对象，例如：

```
$ kubectl get pods,services
```

这将显示给定命名空间的所有 Pod 和服务。

如果您对特定对象的更详细信息感兴趣，请使用`describe`命令：

```
$ kubectl describe <*resource-name*> <*obj-name*>
```

这将提供对象的丰富多行人类可读描述，以及 Kubernetes 集群中任何其他相关对象和事件。

如果您想查看每种支持类型的 Kubernetes 对象的受支持字段列表，可以使用`explain`命令：

```
$ kubectl explain pods
```

有时候，您可能希望持续观察特定 Kubernetes 资源的状态，以便在发生更改时看到资源的变化。例如，您可能正在等待应用程序重新启动。 `--watch`标志可以启用此功能。您可以将此标志添加到任何`kubectl get`命令中，以持续监视特定资源的状态。

# 创建、更新和销毁 Kubernetes 对象

Kubernetes API 中的对象以 JSON 或 YAML 文件表示。这些文件可以由服务器作为响应查询返回，也可以作为 API 请求的一部分发布到服务器上。您可以使用这些 YAML 或 JSON 文件在 Kubernetes 服务器上创建、更新或删除对象。

假设您在*obj.yaml*中有一个简单的对象存储。您可以使用`kubectl`通过运行以下命令在 Kubernetes 中创建此对象：

```
$ kubectl apply -f obj.yaml
```

注意您不需要指定对象的资源类型；它从对象文件本身获取。

类似地，在对对象进行更改后，您可以再次使用`apply`命令来更新对象：

```
$ kubectl apply -f obj.yaml
```

`apply`工具仅会修改与集群中当前对象不同的对象。如果您要创建的对象已经存在于集群中，则它将简单地成功退出而不进行任何更改。这使其在希望确保集群状态与文件系统状态匹配的循环中非常有用。您可以重复使用`apply`来调和状态。

如果您想查看`apply`命令在不实际进行更改的情况下将要执行的操作，可以使用`--dry-run`标志将对象打印到终端而不实际发送到服务器。

###### 注意

如果您希望进行交互式编辑而不是编辑本地文件，您可以使用`edit`命令，该命令将下载最新的对象状态，然后启动包含定义的编辑器：

```
$ kubectl edit <*resource-name*> <*obj-name*>
```

保存文件后，它将自动上传回 Kubernetes 集群。

`apply`命令还将先前配置的历史记录在对象内的注释中记录下来。您可以使用`edit-last-applied`、`set-last-applied`和`view-last-applied`命令来操作这些记录。例如：

```
$ kubectl apply -f myobj.yaml view-last-applied
```

将显示应用于对象的最后状态。

当您想要删除一个对象时，您可以简单地运行：

```
$ kubectl delete -f obj.yaml
```

使用 `kubectl` 删除对象时不会提示确认删除。一旦执行命令，对象*将*被删除。

同样，您可以使用资源类型和名称来删除对象：

```
$ kubectl delete <*resource-name*> <*obj-name*>
```

# 对象标记和注释

标签和注释是您对象的标记。我们将在第六章中讨论它们的差异，但现在，您可以使用`label`和`annotate`命令更新任何 Kubernetes 对象上的标签和注释。例如，要向名为`bar`的 Pod 添加`color=red`标签，您可以运行：

```
$ kubectl label pods bar color=red
```

注释的语法是相同的。

默认情况下，`label`和`annotate`不允许覆盖现有标签。要实现此目的，您需要添加`--overwrite`标志。

如果要删除标签，您可以使用*`<label-name>-`*语法：

```
$ kubectl label pods bar color-
```

这将从名为`bar`的 Pod 中删除`color`标签。

# 调试命令

`kubectl`还提供了许多用于调试您的容器的命令。您可以使用以下命令查看正在运行容器的日志：

```
$ kubectl logs <*pod-name*>
```

如果您的 Pod 中有多个容器，您可以使用`-c`标志选择要查看的容器。

默认情况下，`kubectl logs`列出当前日志并退出。如果您希望连续将日志流式传输回终端而不退出，则可以添加`-f`（跟随）命令行标志。

您还可以使用`exec`命令在运行的容器中执行命令：

```
$ kubectl exec -it <*pod-name*> -- bash
```

这将为您提供一个在运行的容器内的交互式 shell，以便您可以执行更多调试。

如果您的容器内没有 bash 或其他终端可用，您始终可以`attach`到正在运行的进程：

```
$ kubectl attach -it *<pod-name>*
```

`attach`命令类似于`kubectl logs`，但允许您将输入发送到运行的进程，假设该进程已设置为从标准输入读取。

您还可以使用`cp`命令在容器之间复制文件：

```
$ kubectl cp <*pod-name>:</path/to/remote/file> </path/to/local/file>*
```

这将从正在运行的容器中复制文件到您的本地机器。您还可以指定目录，或者颠倒语法以从本地机器复制文件到容器外。

如果您想通过网络访问您的 Pod，您可以使用`port-forward`命令将本地机器上的网络流量转发到 Pod。这使您能够安全地通过隧道传输网络流量到可能不会在公共网络上暴露的容器中。例如，以下命令：

```
$ kubectl port-forward *<pod-name>* 8080:80
```

打开一个连接，将本地机器上 8080 端口的流量转发到远程容器的 80 端口。

###### 注意

您还可以使用`port-forward`命令通过指定`services/*<service-name>*`而不是`*<pod-name>*`与服务一起使用。但请注意，如果您对服务进行端口转发，请求将只转发到该服务中的单个 Pod。它们不会经过服务负载均衡器。

如果您想查看 Kubernetes 事件，您可以使用`kubectl get events`命令查看给定命名空间中所有对象的最新 10 个事件列表：

```
$ kubectl get events
```

您还可以通过在`kubectl get events`命令中添加`--watch`来实时查看事件。您可能还希望添加`-A`以查看所有命名空间中的事件。

最后，如果您对集群如何使用资源感兴趣，您可以使用`top`命令查看正在使用的节点或 Pod 资源列表。此命令：

```
$ kubectl top nodes
```

将显示节点使用的总 CPU 和内存，以绝对单位（例如，核心）和可用资源的百分比（例如，总核心数）来计算。同样，此命令：

```
$ kubectl top pods
```

显示所有 Pod 及其资源使用情况。默认情况下，它只显示当前命名空间中的 Pod，但您可以添加`--all-namespaces`标志以查看集群中所有 Pod 的资源使用情况。

这些`top`命令仅在集群中运行有指标服务器时才有效。几乎每个托管的 Kubernetes 环境和许多非托管环境中都存在指标服务器。但如果这些命令失败，可能是因为您需要安装一个指标服务器。

# 集群管理

`kubectl`工具也可以用来管理集群本身。人们在管理集群时最常见的操作是针对特定节点进行关闭和排空。当您*cordon*一个节点时，您阻止将来的 Pod 被调度到该节点上。当您*drain*一个节点时，您会移除当前在该节点上运行的任何 Pod。这些命令的一个典型用例是移除需要维修或升级的物理机器。在这种情况下，您可以先使用`kubectl cordon`，然后再使用`kubectl drain`安全地将该机器从集群中移除。一旦机器修复完成，您可以使用`kubectl uncordon`重新启用将 Pod 调度到该节点上。并没有`undrain`命令；Pods 会自然地被调度到空节点上，因为它们被创建。对于影响节点的快速操作（例如，机器重新启动），通常不需要进行 cordon 或 drain 操作；只有当机器将长时间停止服务时，您才需要将 Pods 移动到其他机器。

# 命令自动补全

`kubectl` 支持与您的 shell 集成，以启用命令和资源的选项补全。根据您的环境，可能需要先安装 `bash-completion` 包，然后再激活命令自动补全。您可以使用适当的包管理器执行此操作：

```
# macOS
$ brew install bash-completion

# CentOS/Red Hat
$ yum install bash-completion

# Debian/Ubuntu
$ apt-get install bash-completion

```

在 macOS 上安装时，请确保按照 `brew` 的说明操作，以启用使用你的 *${HOME}/.bash_profile* 进行选项补全的方法。

安装了 `bash-completion` 后，您可以通过以下方式暂时激活终端的自动补全：

```
$ source <(kubectl completion bash)
```

要使这对每个终端自动完成，请将其添加到您的 *${HOME}/.bashrc* 文件中：

```
$ echo "source <(kubectl completion bash)" >> ${HOME}/.bashrc
```

如果您使用 `zsh`，可以在网上找到类似的 [说明](https://oreil.ly/aYujA)。

# 查看集群的其他方法

除了 `kubectl`，还有其他用于与 Kubernetes 集群交互的工具。例如，有几个编辑器的插件可以将 Kubernetes 与编辑器环境集成，包括：

+   [Visual Studio Code](http://bit.ly/32ijGV1)

+   [IntelliJ](http://bit.ly/2Gen1eG)

+   [Eclipse](http://bit.ly/2XHi6gP)

如果您正在使用托管的 Kubernetes 服务，大多数服务还提供了集成到他们基于 Web 的用户体验中的 Kubernetes 图形界面。公共云中的托管 Kubernetes 还集成了复杂的监控工具，帮助您深入了解应用程序的运行情况。

Kubernetes 还有几个开源图形界面，包括 [Rancher Dashboard](https://oreil.ly/mliob) 和 [Headlamp 项目](https://oreil.ly/lDvbs)。

# 总结

`kubectl` 是管理 Kubernetes 集群中应用程序的强大工具。本章展示了该工具的许多常见用法，但 `kubectl` 提供了大量的内置帮助。您可以从以下方式开始查看此帮助：

```
$ kubectl help
```

或者：

```
$ kubectl help *<command-name>*
```
