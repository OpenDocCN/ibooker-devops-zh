# 第七章：探索 Kubernetes API 和关键元数据

在本章中，我们提供了处理 Kubernetes 对象及其 API 基本交互的实例。每个 [Kubernetes 中的对象](https://oreil.ly/kMcj7)，无论是像部署一样的有命名空间的对象，还是像节点一样的全局对象，都有一些可用的字段 — 例如 `metadata`、`spec` 和 `status`。`spec` 描述了对象的期望状态（规范），而 `status` 则捕获了对象的实际状态，由 Kubernetes API 服务器管理。

# 7.1 探索 Kubernetes API 服务器的端点

## 问题

您希望发现 Kubernetes API 服务器上可用的各种 API 端点。

## 解决方案

在这里，我们假设您已经在本地启动了一个开发集群，如 kind 或 Minikube。您可以在单独的终端中运行 `kubectl proxy`。代理允许您使用诸如 `curl` 这样的 HTTP 客户端轻松访问 Kubernetes 服务器 API，而无需担心身份验证和证书。运行 `kubectl proxy` 后，您应该能够通过端口 8001 访问 API 服务器，如下所示：

```
$ curl http://localhost:8001/api/v1/
{
  "kind": "APIResourceList",
  "groupVersion": "v1",
  "resources": [
    {
      "name": "bindings",
      "singularName": "",
      "namespaced": true,
      "kind": "Binding",
      "verbs": [
        "create"
      ]
    },
    {
      "name": "componentstatuses",
      "singularName": "",
      "namespaced": false,
      ...
```

这列出了 Kubernetes API 暴露的所有对象。在列表顶部，您可以看到一个对象的示例，其 `kind` 设置为 `Binding`，以及此对象上允许的操作（此处为 `create`）。

注意，发现 API 端点的另一个便捷方式是使用 `kubectl api-resources` 命令。

## 讨论

您可以通过调用以下端点来发现所有的 API 组：

```
$ curl http://localhost:8001/apis/
{
  "kind": "APIGroupList",
  "apiVersion": "v1",
  "groups": [
    {
      "name": "apiregistration.k8s.io",
      "versions": [
        {
          "groupVersion": "apiregistration.k8s.io/v1",
          "version": "v1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "apiregistration.k8s.io/v1",
        "version": "v1"
      }
    },
    {
      "name": "apps",
      "versions": [
      ...
```

从此列表中选择一些要探索的 API 组，例如以下内容：

+   `/apis/apps`

+   `/apis/storage.k8s.io`

+   `/apis/flowcontrol.apiserver.k8s.io`

+   `/apis/autoscaling`

每个这些端点对应一个 API 组。核心 API 对象在 `/api/v1` 组中可用，而其他更新的 API 对象在 `/apis/` 端点下的命名组中可用，例如 `storage.k8s.io/v1` 和 `apps/v1`。在一个组内，API 对象被版本化（例如 `v1`、`v2`、`v1alpha`、`v1beta1`）以指示对象的成熟度。例如，Pods、services、config maps 和 secrets 都是 `/api/v1` API 组的一部分，而 `/apis/autoscaling` 组则有 `v1` 和 `v2` 版本。

对象所属的组被称为对象规范中的 `apiVersion`，可通过 [API 参考](https://oreil.ly/fvO82) 获取。

## 参见

+   Kubernetes [API 概述](https://oreil.ly/sANzL)

+   Kubernetes [API 约定](https://oreil.ly/ScJvH)

# 7.2 理解 Kubernetes 清单的结构

## 问题

尽管 Kubernetes 具有方便的生成器如 `kubectl run` 和 `kubectl create`，您必须学会如何编写 Kubernetes 清单文件以支持 Kubernetes 对象规范的声明性特性。为此，您需要理解清单的一般结构。

## 解决方案

在 Recipe 7.1 中，您了解了各种 API 组以及如何发现特定对象所属的组。

所有 API 资源都是对象或列表。所有资源都有一个 `kind` 和一个 `ap⁠i​Ve⁠rs⁠ion`。此外，每个对象 `kind` 必须有 `metadata`。`metadata` 包含对象的名称、它所在的命名空间（参见 Recipe 7.3）、以及可选的一些标签（参见 Recipe 7.6）和注释（参见 Recipe 7.7）。

例如，一个 pod 将是 `kind Pod` 和 `apiVersion v1`，并且以 YAML 编写的简单清单的开头看起来像这样：

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
...
```

要完成一个清单，大多数对象都将有一个 `spec`，一旦创建，还会返回一个描述对象当前状态的 `status`：

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  ...
status:
  ...
```

## 讨论

Kubernetes 清单可用于定义集群的期望状态。由于清单是文件，可以将其存储在像 Git 这样的版本控制系统中。这允许开发人员和运维人员进行分布式和异步的协作，并且还支持为持续集成和部署创建自动化流水线。这就是 GitOps 的基本理念，其中对系统的任何更改都是通过更改版本控制系统中的真实源来进行的。由于所有更改都记录在系统中，因此可以恢复到先前的状态或多次重现给定的状态。基础设施即代码（IaC）是一个经常用来描述基础设施状态的声明性真实源的术语（与应用程序相对）。

## 参见

+   [Kubernetes 中的对象](https://oreil.ly/EONxU)

# 7.3 创建命名空间以避免名称冲突

## 问题

您希望创建两个具有相同名称的对象，但希望避免名称冲突。

## 解决方案

创建两个命名空间，并在每个命名空间中创建一个对象。

如果您没有指定任何内容，则对象将在 `default` 命名空间中创建。尝试创建一个名为 `my-app` 的第二个命名空间，如下所示，并列出现有的命名空间。您将看到 `default` 命名空间，以及启动时创建的其他命名空间（`kube-system`、`kube-public` 和 `kube-node-lease`），以及您刚刚创建的 `my-app` 命名空间：

```
$ kubectl create namespace my-app
namespace/my-app created

$ kubectl get ns
NAME                 STATUS   AGE
default              Active   5d20h
kube-node-lease      Active   5d20h
kube-public          Active   5d20h
kube-system          Active   5d20h
my-app               Active   13s

```

###### 注意

或者，您可以编写一个清单来创建您的命名空间。如果您将以下清单保存为 *app.yaml*，然后可以使用 `kubectl apply -f app.yaml` 命令创建命名空间：

```
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
```

## 讨论

尝试在同一命名空间中启动两个具有相同名称的对象（例如 `default`）会导致冲突，并且 Kubernetes API 服务器将返回错误。但是，如果您在不同的命名空间中启动第二个对象，API 服务器将会创建它：

```
$ kubectl run foobar --image=nginx:latest
pod/foobar created

$ kubectl run foobar --image=nginx:latest
Error from server (AlreadyExists): pods "foobar" already exists

$ kubectl run foobar --image=nginx:latest --namespace my-app
pod/foobar created

```

###### 注意

`kube-system` 命名空间是保留给管理员的，而 [`kube-public` 命名空间](https://oreil.ly/kQFsq) 则用于存储对集群中任何用户可用的公共对象。

# 7.4 在命名空间内设置配额

## 问题

您希望限制命名空间中可用的资源，例如命名空间中可以运行的 pod 总数。

## 解决方案

使用 `ResourceQuota` 对象来指定命名空间基础上的限制。

首先创建一个资源配额的清单，并将其保存在名为 *resource-quota-pods.yaml* 的文件中：

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: podquota
spec:
  hard:
    pods: "10"

```

然后创建一个新的命名空间并对其应用配额：

```
$ kubectl create namespace my-app
namespace/my-app created

$ kubectl apply -f resource-quota-pods.yaml --namespace=my-app
resourcequota/podquota created

$ kubectl describe resourcequota podquota --namespace=my-app
Name:       podquota
Namespace:  my-app
Resource    Used  Hard
--------    ----  ----
pods        1     10

```

## 讨论

您可以在每个命名空间基础上设置多个配额，包括但不限于 pods、secrets 和 config maps。

## 参见

+   [为 API 对象配置配额](https://oreil.ly/jneBT)

# 7.5 给对象打标签

## 问题

您希望为对象打上标签，以便稍后轻松找到它。标签可用于进一步的最终用户查询（参见 Recipe 7.6）或在系统自动化的上下文中使用。

## 解决方案

使用 `kubectl label` 命令。例如，要为名为 `foobar` 的 pod 打上键/值对 `tier=frontend` 的标签，请执行以下操作：

```
$ kubectl label pods foobar tier=frontend
pod/foobar labeled

```

###### 提示

查看命令的完整帮助 (`kubectl label --help`)。您可以使用它来了解如何删除标签、覆盖现有标签，甚至为命名空间中的所有资源打上标签。

## 讨论

在 Kubernetes 中，您可以使用标签以灵活、非层次化的方式组织对象。标签是一个在 Kubernetes 中没有预定义含义的键/值对。换句话说，系统不会解释键/值对的内容。您可以使用标签来表示成员关系（例如，对象 X 属于部门 ABC）、环境（例如，此服务在生产环境运行）或任何需要组织对象的内容。您可以在 [Kubernetes 文档](https://oreil.ly/SMl_N) 中了解一些常用的有用标签。请注意，标签在其 [长度和允许值](https://oreil.ly/AzeM8) 方面有一些限制。然而，对于键的命名，有一个 [社区指南](https://oreil.ly/lTkhW)。

# 7.6 使用标签进行查询

## 问题

您希望高效查询对象。

## 解决方案

使用 `kubectl get --selector` 命令。例如，给定以下的 pods：

```
$ kubectl get pods --show-labels
NAME         READY   STATUS    RESTARTS        AGE     LABELS
foobar       1/1     Running   0               18m     run=foobar,tier=frontend
nginx1       1/1     Running   0               72s     app=nginx,run=nginx1
nginx2       1/1     Running   0               68s     app=nginx,run=nginx2
nginx3       1/1     Running   0               65s     app=nginx,run=nginx3

```

您可以选择属于 NGINX 应用程序 (`app=nginx`) 的 pods：

```
$ kubectl get pods --selector app=nginx
NAME     READY   STATUS    RESTARTS   AGE
nginx1   1/1     Running   0          3m45s
nginx2   1/1     Running   0          3m41s
nginx3   1/1     Running   0          3m38s

```

## 讨论

标签是对象元数据的一部分。在 Kubernetes 中，任何对象都可以被标记。标签也被 Kubernetes 本身用于通过 deployments（参见 Recipe 4.1）和 services（参见 Chapter 5）选择 pod。

标签可以通过 `kubectl label` 命令手动添加（参见 Recipe 7.5），或者您可以在对象清单中定义标签，如下所示：

```
apiVersion: v1
kind: Pod
metadata:
  name: foobar
  labels:
    tier: frontend
...
```

一旦标签存在，您可以使用 `kubectl get` 列出它们，注意以下内容：

+   `-l` 是 `--selector` 的缩写形式，将查询具有指定 `*key*``=``*value*` 对的对象。

+   `--show-labels` 将显示返回的每个对象的所有标签。

+   `-L` 将在返回的结果中添加一个列，该列的值为指定标签的值。

+   许多对象类型支持基于集合的查询，这意味着您可以以“必须具有 X 和/或 Y 标签”的形式进行查询。例如，`kubectl get pods -l 'env in (production, development)'` 将给出在生产环境或开发环境中的 Pod。

当有两个运行中的 Pod，一个带有标签`run=barfoo`，另一个带有标签`ru⁠n=​fo⁠oba⁠r`时，您将会得到类似以下的输出：

```
$ kubectl get pods --show-labels
NAME                      READY   ...    LABELS
barfoo-76081199-h3gwx     1/1     ...    pod-template-hash=76081199,run=barfoo
foobar-1123019601-6x9w1   1/1     ...    pod-template-hash=1123019601,run=foobar

$ kubectl get pods -L run
NAME                      READY   ...    RUN
barfoo-76081199-h3gwx     1/1     ...    barfoo
foobar-1123019601-6x9w1   1/1     ...    foobar

$ kubectl get pods -l run=foobar
NAME                      READY   ...
foobar-1123019601-6x9w1   1/1     ...

```

## 参见

+   Kubernetes 上关于[标签和选择器](https://oreil.ly/ku1Sc)的文档。

# 7.7 使用一个命令给资源加注释

## 问题

您希望使用一个通用的、非标识的键值对对资源进行注释，可能使用非人类可读的数据。

## 解决方案

使用`kubectl annotate`命令：

```
$ kubectl annotate pods foobar \
    description='something that you can use for automation'
pod/foobar annotated

```

## 讨论

注释通常用于增强 Kubernetes 的自动化。例如，当您使用`kubectl create deployment`命令创建一个部署时，您会注意到您的发布历史中的`change-cause`列（参见 Recipe 4.6）是空的。从 Kubernetes v1.6.0 开始，您可以使用`kubernetes.io/change-cause`键对其进行注释，以开始记录导致部署变更的命令。假设有一个名为 `foobar` 的部署，您可以使用以下内容对其进行注释：

```
$ kubectl annotate deployment foobar \
    kubernetes.io/change-cause="Reason for creating a new revision"

```

对部署的后续更改将被记录下来。

注释和标签之间的主要区别之一在于，标签可以用作过滤条件，而注释则不能。除非您计划将元数据用于过滤，否则通常更倾向于使用注释。
