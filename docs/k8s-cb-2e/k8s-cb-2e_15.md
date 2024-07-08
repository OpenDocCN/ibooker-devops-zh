# 第十五章：扩展 Kubernetes

现在你已经看到了如何安装、交互和使用 Kubernetes 来部署和管理应用程序，本章节我们将重点放在如何将 Kubernetes 调整为符合你需求的地方。在本章的示例中，你需要安装 [Go](https://go.dev) 并且可以访问托管在 [GitHub](https://github.com/kubernetes/kubernetes) 上的 Kubernetes 源代码。我们展示如何编译整个 Kubernetes，以及如何编译像客户端 `kubectl` 这样的特定组件。我们还演示了如何使用 Python 与 Kubernetes API 服务器通信，并展示了如何通过自定义资源定义来扩展 Kubernetes。

# 15.1 编译源码

## 问题

你想要从源码构建自己的 Kubernetes 二进制文件，而不是下载官方发布的二进制文件（参见 Recipe 2.9）或第三方构件。

## 解决方案

克隆 Kubernetes Git 仓库并从源码构建。

如果你的开发机器已经安装了 Docker Engine，你可以使用根目录 *Makefile* 的 `quick-release` 目标，如下所示：

```
$ git clone https://github.com/kubernetes/kubernetes.git
$ cd kubernetes
$ make quick-release

```

###### 提示

这种基于 Docker 的构建需要至少 8 GB 的 RAM 才能完成。确保你的 Docker 守护进程可以访问到足够的内存。在 macOS 上，访问 Docker for Mac 首选项并增加分配的 RAM。

二进制文件将位于 *_output/release-stage* 目录中，完整的捆绑包将位于 *_output/release-tars* 目录中。

或者，如果你已经正确设置了 [Golang](https://go.dev/doc/install) 环境，可以使用根目录 *Makefile* 的 `release` 目标：

```
$ git clone https://github.com/kubernetes/kubernetes.git
$ cd kubernetes
$ make

```

二进制文件将位于 *_output/bin* 目录中。

## 参见

+   Kubernetes 的 [开发指南](https://oreil.ly/6CSWo)

# 15.2 编译特定组件

## 问题

你想要从源码中构建 Kubernetes 的某个特定组件。例如，你只想要构建客户端 `kubectl`。

## 解决方案

而不是使用 `make quick-release` 或简单地使用 `make`，如 Recipe 15.1 中所示，使用 `make kubectl`。

在根目录的 *Makefile* 中有目标可以构建各个单独的组件。例如，要编译 `kubectl`、`kubeadm` 和 `kubelet`，请执行以下操作：

```
$ make kubectl kubeadm kubelet

```

二进制文件将位于 *_output/bin* 目录中。

###### 提示

要获取 *Makefile* 的完整构建目标列表，请运行 `make help`。

# 15.3 使用 Python 客户端与 Kubernetes API 交互

## 问题

作为开发者，你想要使用 Python 编写脚本来使用 Kubernetes API。

## 解决方案

安装 Python 的 `kubernetes` 模块。这个 [模块](https://oreil.ly/OolLt) 是 Kubernetes 的官方 Python 客户端库。你可以从源码或者 [Python Package Index (PyPi) 网站](https://pypi.org) 安装这个模块：

```
$ pip install kubernetes

```

使用默认的 `kubectl` 上下文可以访问 Kubernetes 集群后，你现在可以准备使用这个 Python 模块与 Kubernetes API 交互了。例如，下面的 Python 脚本列出所有的 Pod 并打印它们的名称：

```
from kubernetes import client, config

config.load_kube_config()

v1 = client.CoreV1Api()
res = v1.list_pod_for_all_namespaces(watch=False)
for pod in res.items:
    print(pod.metadata.name)
```

此脚本中的`config.load_kube_config()`调用将从您的`kubectl`配置文件中加载 Kubernetes 凭据和终端。默认情况下，它将为当前上下文加载集群终端和凭据。

## 讨论

使用 OpenAPI 规范构建的 Python 客户端。它是最新的并且是自动生成的。所有 API 都可以通过此客户端访问。

每个 API 组对应于一个特定的类，因此要在属于`/api/v1` API 组的 API 对象上调用方法，您需要实例化`CoreV1Api`类。要使用部署，您需要实例化`extensionsV1beta1Api`类。可以在自动生成的[*README*](https://oreil.ly/ITREP)中找到所有方法和相应的 API 组实例。

## 参见

+   [项目存储库中的示例](https://oreil.ly/6rw3l)

# 15.4 扩展 API 使用自定义资源定义

## 问题

您有一个自定义工作负载，而现有的资源（如`Deployment`，`Job`或`StatefulSet`）都不太适合。因此，您希望通过新资源扩展 Kubernetes API，表示您的工作负载，同时继续以通常的方式使用`kubectl`。

## 解决方案

使用[自定义资源定义（CRD）](https://oreil.ly/d2MmH)。

假设您想要定义一种名为`Function`的自定义资源。这代表了一种类似于 AWS Lambda 提供的短期运行`Job`的资源，即函数即服务（FaaS，有时误称为“无服务器函数”）。

###### 注意

对于在 Kubernetes 上运行的生产就绪 FaaS 解决方案，请参见第十四章。

首先，在名为*functions-crd.yaml*的清单文件中定义 CRD：

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: functions.example.com
spec:
  group: example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              code:
                type: string
              ram:
                type: string
  scope:              Namespaced
  names:
    plural: functions
    singular: function
    kind: Function
```

然后让 API 服务器知道您的新 CRD（注册可能需要几分钟）：

```
$ kubectl apply -f functions-crd.yaml
customresourcedefinition.apiextensions.k8s.io/functions.example.com created

```

现在您已经定义了`Function`的自定义资源，并且 API 服务器知道它，您可以使用名为*myfaas.yaml*的清单来实例化它，内容如下：

```
apiVersion: example.com/v1
kind: Function
metadata:
  name: myfaas
spec:
  code: "http://src.example.com/myfaas.js"
  ram: 100Mi
```

然后按照通常的方式创建`myfaas`类型为`Function`的资源：

```
$ kubectl apply -f myfaas.yaml
function.example.com/myfaas created

$ kubectl get crd functions.example.com -o yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  creationTimestamp: "2023-05-02T12:12:03Z"
  generation: 1
  name: functions.example.com
  resourceVersion: "2494492251"
  uid: 5e0128b3-95d9-412b-b84d-b3fac030be75
spec:
  conversion:
    strategy: None
  group: example.com
  names:
    kind: Function
    listKind: FunctionList
    plural: functions
    shortNames:
    - fn
    singular: function
  scope: Namespaced
  versions:
  - name: v1
    schema:
      openAPIV3Schema:
        properties:
          spec:
            properties:
              code:
                type: string
              ram:
                type: string
            type: object
        type: object
    served: true
    storage: true
status:
  acceptedNames:
    kind: Function
    listKind: FunctionList
    plural: functions
    shortNames:
    - fn
    singular: function
  conditions:
  - lastTransitionTime: "2023-05-02T12:12:03Z"
    message: no conflicts found
    reason: NoConflicts
    status: "True"
    type: NamesAccepted
  - lastTransitionTime: "2023-05-02T12:12:03Z"
    message: the initial names have been accepted
    reason: InitialNamesAccepted
    status: "True"
    type: Established
  storedVersions:
  - v1

$ kubectl describe functions.example.com/myfaas
Name:         myfaas
Namespace:    triggermesh
Labels:       <none>
Annotations:  <none>
API Version:  example.com/v1
Kind:         Function
Metadata:
  Creation Timestamp:  2023-05-02T12:13:07Z
  Generation:          1
  Resource Version:    2494494012
  UID:                 bed83736-6c40-4039-97fb-2730c7a4447a
Spec:
  Code:  http://src.example.com/myfaas.js
  Ram:   100Mi
Events:  <none>

```

要发现 CRD，只需访问 API 服务器。例如，使用`kubectl proxy`，您可以在本地访问 API 服务器并查询键空间（在我们的情况下是`example.com/v1`）：

```
$ curl 127.0.0.1:8001/apis/example.com/v1/ | jq .
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "example.com/v1",
  "resources": [
    {
      "name": "functions",
      "singularName": "function",
      "namespaced": true,
      "kind": "Function",
      "verbs": [
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "create",
        "update",
        "watch"
      ],
      "shortNames": [
        "fn"
      ],
      "storageVersionHash": "FLWxvcx1j74="
    }
  ]
}

```

在这里，你可以看到资源以及允许的动词。

当您想要摆脱自定义资源实例时，只需删除它：

```
$ kubectl delete functions.example.com/myfaas
function.example.com "myfaas" deleted

```

## 讨论

正如您所见，创建 CRD 非常简单。从最终用户的角度来看，CRD 提供了一致的 API，并且在某种程度上与像 Pod 或 Job 这样的本机资源难以区分。所有通常的命令，如`kubectl get`和`kubectl delete`，都能正常工作。

创建一个 CRD 实际上只是完全扩展 Kubernetes API 所需工作的不到一半。单独来看，CRD 只允许您通过 etcd 中的 API 服务器存储和检索自定义数据。您还需要编写一个[自定义控制器](https://oreil.ly/kYmqw)，该控制器解释用户意图表达的自定义数据，建立一个控制循环，比较当前状态与声明状态，并尝试协调两者。

## 参见

+   在 Kubernetes 文档中的["使用 `CustomResourceDefinitions` 扩展 Kubernetes API"](https://oreil.ly/mz2bH)

+   在 Kubernetes 文档中的["自定义资源"](https://oreil.ly/gp0xn)
