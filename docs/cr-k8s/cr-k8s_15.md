# 15 安装应用程序

本章涵盖

+   审查 Kubernetes 应用程序管理

+   安装典型的 Guestbook 应用程序

+   构建一个适合生产的 Guestbook 应用程序版本

在 Kubernetes 中管理应用程序通常比在裸服务器上部署应用程序要容易得多，因为所有应用程序的配置都可以通过统一的命令行界面完成。尽管如此，当您将成百上千个容器移动到 Kubernetes 环境中时，需要自动化的配置管理量可能难以从统一的角度来处理。ConfigMaps、Secrets、API 服务器凭证以及卷类型的定制只是日常中可能使 Kubernetes 管理变得繁琐的几个方面。

在本章中，我们（终于）将暂时从 Kubernetes 实现的内部细节中退一步，花一点时间来关注应用程序配置和管理的高级方面。我们将从思考应用程序是什么以及我们如何在 Kubernetes 上安装应用程序开始。

## 15.1 在 Kubernetes 中思考应用程序

对于我们的目的，我们将把 Kubernetes 应用程序称为一组需要部署以提供服务的 API 资源。这个典范的例子可能是 Guestbook 应用程序，定义在 [`mng.bz/y4NE`](http://mng.bz/y4NE)。此应用程序包括

+   一个 Redis 主数据库 Pod

+   一个 Redis 从属数据库 Pod

+   一个与我们的 Redis 主数据库通信的前端 Pod

+   为这三个 Pod 提供的服务

应用交付涉及升级、降级、参数化和定制许多不同的 Kubernetes 资源。由于这是一个备受争议的主题，存在许多不同且相互竞争的技术解决方案，我们在此不会尝试为您解决这个整个谜题。相反，我们在本章中包含了大量应用工具，因为您如何部署您的 Pods 的很大一部分与您如何配置它们以及您最初如何安装应用程序有关。我们可以这样在任何 Kubernetes 集群上运行我们的 Guestbook 应用程序：

```
$ kubectl create -f https://github.com/kubernetes/examples/blob/master/
    guestbook/all-in-one/guestbook-all-in-one.yaml
```

从互联网上 curl

我们之前已经提到过这一点，但我们会再次强调：从互联网上 curl YAML 文件可能是一件危险的事情。在我们的案例中，我们直接从 github.com/kubernetes `curl` 我们的 YAML 文件，这是一个由数千名知名且经过审查的 CNCF（云原生计算基金会）组织成员维护的可信仓库。到本章结束时，我们将拥有一个更符合企业级和现实的方式安装相同的 Guestbook 应用程序，所以请耐心等待。

在发出上一条命令后，我们很快就会看到所有我们的 Pods 都启动并运行，我们的三个前端和 Redis 从属 Pods 有多个副本。在最基本层面上，这就是安装 Kubernetes 应用程序的样子：

```
$ kubectl get pods -A
NAMESPACE  NAME                  READY   STATUS             RESTARTS  AGE
default    frontend-6c-7wjx8     1/1     Running            0         3m18s
default    frontend-6c-g7z8z     1/1     Running            0         3m18s
default    frontend-6c-xd5q2     0/1     ContainerCreating  0         3m18s
default    redis-master-f46-l2d  1/1     Running            0         3m18s
default    redis-slave-797-nv9   1/1     Running            0         3m18s
default    redis-slave-797-9qc   1/1     Running            0         3m18s
```

### 15.1.1 应用程序范围影响您应该使用哪些工具

安装 Guestbook 的那一刻起，我们必须问自己几个明显的问题。这些问题与扩展、升级、定制、安全和模块化有关：

+   用户是否将手动调整 Guestbook 网络应用 Pods 的数量？或者我们希望根据负载自动扩展我们的部署？

+   我们是否希望定期升级 Guestbook 应用程序，如果是这样，我们是否希望与其他 Pods 同时升级？如果是这样，我们应该构建一个 Kubernetes Operator 吗？

+   我们是否有明确的离散配置数量（例如，是否有一些我们关心的替代 Redis 配置）？如果是这样，我们应该使用 `ytt`、`kustomize` 或其他工具，这样我们就不需要在保存应用程序设置的新版本时每次都复制和粘贴大量冗余的 YAML 文件。

+   我们的 Redis 数据库是否安全？它需要安全吗？如果是这样，我们应该为更新或编辑应用程序所在的命名空间添加 RBAC 凭证，还是应该安装与它并行的 NetworkPolicy 规则？我们可以在 [`redis.io/topics/security`](https://redis.io/topics/security) 上查看所有规则，并使用 Secrets、ConfigMaps 等实现这些规则。此外，我们可能需要定期轮换这些 Secrets，这将引入对某种定期自动化的需求。（这会暗示需要 Operator。）

+   尽管我们可以在特定的命名空间中部署我们的应用程序，但我们如何能够跟踪应用程序的整体健康状况随时间的变化，并将其作为一个原子单元进行升级？将大量 Kubernetes 资源作为应用程序从大文件部署出去，在多个方面都显得笨拙。其中之一是，一旦它在我们集群中丢失，我们就不清楚我们的应用程序究竟是什么。

## 15.2 微服务应用程序通常需要数千行配置代码

微服务将应用程序的个体功能分解为单独的服务，每个服务都有自己的独特配置。与大型单体应用程序相比，这会带来高昂的成本，因为在大型单体应用程序中，容器的许多通信和安全都是通过内存计算的内禀使用来完成的。回到我们的 Guestbook 应用程序，它有三个微服务和 200 行代码，我们可以看到，为每个我们创建的 API 对象，可能需要 10 到 50 行代码。

一个典型的企业级 Kubernetes 应用程序将涉及更复杂的配置，从 10 到 20 个容器不等，每个容器通常至少有一个服务、一个 ConfigMap 对象以及一些与其相关的 Secrets。对于这样一个应用程序配置中的代码量，一个粗略的估计是数千行 YAML，分布在许多文件中。显然，每次部署新应用程序时复制数千行代码是不切实际的。那么，让我们来看看管理长期运行、现实世界应用程序的 Kubernetes 配置的几种不同解决方案。

不要害怕重新思考您的应用程序

在您深入探索应用程序选项的无穷无尽之前，我们只想提醒您注意一个警告：如果您的安装工具过于复杂，那么您很可能正在掩盖您底层应用程序中的损坏元素。在这些情况下，简化您应用程序的管理方式可能是个明智的选择。作为一个主要例子，很多时候，开发者创建了过多的微服务，或者在一个应用程序中构建了比必要的更多灵活性（通常是因为没有足够的测试来衡量和设置正确的配置默认值）。有时，配置管理的最佳解决方案就是完全消除可配置性。

## 15.3 重新思考我们的 Guestbook 应用在现实世界中的安装

既然我们已经定义了我们的整体问题空间，即管理 Kubernetes 应用程序配置，那么让我们带着这些解决方案重新思考我们的 Guestbook 应用程序：

+   *Yaml 模板化*—我们将使用`ytt`来完成这项工作，但也可以使用`kustomize`或`helm3`等工具来实现这一功能。

+   *部署和升级我们的应用*—在这里我们将使用 Carvel `kapp-controller`项目，但我们也可以构建一个 Kubernetes Operator 来完成这项工作。

为什么不使用`helm`？

我们并不想明确地支持某一工具而反对另一工具：`helm3`、`kustomize`和`ytt`都可以根据需要用来实现相同的目标。我们更喜欢`ytt`，因为它具有模块化和完全可编程的特性（并且它与 Starlark 集成）。但最终，选择一个工具。`helm3`、`kustomize`、`ytt`都是优秀的工具，但还有许多其他优秀的工具可以解决 YAML 过载问题。没有特定的原因说明为什么这些示例不能使用其他技术来实现。至于这一点，`sed`总是可用。

Carvel 工具包（[`carvel.dev`](https://carvel.dev)工具包）有几个不同的模块化工具，可以一起或单独使用来管理我们迄今为止所描述的整个问题空间。实际上，它是 VMware Tanzu 发行版许多功能的基础。

## 15.4 安装 Carvel 工具包

探索如何提高我们的 "guestbook-fu" 的第一步是安装 Carvel 工具包。然后我们可以从命令行执行这些工具中的每一个。以下代码片段显示了安装工具包的命令。从现在开始，我们将使用 `ytt`、`kapp` 和 `kapp-controller` 逐步改进和自动化我们的 Guestbook 应用程序：

```
# on macOS, do this
$ brew tap vmware-tanzu/carvel ; brew install ytt kbld kapp imgpkg kwt vendir
# or on Linux, do this
$ curl -L https://carvel.dev/install.sh | bash
```

我们真的需要所有的 Carvel 吗？

虽然我们在这个章节中不需要所有的 Carvel 工具，但我们仍然会安装它们，因为它们配合得很好。我们建议您自己探索其中的一些工具（例如 `imgpkg` 或 `vendir`）作为额外的练习。Carvel 的每个二进制文件都易于运行，自包含，并且占用系统空间极小。请随意根据您的学习目标自定义此安装。

### 15.4.1 第一部分：将资源模块化到单独的文件中

当我们看到 200 行的 YAML 墙时，首先考虑的可能是将其分解成更小、更易于理解的块。这样做的原因非常明显：

+   当我们没有大量重复字符串时，使用 `grep` 或 `sed` 等工具要容易得多。

+   跟踪谁可能更改了特定于特定功能的特定内容，简化了小文件的版本控制。

+   将新的 Kubernetes 对象添加到我们的文件中最终会变得难以管理，因此模块化将是最终的需求。我们不妨现在就提前做好准备。

我们已经将 Guestbook 的分解工作分成了两个独立的目录。我们将这些目录放在[`mng.bz/M2wm`](http://mng.bz/M2wm)上供您克隆和尝试。请随意以您觉得直观的方式分解文件。

按照本节中的确切分解步骤不是强制性的，因为如果你问 10 个不同的程序员如何分解一个对象，你会得到 100 个不同的答案。然而，在分解时，请确保进行一个重要的修改：每个 Kubernetes 资源都应该有一个*唯一名称*。例如，Redis 主部署不能与 Redis 服务对象同名。例如，在下面的代码中，我们给我们的 Redis 主部署名称添加了 *-dep* 后缀：

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-master-dep
---
```

我们也以同样的方式处理了前端 YAML 文件。以下代码片段显示了添加 *-dep* 后缀后的目录结构：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-dep
spec:
->  carvel-guestbook git:(master) ✗ tree
.
├──  v0
│      └──  original.yaml
└──  v1
 ├── fe-dep.yaml
 ├── fe-svc.yaml
 ├── redis-master-dep.yaml
 ├── redis-master-svc.yaml
 ├── redis-slave-dep.yaml
 └── redis-slave-svc.yaml
```

重命名 Redis 主和前端资源

如果您不打算使用[`mng.bz/M2wm`](http://mng.bz/M2wm)上的文件，并且您正在自己拆分 guestbook YAML，请确保将 Redis 主和前端部署文件的 `metadata.name` 字段重命名为 redis-master-dep 和 frontend-dep（如前述代码片段所示）。这将使我们能够稍后使用 `ytt` 容易地查找和替换 YAML 构造的值。

我们现在可以通过运行 `kubectl create -f v1/` 来测试我们的分解是否与原始应用程序等效。我们将相信你会运行这个命令并确认三个前端和两个后端 Redis Pod 都已启动并正常运行。然后，你可以设置端口转发，在本地通过端口 8080 浏览 Guestbook 应用程序。例如：

```
$ kubectl port-forward service/frontend 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

现在，你可以在应用程序的消息字段中轻松输入几个值，并看到这些值存储在后端的 Redis 数据库中。注意，这些值也会显示在你的 Guestbook 登录页面上。

### 15.4.2 第二部分：使用 ytt 修补我们的应用程序文件

我们有一个具有前端和后端的工作应用程序。现在如果我们决定开始给它增加更多负载会发生什么？我们可能想要给它分配更多的 CPU。为了做到这一点，我们需要修改 fe-dep.yaml 文件以增加 `requests.cpu` 的值。这意味着我们需要编辑一些 YAML：

```
containers:
- name: php-redis
  image: gcr.io/google-samples/gb-frontend:v4
  resources:
    requests:
      cpu: 100m        ❶
      memory: 100Mi
```

❶ 核心的一十分之一对于生产应用来说并不是很多 CPU。

在代码示例中，我们可以轻松地将 `100m` 替换为 `1`，但那样我们只是将一个硬编码的常量替换为另一个。如果我们能对这个值进行参数化会更好。此外，我们可能还想增加 Redis 的 CPU 要求。幸运的是，我们有 `ytt`。

YAML 模板引擎 `ytt` ([`carvel.dev/ytt/`](https://carvel.dev/ytt/)) 允许使用覆盖、修补等模式对 YAML 文件进行不同的自定义。它还通过使用 Starlark 语言来实现文本操作的逻辑决策，支持高级结构。因为我们已经安装了 Carvel 工具包，所以让我们直接深入了解我们如何在第一个 `ytt` 示例中自定义应用程序的 CPU。

YAML 输入，YAML 输出

`ytt` 是一个 YAML 输入，YAML 输出工具，这是一个需要牢记的重要概念。与其他在 Kubernetes 生态系统中来来去去的工具不同，`ytt`（就像 Carvel 框架中的其他工具一样）专注于做一项具体的工作，并且做得非常好。它操作 YAML！它不会为我们安装文件，并且它以任何方式都不特定于 Kubernetes。

对于我们这个 Guestbook 应用程序的第二次（v2）迭代，我们现在将在一个新目录（称为 v2/）中添加一个新文件（称为 ytt-cpu-overlay.yaml）。我们的目标是匹配 php-redis 前端 Web 应用程序中的 `cpu` 段落与 Redis 主数据库 Pod。以下是代码：

```
#@ load("@ytt:overlay", "overlay")
#@overlay/match by=overlay.subset(
   {"metadata":{"name":"frontend-dep"}})   ❶
---
spec:
  template:
    spec:
      containers:
      #@overlay/match by="name"            ❷
      - name: php-redis
        #@overlay/match missing_ok=False
        resources:
          requests:
            cpu: 200m                      ❸
```

❶ 我们 ytt 遮罩识别我们想要匹配的 YAML 片段的名称。

❷ 一旦在容器内部，它将容器替换为 php-redis 名称。

❸ 原始的 CPU 值 100m 现在翻倍为 200m。

类似地，我们也可以为我们的数据库 Pod 做同样的事情。我们可以创建一个新文件，称为 v2/ytt-cpu-overlay-db.yaml，它执行与之前文件相同的功能：

```
#@ load("@ytt:overlay", "overlay")
#@overlay/match by=overlay.subset({"metadata":{"name":"redis-master-dep"}})
---
spec:
  template:
    spec:
      containers:
      #@overlay/match by="name"
      - name: master
        resources:
          requests:
              #@overlay/match missing_ok=True
              cpu: 300m                        ❶
```

❶ 添加一个新的 CPU 值（这次，300m 以区分两者）

我们现在可以调用这个 YAML 的转换。例如：

```
$ tree v2
v2
 ytt-cpu-overlay-db.yml
 ytt-cpu-overlay.yml

$ ytt -f ./v1/  -f v2/ytt-cpu-overlay.yml  -f v2/ytt-cpu-overlay-db.yml
...
            cpu: 200m
...
            cpu: 300m     ❶
$ kubectl delete -f v0/original.yaml
$ ytt -f ./v1/  -f v2/ytt-cpu-overlay.yml \
   | -f v2/ytt-cpu-overlay-db.yml | kubectl create -f -
deployment.apps/frontend-dep created
service/frontend created
deployment.apps/redis-master-dep created
service/redis-master created
deployment.apps/redis-slave created
service/redis-slave created
```

❶ 仅修改文件以为主节点请求更高的 CPU

太好了，我们现在又回到了起点！最初我们只有一个文件，它很容易打包但难以修改。使用 `ytt`，我们取了许多不同的文件，并在它们之上添加了一层定制，这样它们就可以像单个 YAML 资源一样被流式传输到 `kubectl` 命令中。

我们可能会想象，我们的应用程序现在已经准备好投入生产，因为它能够添加和替换我们的开发配置，以使用现实世界的配置。如果你浏览了 [`carvel.dev/ytt/`](https://carvel.dev/ytt/) 的文档，你会看到还可以进行许多进一步的定制：添加数据值、添加全新的 YAML 构造，等等。然而，在我们的情况下，我们将保持现状，并向上移动到查看我们的修补过的 YAML 资源现在如何被捆绑成一个单一的、可执行的应用程序，其状态被作为一等公民来管理。

### 15.4.3 第三部分：将 Guestbook 作为单个应用程序管理和部署

你可能已经对我们在这本书中多次运行 `kubectl delete` 感到厌烦。如果你问过自己为什么这样做，通常是因为我们没有将我们的应用程序与集群中的其他应用程序隔离开来。一个简单的方法是将整个应用程序部署在一个命名空间中，然后可以删除或创建该命名空间。然而，一旦我们开始将多个资源视为单个应用程序，我们就有一系列新的问题想要回答：

+   在给定的命名空间中，我运行了多少个不同的应用程序？

+   给定应用中所有资源的升级是否成功？

+   我将多少种类型的资源关联到了我的应用程序中？

这些问题可以通过结合使用 `kubectl`、`grep` 和一些巧妙的 Bash 聚合来回答。然而，如果你在应用中有成百上千个容器、Secrets、ConfigMaps 等其他资源，这种方法就无法扩展。此外，它也无法扩展到集群中所有应用的全范围，这些应用的数量也可能轻易达到数百或数千。这就是 `kapp` 工具对我们来说变得重要的地方。

那么 `helm` 呢？

`helm` 是 Kubernetes 最早且最成功的应用程序管理解决方案之一。最初，它结合了有状态升级和资源安装的方面，并使用 YAML 模板。Carvel 项目借鉴了 `helm` 的经验，将这些功能中的许多分离成单独的工具。

`helm3` 实际上是一个更模块化的尝试，用于以无状态方式管理应用程序，这与我们将要看到的 `kapp` 工具类似。无论如何，`helm3` 和 Carvel 生态系统有很多重叠，两者都可以用于类似的情况，但它们由不同的观点、哲学方法和社区所引导。我们鼓励你探索两者，特别是如果你觉得 `kapp` 不是一个理想的解决方案。无论如何，通过使用 `kapp` 时跟踪 Guestbook 的演变，你将学会很多关于管理 Kubernetes 应用程序的一般知识，所以继续前进！

使用 `kapp` 工具([`carvel.dev/kapp/`](https://carvel.dev/kapp/)) 非常简单，尤其是现在我们有了使用 `ytt` 定制应用程序的能力。为了尝试一下，让我们最后清理一下我们的 Guestbook 应用程序，以防它还在运行：

```
$ ytt -f ./v1/  -f v2/ytt-cpu-overlay.yml |
    -f v2/ytt-cpu-overlay-db.yml |
    kubectl delete -f -              ❶
```

❶ 删除我们在上一节中创建的资源（以防它们还在）

我们假设你已经安装了 `kapp` 二进制文件。现在，让我们运行相同的 `ytt` 命令来生成我们的应用程序，但使用 `kapp` 而不是 `kubectl` 来安装它。注意，在这个例子中，我们使用 Antrea 作为我们的 CNI 提供者。但你在此时运行的 CNI 并不重要，只要你有一个（注意，由于页面限制，此代码片段中的几个列被省略了）：

```
$ kapp deploy -a guestbook -f <(ytt -f ./v1/
➥ -f v2/ytt-cpu-overlay.yml
➥ -f v2/ytt-cpu-overlay-db.yml)         ❶

Target cluster 'https://127.0.0.1:53756' (nodes: antrea-control-plane, 2+)

Changes

Namespace  Name              Kind        ...  Op      Op st.  Wait to   ...
default    frontend          Service     ...  create  -       reconcile ...
^          frontend-dep      Deployment  ...  create  -       reconcile ...
^          redis-master      Service     ...  create  -       reconcile ...
^          redis-master-dep  Deployment  ...  create  -       reconcile ...
^          redis-slave       Deployment  ...  create  -       reconcile ...
^          redis-slave       Service     ...  create  -       reconcile ...

Op:      6 create, 0 delete, 0 update, 0 noop
Wait to: 6 reconcile, 0 delete, 0 noop

Continue? [yN]:
```

❶ 将 ytt 生成 YAML 的语句推送到 kapp 作为输入

如果你输入 `y`，你会看到 `kapp` 做很多工作，包括注释你的资源，以便它可以后来为你管理它们。它将升级或删除资源，或者通过名称给出你应用程序的整体状态。在我们的例子中，我们称我们的应用程序为 Guestbook，但我们可以称它为任何我们想要的名称。

在输入“是”（通过输入 `y`）后，你现在将看到比 `kubectl` 可用的更多信息。这是因为对于 `kapp` 来说，应用程序实际上是一个一等公民，它想确保你所有的资源都以健康状态出现。你可以想象 `kapp` 可以如何用于 CI/CD 环境来完全自动化应用程序的升级和管理。例如：

```
11:17:40AM: ---- applying 6 changes [0/6 done] ----
11:17:40AM: create deployment/frontend-dep (apps/v1) namespace: default
11:17:40AM: create deployment/redis-master-dep (apps/v1) namespace: default
11:17:40AM: create service/redis-master (v1) namespace: default
11:17:40AM: create service/redis-slave (v1) namespace: default
11:17:40AM: create deployment/redis-slave (apps/v1) namespace: default
11:17:40AM: create service/frontend (v1) namespace: default
11:17:40AM: ---- waiting on 6 changes [0/6 done] ----
11:17:40AM: ok: reconcile service/frontend (v1) namespace: default
11:17:40AM: ok: reconcile service/redis-slave (v1) namespace: default
11:17:40AM: ok: reconcile service/redis-master (v1) namespace: default
11:17:41AM: ongoing: reconcile deployment/frontend-dep (apps/v1)
➥ namespace: default
11:17:41AM:  ^ Waiting for generation 2 to be observed
11:17:41AM:  L ok: waiting on replicaset/frontend-dep-7bf896bf7c (apps/v1)
➥ namespace: default
11:17:41AM:  L ongoing: waiting on pod/frontend-dep-7bf896bf7c-vbn22 (v1)
➥ namespace: default
11:17:41AM:     ^ Pending: ContainerCreating
11:17:41AM:  L ongoing: waiting on pod/frontend-dep-7bf896bf7c-qph5b (v1)
➥ namespace: default
...
11:17:44AM: ---- waiting on 1 changes [5/6 done] ----
11:18:01AM: ok: reconcile deployment/redis-master-dep (apps/v1)
➥ namespace: default
11:18:01AM: ---- applying complete [6/6 done] ----
11:18:01AM: ---- waiting complete [6/6 done] ----
Succeeded
```

我们现在可以回到我们的应用程序，再次确认它仍在运行。使用以下命令进行检查：

```
$ kapp list
Target cluster 'https://127.0.0.1:53756' (nodes: antrea-control-plane, 2+)

Apps in namespace 'default'

Name       Namespaces  Lcs   Lca
guestbook  default     true  12m

Lcs: Last Change Successful
Lca: Last Change Age

1 apps

Succeeded
```

我们还可以使用 `kapp` 来获取关于运行中的应用程序的详细信息。为此，我们使用 `inspect` 命令：

```
$ kapp inspect --app=guestbook
Target cluster 'https://127.0.0.1:53756' (nodes: antrea-control-plane, 2+)

Resources in app 'guestbook'

Name                         Kind           Owner     Conds.  Age
frontend                     Endpoints      cluster   -       12m
frontend                     Service        kapp      -       12m
frontend-dep                 Deployment     kapp      2/2 t   12m
frontend-dep-7bf7c           ReplicaSet     cluster   -       12m
frontend-dep-7bf7c-g7jlt     Pod            cluster   4/4 t   12m
frontend-dep-7bf7c-qph5b     Pod            cluster   4/4 t   12m
frontend-dep-7bf7c-vbn22     Pod            cluster   4/4 t   12m
frontend-sccps               EndpointSlice  cluster   -       12m
redis-master                 Endpoints      cluster   -       12m
redis-master                 Service        kapp      -       12m
redis-master-dep             Deployment     kapp      2/2 t   12m
redis-master-dep-64fcb       ReplicaSet     cluster   -       12m
redis-master-dep-64fcb-t4hjl Pod            cluster   4/4 t   12m
redis-master-zqdvc           EndpointSlice  cluster   -       12m
redis-slave                  Deployment     kapp      2/2 t   12m
redis-slave                  Endpoints      cluster   -       12m
redis-slave                  Service        kapp      -       12m
redis-slave-dffcf            ReplicaSet     cluster   -       12m
redis-slave-dffcf-75vfq      Pod            cluster   4/4 t   12m
redis-slave-dffcf-lwch9      Pod            cluster   4/4 t   12m
redis-slave-vlnkh            EndpointSlice  cluster   -       12m

Rs: Reconcile state
Ri: Reconcile information

21 resources

Succeeded
```

注意到一些由 Kubernetes 在幕后创建的对象，例如 Endpoints 和 EndpointSlices，都包含在这个读取结果中。EndpointSlices 及其作为服务负载均衡目标的可用性对于任何应用程序都能被最终用户使用至关重要。`kapp` 已经为我们捕获了这些信息，包括我们应用程序中所有资源的成功和失败状态，以单一、易于阅读的表格格式。

最后，我们现在可以使用`kapp`通过运行`kapp delete --app=guestbook`轻松且完整地删除我们的应用程序。这将是我们`kapp deploy`操作的逆操作，因此我们不会显示输出，因为此命令的结果主要不言自明。

### 15.4.4 第四部分：构建一个 kapp 操作符以打包和管理我们的应用程序

现在我们已经将整个应用程序打包成一组原子管理的资源，这些资源具有明确的名称，我们实际上已经构建了可能被认为是自定义资源定义（CRD）的东西。`kapp-controller`项目允许我们使用一些自动化优点将任何`kapp`应用程序包装起来。这次最后的探索完成了我们从“来自互联网的一个大块 YAML”到状态化、自动管理应用程序的过渡，我们可以在企业环境中运行这个应用程序以及数百个其他应用程序。它还将温和地向您介绍如何构建 Kubernetes 操作符的概念。

我们首先要做的是使用`kapp`安装`kapp-controller`工具。我们再次从互联网上安装东西，但，就像往常一样，在安装之前请随意检查 YAML。为了您的方便，以下是 YAML：

```
$ kapp deploy -a kc -f https://github.com/vmware-tanzu/
       carvel-kapp-controller/releases/latest/
         download/release.yml                             ❶
$ kapp deploy -a default-ns-rbac -f
     https://raw.githubusercontent.com/vmware-tanzu/
       carvel-kapp-controller/
        develop/examples/rbac/default-ns.yml              ❷
```

❶ 使用 kapp 工具安装 kapp-controller

❷ 安装一个简单的 RBAC 定义

您可能想知道为什么我们需要为`kapp-controller`设置 RBAC 规则。安装 RBAC 定义（default-ns.yml）允许默认命名空间中的`kapp-controller`像任何操作符一样读取和写入 API 对象。操作符是管理应用程序，`kapp-controller` Pod 需要创建、编辑和更新各种 Kubernetes 资源，以便作为我们应用程序的通用操作符执行其工作。

现在`kapp-controller`已经在我们的集群中运行，我们可以使用它来自动化上一节中的`ytt`复杂性，并且我们可以以声明式的方式完成，这完全在 Kubernetes 内部管理。为此，我们需要创建一个`kapp` CR（自定义资源）。`kapp`应用程序的规范描述在[`mng.bz/PWqv`](http://mng.bz/PWqv)。我们关心的特定字段是

+   `git`——定义了我们应用程序源代码的可克隆 Git 仓库

+   `template`——定义了安装应用程序的`ytt`模板所在的位置

我们首先要做的是为我们的原始 Guestbook 应用程序创建一个应用程序规范，该应用程序将作为一个`kapp`控制的应用程序运行。之后，我们将重新添加我们的`ytt`模板：

```
apiVersion: kappctrl.k14s.io/v1alpha1
kind: App
metadata:
  name: guestbook
  namespace: default
spec:
  serviceAccountName: default-ns-sa    ❶
  fetch:                               ❷
  - git:
      url: https://github.com/jayunit100/k8sprototypes
      ref: origin/master

      # We have a directory, named 'app', in the root of our repo.
      # Files describing the app (i.e. pod, service) are in that directory.
      subPath: carvel-guestbook/v1/
  template:                            ❸
  - ytt: {}
  deploy:
  - kapp: {}
```

❶ 使用之前为该应用程序安装创建的服务帐户

❷ 指定我们的应用程序定义的位置

❸ 因为我们的应用程序代码实际上位于 carvel-guestbook/v1/，所以我们需要指定这个子路径。

到目前为止，你心中的灯泡可能已经亮了起来，关于持续交付的想法，这是完全正确的。这个单独的 YAML 声明让我们可以将我们应用程序的整个管理权交给 Kubernetes 和我们的在线 `kapp-controller` 操作符本身。让我们试一试。运行 `kubectl create -f` 命令，使用之前显示的 YAML 片段创建 Guestbook 应用程序，然后执行以下命令：

```
$ kapp list
Target cluster 'https://127.0.0.1:53756' (nodes: antrea-control-plane, 2+)

Apps in namespace 'default'

Name             Namespaces  Lcs   Lca
default-ns-rbac  default     true  14m
guestbook-ctrl   default     true  1m
...
```

我们可以看到，我们的 guestbook-ctrl 应用程序是由 `kapp-controller` 自动为我们创建的。我们还可以再次使用 `kapp` 来检查这个应用程序：

```
$ kapp inspect --app=guestbook-ctrl
Target cluster 'https://127.0.0.1:53756'
  (nodes: antrea-control-plane, 2+)

Resources in app 'guestbook-ctrl'

Namespace  Name Kind      Owner    Conds.  Rs
default    fe   Dep       kapp     2/2 t   ok
^          fe   Endpoints cluster  -       ok
^          fe   Service   kapp     -       ok
...
```

我们现在已经将我们的应用程序集成到一个 CI/CD 系统中，该系统可以完全在 Kubernetes 内部管理。太棒了！现在可以想象构建任意复杂的系统，让开发者提交和维护他们应用程序的 CRDs，这些 CRDs 最终由运行在我们默认命名空间中的单个 `kapp-controller` 操作符部署和管理。

如果我们想的话，我们可以在新的命名空间中重新部署这个相同的应用程序（通常称为“App CR”）。为此，我们只需运行 `kubectl get apps` 命令来添加或删除这些内容，因为 `kapp-controller` Pod 已经为我们集群中的 `kapp` 应用程序安装了一个 CRD：

```
$ kubectl get apps
NAME        DESCRIPTION           SINCE-DEPLOY   AGE
guestbook   Reconcile succeeded   5m16s          6m8s
```

我们刚刚实现了 Guestbook 应用程序的完整 Operator 部署。现在，让我们尝试将我们的 `ytt` 模板重新添加进来。在这个例子中，我们将之前示例中的 `ytt` 输出推送到 k8sprototypes 仓库中的特定目录（你可能想为这个练习使用你自己的 GitHub 仓库，但这不是必需的）：

```
apiVersion: kappctrl.k14s.io/v1alpha1
kind: App
metadata:
  name: guestbook
  namespace: default
spec:
  serviceAccountName: default-ns-sa
  fetch:
  - git:
      url: https://github.com/jayunit100/k8sprototypes
      ref: origin/master
      subPath: carvel-guestbook/v2/output/
  template:
  - ytt: {}
  deploy:
  - kapp: {}
```

我们现在可以通过简单地编写转换后的 `ytt` 模板到另一个目录来为我们的 Guestbook 应用程序创建一个新的定义，该定义包括我们的 `ytt` 模板。使用 Operator 来管理我们的应用程序的另一个优点是，我们可以创建和删除它们而无需任何特殊工具。这是因为 `kubectl` 客户端将它们视为 API 资源。要删除 Guestbook 应用程序，请运行此命令：

```
$ kubectl delete app guestbook
```

我们现在可以使用 `kubectl` 声明性地删除我们的 Guestbook 应用程序，`kapp-controller` 将完成剩余的工作。我们还可以使用 `kubectl describe` 等命令来查看应用程序的状态。

我们只是触及了 Operator 模型在管理和创建应用程序定义方面的灵活性。作为后续练习，值得探索

+   使用 `kapp-controller` 在许多命名空间中部署相同应用程序的多个副本

+   在 `kapp-controller` 中使用 `ytt` 指令

+   使用 `kapp-controller` 的能力部署和管理 Helm 图表作为应用程序

+   将机密嵌入到你的 `kapp` 应用程序中，以便你可以安全地部署 CI/CD 工作流

这总结了我们对 Guestbook 应用程序的迭代改进。我们将通过查看我们熟悉的老朋友 Calico 和 Antrea CNI 提供者来结束这一章，看看它们如何实现具有细粒度 CRDs 的完整 Kubernetes Operator。

## 15.5 重新审视 Kubernetes Operator

`kapp` 和 `kapp-controller` 工具为我们提供了一种自动的、原子化的方式来以有状态的方式处理 Guestbook 应用程序中的所有服务。因此，这以有机的方式向我们介绍了 Operator 的概念。对于许多应用程序，使用内置工具如 `kapp-controller` 或类似 `Helm` ([`helm.sh`](https://helm.sh)) 可以节省你构建完整的 Kubernetes CRD 和 Operator 实现的时间和复杂性。然而，CRDs 在现代 Kubernetes 生态系统中无处不在，如果我们不至少稍微探索一下它们，那将是对你的一种不公。

Operator 工厂

如果你确实认为你的应用程序足够先进，需要自己的细粒度 Kubernetes API 扩展，那么你将想要构建一个 Operator。构建 Operator 的过程通常涉及为自定义 Kubernetes CRDs 自动生成 API 客户端，并确保这些客户端在资源创建、销毁或编辑时执行“正确”的操作。网上有许多工具，例如 [`github.com/kubernetes-sigs/kubebuilder`](https://github.com/kubernetes-sigs/kubebuilder) 项目，可以轻松构建完整的 Operator。

让我们启动一个 `kind` 集群，使用两个不同的 CNI 提供者（Calico 和 Antrea），以此作为深入了解我们如何使用 CRD 的方式。在这种情况下，因为我们还可能想要向我们的集群添加 NetworkPolicy 对象，所以让我们创建一个基于 Calico 的集群。我们可以使用 `kind-local-up.sh` 脚本来完成这项工作：

```
# make sure you've already installed kind and kubectl before running this...
$ git clone \
   https://github.com/jayunit100/k8sprototypes.git
$ cd kind ; ./kind-local-up.sh
```

Kubernetes 原生应用程序通常为它们的应用程序创建大量的 CRDs。CRDs 允许任何应用程序使用 Kubernetes API 服务器来存储配置数据，并启用 Operators 的创建（我们将在本章后面详细探讨）。Operators 是监视 API 服务器更改并随后运行 Kubernetes 管理任务的 Kubernetes 控制器。例如，如果我们查看我们新创建的 `kind` 集群，我们可以看到几个 Calico CRDs，它们为作为 CNI 提供者的 Calico 提供了特定的配置：

```
$ kubectl get crd                        ❶
NAME
bgpconfigurations.crd.projectcalico.org
bgppeers.crd.projectcalico.org
blockaffinities.crd.projectcalico.org
clusterinformations.crd.projectcalico.org
felixconfigurations.crd.projectcalico.org
globalnetworkpolicies.crd.projectcalico.org
globalnetworksets.crd.projectcalico.org
hostendpoints.crd.projectcalico.org
ipamblocks.crd.projectcalico.org
ipamconfigs.crd.projectcalico.org
ipamhandles.crd.projectcalico.org
ippools.crd.projectcalico.org
kubecontrollersconfigurations.crd.projectcalico.org
networkpolicies.crd.projectcalico.org
networksets.crd.projectcalico.org
$ kubectl get kubecontrollersconfigurations -o yaml
apiVersion: v1
items:
- apiVersion: crd.projectcalico.org/v1
  kind: KubeControllersConfiguration
  ...
  spec:
    controllers:
      namespace:
        reconcilerPeriod: 5m0s
      node:
        reconcilerPeriod: 5m0s
        syncLabels: Enabled
      policy:
        reconcilerPeriod: 5m0s
      serviceAccount:
        reconcilerPeriod: 5m0s
      workloadEndpoint:
        reconcilerPeriod: 5m0s
    etcdV3CompactionPeriod: 10m0s
    healthChecks: Enabled                ❷
    logSeverityScreen: Info
    prometheusMetricsPort: 9094          ❸
  status:
    environmentVars:
      DATASTORE_TYPE: kubernetes
      ENABLED_CONTROLLERS: node
    runningConfig:
      controllers:
        node:
          hostEndpoint:
            autoCreate: Disabled
          syncLabels: Disabled
      etcdV3CompactionPeriod: 10m0s
      healthChecks: Enabled
      logSeverityScreen: Info
```

❶ 列出我们集群中的所有 CRDs

❷ 如果我们认为不需要，我们可以禁用 healthChecks。

❸ 设置 Calico kube controller 提供 Prometheus 指标所使用的端口。这里是我们可以在需要时更改端口的地方。

有趣的是，Calico 将其配置存储在我们的 Kubernetes 集群中，作为一个具有自己类型的自定义对象。实际上，Antrea 也做了类似的事情。我们可以通过再次运行 `kind-local-up.sh` 脚本来检查 Antrea 集群的内部内容，如下所示：

```
$ kind delete cluster --name=kcalico              ❶
$ C=antrea CONFIG=conf.yaml ./kind-local-up.sh )  ❷
```

❶ 删除我们之前的集群

❷ 使用 Antrea 作为 CNI 提供者创建一个新的集群

几分钟后，我们可以查看 Antrea 使用的一些配置对象，就像我们之前对 Calico 所做的那样。以下代码片段显示了生成此输出的命令：

```
$ kubectl get crd
NAME
antreaagentinfos.clusterinformation.antrea.tanzu.vmwar
antreacontrollerinfos.clusterinformation.antrea.tanzu.
clusternetworkpolicies.security.antrea.tanzu.vmware.co
externalentities.core.antrea.tanzu.vmware.com
networkpolicies.security.antrea.tanzu.vmware.com
tiers.security.antrea.tanzu.vmware.com
traceflows.ops.antrea.tanzu.vmware.com
```

自定义 NetworkPolicys 对象类型：这是供应商真正喜欢 CRD 的一个例子

如果我们查看 Calico 和 Antrea CRDs，我们可以看到它们有一些共同点，其中之一就是网络策略。Kubernetes 中的 NetworkPolicy API 在使用某些 CNIs 时不支持所有可能的网络策略。例如，PortRange 策略（仅在 Kubernetes v1.21 中添加）在 Calico 和 Antrea 中都是一段时间的供应商特定策略。然而，由于 Calico 和 Antrea 都有自己的网络策略自定义资源，因此用户可以创建这些特定 CNIs 可以理解的较新的 NetworkPolicy 对象。CRDs 提供了一种优雅的方式来区分产品，而无需为管理这些产品创建供应商特定的工具。例如，您可以使用 `kubectl edit` 指令编辑 k8s.io NetworkPolicy 对象，就像您可以编辑任何 CRD 一样。

如果您想了解更多关于扩展 Kubernetes 网络安全能力的特定网络策略，您可能会对 [`mng.bz/aD9Y`](http://mng.bz/aD9Y) 或 [`mng.bz/g4mn`](http://mng.bz/g4mn) 感兴趣。当然，如果您还没有学习关于基本 Kubernetes NetworkPolicy API 的知识，您可能首先需要研究这些内容，请访问 [`mng.bz/enBZ`](http://mng.bz/enBZ)。

注意，为 Calico 或 Antrea 创建、编辑或删除 NetworkPolicy 对象会导致立即创建防火墙规则。然而，编辑这些应用程序的其他 CRDs 可能不会立即更改它们的配置，并且这些更改可能直到您重启相应的 Calico 或 Antrea Pods 才会实现。因此，尽管 CRDs 给您提供了扩展 Kubernetes API 服务器的方式，但它们并不保证您的新 API 构造将如何实现。

我们之前作为 CNI 提供者安装的 Calico 是通过 YAML 文件部署的，它关联着几个配置对象。或者，我们也可以使用 Tigera 的 `operator` 工具（[`github.com/tigera/operator`](https://github.com/tigera/operator)）来部署它，这个工具会为我们处理 Calico YAML 清单的升级和创建。作为一个实时配置选项，我们还可以安装 `calicoctl` 工具，它也可以为我们配置其某些方面。

类似地，我们的 Antrea 安装也是使用 YAML 清单完成的（正如我们在之前的 CNI 章节中详细讨论的那样）。就像 Calico 一样，一个 Antrea 集群涉及到创建几个配置组件，这些组件存在于我们的集群内部（见 [`mng.bz/J1qa`](http://mng.bz/J1qa)）。

我们现在已经探索了 Kubernetes 应用程序管理的许多方面。这个领域的新工具持续发布，因此这可以被视为您探索如何在生产中扩展和管理大量应用程序的开始。对于许多新来者来说，将 `ytt` 添加到简单的应用程序部署工作流程中可能已经足够他们启动 Kubernetes 应用程序自动化。

## 15.6 Tanzu Community Edition：Carvel 工具包的端到端示例

Tanzu Community Edition（TCE）是了解 Cluster API 和 Image Builder 项目的好方法。TCE 大量使用 Carvel 来处理极其复杂的集群配置配置文件，以及管理需要由最终用户升级和修改的微服务集群。其逻辑核心的大部分都是围绕 Carvel 家族工具构建的。

如果你想了解`kapp`、`imgpkg`、`ytt`以及 Carvel 堆栈中的其他工具如何在现实世界中使用，请查看[`github.com/vmware-tanzu/tce`](https://github.com/vmware-tanzu/tce)和[`github.com/vmware-tanzu/tanzu-framework`](https://github.com/vmware-tanzu/tanzu-framework)。这两个存储库构成了整个 VMware Tanzu Kubernetes 分布的开放源代码安装工具包。在这个分布中

+   `ytt`使用 Kubernetes Cluster API 规范安装和定义复杂的集群模板。

    例如，`ytt`用 Linux 集群规范替换了 Windows 集群规范文件（这些文件会手动将 Antrea 代理作为 Windows 进程安装）。随着时间的推移，这可以修改为使用其他 Cluster API 概念，但截至本文撰写时，你可以在[`mng.bz/p2Z0`](http://mng.bz/p2Z0)上看到这些示例的实际应用。`ytt`将这些目录中的各种文件逐个应用，创建一个单一的、庞大的 YAML 文件，该文件定义了整个集群的蓝图。

+   `kapp`和`kapp-controller`协调从 CNI 规范到这些分布中使用的各种附加应用的一切。

+   `imgpkg`和`vendir`（我们并未深入探讨）也被用于各种容器打包和发布管理任务。

如果你想了解更多关于 Carvel 工具的信息，你可以加入 Kubernetes Slack 上的#carvel 频道（slack.k8s.io）。在那里，你将找到一个充满活力的提交者社区，他们可以帮助你解决关于这些工具的特定和一般性问题。

Antrea LIVE 节目

作为一个简短的说明，关于 Carvel 工具包各个方面的全面介绍，包括它如何借鉴了 Antrea 的一些概念，可以在 Antrea LIVE 节目中找到。直播的节目可以在[antrea.io/live](https://antrea.io/live)上查看。本书中包括 Prometheus 指标、CNI 提供者等内容，在其他广播中已有涉及。

## 摘要

+   使用`kubectl`和大型 YAML 文件管理 Kubernetes 上的应用是一种简单的方法，但很快就会力不从心。有许多优秀的工具可以帮助你处理 YAML 超载。`ytt`是我们在本章中介绍的一个。

+   Carvel 工具包有几个应用，帮助我们以高级别编排 Kubernetes 应用。

+   你可以使用`ytt`或`kustomize`干净地实现 YAML 文件的定制。

+   `ytt` 可以通过在覆盖文件中添加一个 `overlay/match` 子句来匹配 YAML 文件的任意部分，该子句在读取原始文件之后应用。然后，它在一个简单、预先存在的标准 Kubernetes YAML 文件之上构建。

+   您可以使用像 `kapp` 或 `Helm` 这样的工具将不同 YAML 资源集合视为单个应用程序。

+   如果您想以有状态的方式打包应用程序，但又不想构建 Operator，您可以使用像 `kapp-controller` 这样的工具。`kapp-controller` 会以有状态的方式管理应用程序资源集合，随着时间的推移进行管理。这比构建一个完整的 Operator 简一步，但可以通过几秒钟的时间完成，并且具有许多相同的优势。

+   Operator 可以用来在 Kubernetes 中定义更高级别的 API。这些 API 了解您应用程序的特定生命周期，通常涉及将 Kubernetes 客户端打包到您在集群中运行的容器中。

+   Calico 和 Antrea 都实现了 Kubernetes Operator 模式，用于高度复杂的 Kubernetes API 扩展。这使得您可以通过创建和编辑 Kubernetes 资源来完全管理它们的配置。

+   Carvel 工具包和本书中的许多其他主题在 Antrea 的 YouTube 直播中都有涉及，可以在 [antrea.io/live](https://antrea.io/live) 上观看。
