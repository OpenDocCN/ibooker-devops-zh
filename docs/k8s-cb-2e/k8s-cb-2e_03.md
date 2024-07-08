# 第三章：学习使用 Kubernetes 客户端

本章汇集了围绕 Kubernetes CLI `kubectl` 的基本用法的实用技巧。请参阅 第一章 了解如何安装 CLI 工具；对于高级用例，请参阅 第七章，我们将展示如何使用 Kubernetes API。

# 3.1 列出资源

## 问题

你想要列出特定类型的 Kubernetes 资源。

## 解决方案

使用`kubectl`的`get`动词，加上资源类型。要列出所有 pod，请执行以下操作：

```
$ kubectl get pods
```

要列出所有服务和部署（请注意逗号后面没有空格），请执行以下操作：

```
$ kubectl get services,deployments
```

要列出特定部署，请执行以下操作：

```
$ kubectl get deployment *<deployment-name>*
```

要列出所有资源，请执行以下操作：

```
$ kubectl get all
```

请注意，`kubectl get`是一个非常基本但极其有用的命令，可以快速查看集群中的情况——它本质上相当于 Unix 上的 `ps` 命令。

## 讨论

我们强烈建议启用自动完成功能，以避免记住所有 Kubernetes 资源名称。请参阅 Recipe 12.1 获取详细信息。

# 3.2 删除资源

## 问题

你不再需要资源并希望摆脱它们。

## 解决方案

使用`kubectl`的`delete`动词，加上你想要删除的资源的类型和名称。

要删除命名空间`my-app`中的所有资源，以及命名空间本身，请执行以下操作：

```
$ kubectl get ns
NAME          STATUS    AGE
default       Active    2d
kube-public   Active    2d
kube-system   Active    2d
my-app        Active    20m

$ kubectl delete ns my-app
namespace "my-app" deleted
```

注意，在 Kubernetes 中你不能删除`default`命名空间。这也是创建自己命名空间的一个好理由，因为这样清理环境会更容易。话虽如此，你仍然可以通过以下命令删除`default`命名空间中的所有对象：

```
$ kubectl delete all --all -n *<namespace>*
```

如果想知道如何创建命名空间，请参阅 Recipe 7.3。

你还可以删除特定资源和/或影响它们被销毁的过程。要删除标记有`app=niceone`的服务和部署，请执行以下操作：

```
$ kubectl delete svc,deploy -l app=niceone
```

要强制删除名为`hangingpod`的 pod，请执行以下操作：

```
$ kubectl delete pod hangingpod --grace-period=0 --force
```

要删除命名空间`test`中的所有 pod，请执行以下操作：

```
$ kubectl delete pods --all --namespace test
```

## 讨论

不要删除由部署直接控制的监督对象，如 pod 或副本集。相反，终止它们的监督或使用专门的操作来摆脱托管资源。例如，如果将部署的副本数缩减至零（参见 Recipe 9.1），那么你有效地删除了它管理的所有 pod。

另一个需要考虑的方面是级联删除与直接删除的区别——例如，当你删除自定义资源定义（CRD）时，如 Recipe 15.4 中所示，其依赖对象也会被删除。要了解如何影响级联删除策略，请阅读 Kubernetes 文档中的 [垃圾回收](https://oreil.ly/8AcpW)。

# 3.3 使用 kubectl 监视资源变更

## 问题

你希望在终端中以交互方式观察 Kubernetes 对象的变更。

## 解决方案

`kubectl` 命令有一个 `--watch` 选项，可以实现这种行为。例如，要监视 Pods，请执行以下操作：

```
$ kubectl get pods --watch
```

请注意，这是一个阻塞和自动更新的命令，类似于 `top`。

## 讨论

`--watch` 选项很有用，但有些人更喜欢来自 [`watch` 命令](https://oreil.ly/WPueN) 的输出格式，例如：

```
$ watch kubectl get pods
```

# 3.4 使用 kubectl 编辑对象

## 问题

你想要更新 Kubernetes 对象的属性。

## 解决方案

使用 `kubectl` 的 `edit` 命令以及对象类型：

```
$ kubectl run nginx --image=nginx
$ kubectl edit pod/nginx
```

现在在编辑器中编辑 `nginx` 的 Pod —— 例如，添加一个名为 `mylabel` 值为 `true` 的新标签。保存后，你会看到类似这样的输出：

```
pod/nginx edited
```

## 讨论

如果你的编辑器无法打开，或者你想指定使用哪个编辑器，请将 `EDITOR` 或 `KUBE_EDITOR` 环境变量设置为你希望使用的编辑器名称。例如：

```
$ export EDITOR=vi
```

同样要注意，并非所有更改都会触发对象更新。

有些触发器有快捷方式；例如，如果你想要更改 Deployment 使用的镜像版本，只需使用 `kubectl set image`，它会更新资源（适用于 Deployments、ReplicaSets/Replication Controllers、DaemonSets、Jobs 和简单 Pods）的现有容器镜像。

# 3.5 请求 kubectl 解释资源和字段

## 问题

你想更深入地了解某个资源，例如一个 `Service`，和/或者准确理解 Kubernetes 清单中某个字段的含义，包括默认值以及是否必填或可选。

## 解决方案

使用 `kubectl` 的 `explain` 命令：

```
$ kubectl explain svc
KIND:       Service
VERSION:    v1

DESCRIPTION:
Service is a named abstraction of software service (for example, mysql)
consisting of local port (for example 3306) that the proxy listens on, and the
selector that determines which pods will answer requests sent through the proxy.

FIELDS:
   status       <Object>
     Most recently observed status of the service. Populated by the system.
     Read-only. More info: https://git.k8s.io/community/contributors/devel/
     api-conventions.md#spec-and-status/

   apiVersion   <string>
     APIVersion defines the versioned schema of this representation of an
     object. Servers should convert recognized schemas to the latest internal
     value, and may reject unrecognized values. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#resources

   kind <string>
     Kind is a string value representing the REST resource this object
     represents. Servers may infer this from the endpoint the client submits
     requests to. Cannot be updated. In CamelCase. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions
     .md#types-kinds

   metadata     <Object>
     Standard object's metadata. More info:
     https://git.k8s.io/community/contributors/devel/api-conventions.md#metadata

   spec <Object>
     Spec defines the behavior of a service. https://git.k8s.io/community/
     contributors/devel/api-conventions.md#spec-and-status/

$ kubectl explain svc.spec.externalIPs
KIND:       Service
VERSION:    v1

FIELD: externalIPs <[]string>

DESCRIPTION:
     externalIPs is a list of IP addresses for which nodes in the cluster will
     also accept traffic for this service.  These IPs are not managed by
     Kubernetes.  The user is responsible for ensuring that traffic arrives at a
     node with this IP.  A common example is external load-balancers that are not
     part of the Kubernetes system.
```

## 讨论

[`kubectl explain`](https://oreil.ly/chI_-) 命令从 API 服务器公开的 [Swagger/OpenAPI 定义](https://oreil.ly/19vi3) 中提取资源和字段的描述信息。

你可以把 `kubectl explain` 看作是描述 Kubernetes 资源结构的一种方式，而 `kubectl describe` 则是描述对象值的一种方式，它们是这些结构化资源的实例。

## 参见

+   Ross Kukulinski 的博客文章，[“kubectl explain — #HeptioProTip”](https://oreil.ly/LulwG)
