# 第十三章：开发工作流程

> 冲浪是一个令人惊奇的概念。你用一根小棍子挑战大自然，说着 *我要骑你！*，但很多时候大自然会回答 *不，你做不到！*，把你摔到水底。
> 
> 乔琳·布拉洛克

在本章中，我们将扩展讨论第十二章中的内容，关注整个应用程序生命周期，从本地开发到在 Kubernetes 集群中部署更新，包括数据库迁移这个棘手的主题。我们将介绍一些工具，帮助你开发、测试和部署应用程序，包括 Skaffold 和 Telepresence。我们还将介绍 Knative 和 OpenFaaS，这两个选项可在你的集群上运行“无服务器”架构。我们还将查看更复杂的部署，并使用 Helm 钩子来顺序应用程序的部署。

# 开发工具

在 第十二章 中，我们看了一些工具，帮助你编写、构建和部署你的 Kubernetes 资源清单。这在一定程度上很好，但是当你开发运行在 Kubernetes 中的应用程序时，通常希望能够即时尝试并查看变化，而不必经历完整的构建-推送-部署-更新循环。

## Skaffold

[Skaffold](https://oreil.ly/UTaRj) 是来自 Google 的开源工具，旨在提供快速的本地开发工作流程。它在你本地开发时自动重建你的容器，并将这些更改部署到本地或远程集群。

在你的仓库中的 *skaffold.yaml* 文件中定义你想要的工作流程，并运行 `skaffold` 命令行工具启动流水线。当你在本地目录中的文件上进行更改时，Skaffold 将唤醒，使用更改构建一个新的容器，然后自动为你部署它，省去了往返到容器的时间。

我们在 [演示仓库](https://oreil.ly/LAI8f) 中有一个 Skaffold 示例来展示它的工作原理。按照你操作系统的 Skaffold [安装说明](https://oreil.ly/heDih)，将你的 `kubectl` 指向你的本地开发 Kubernetes 集群，并切换到 *hello-skaffold* 目录查看其工作方式：

```
`cd hello-skaffold`
`skaffold dev`
Listing files to watch...
 - skaffold-demo Generating tags...
 - skaffold-demo -> skaffold-demo:e50c9e7-dirty Checking cache...
 - skaffold-demo: Found Locally Tags used in deployment:
 - skaffold-demo -> skaffold-demo:39f0eb63b0ced173... Starting deploy... Helm release demo not installed. Installing... NAME: demo NAMESPACE: demo STATUS: deployed REVISION: 1 TEST SUITE: None Waiting for deployments to stabilize...
 - demo:deployment/demo is ready. Deployments stabilized in 1.097 second Press Ctrl+C to exit Watching for changes... ...
```

Skaffold 自动使用 Helm 构建和部署演示应用程序，现在正在监视代码或图表的更改。它使用随机生成的 SHA 作为临时占位符容器标签，这是在进行开发时的情况。如果你 `curl localhost:8888`，你应该会看到 `Hello, skaffold!`，因为这是在本示例的 `main.go` 中设置的。保持 Skaffold 运行状态，并更改 *main.go* 文件以说一些其他内容，比如 `Hola, skaffold!`。只要保存更改，Skaffold 将自动重新构建和重新部署应用程序的更新副本：

```
...
Watching for changes...
Tags used in deployment:
 - skaffold-demo -> skaffold-demo:39f0eb63b0ced173...
Starting deploy...
Release "demo" has been upgraded. Happy Helming!
NAME: demo
NAMESPACE: demo
STATUS: deployed
REVISION: 2
TEST SUITE: None
Waiting for deployments to stabilize...
 - demo:deployment/demo is ready.
Deployments stabilized in 1.066 second
Watching for changes...
```

现在，如果您`curl localhost:8888`，您将看到您修改后的代码正在运行。Skaffold 构建了演示容器的新副本，并使用 Helm 发布了一个新版本到您的集群。继续在运行 Skaffold 时进行更改，它将继续自动更新应用程序。当您玩够了，可以使用`Ctrl+C`停止 Skaffold，并自动卸载演示应用程序。

```
Cleaning up...
release "demo" uninstalled
```

您可以使用 Skaffold 进行其他类型的构建，例如 Bazel，或使用您自己的自定义构建脚本。它支持使用[Google Cloud Build](https://cloud.google.com/build)作为远程镜像构建工具，而不是使用您的本地计算机。您还可以将测试套件整合到工作流程中，并在构建和部署过程中自动运行测试。

## Telepresence

[Telepresence](https://www.telepresence.io)与 Skaffold 采取了稍微不同的方法。您不需要本地 Kubernetes 集群；Telepresence Pod 在您的实际集群中作为应用程序的占位符运行。然后拦截和路由应用程序 Pod 的流量，将其转发到在您本地计算机上运行的容器。

这使开发者能够使其本地计算机参与远程集群。随着他们对应用程序代码的更改，这些更改将在实时集群中反映出来，而无需部署新的容器。如果您的应用程序构建时间特别长，这可能是一个值得尝试的吸引人工具。

开发应用程序时，常见的抱怨之一是需要构建一个新的容器来包含新的代码更改，如果重新构建速度慢的话，这可能会让人沮丧。Telepresence 旨在减轻这种沮丧，并加快本地开发过程，同时仍然使用容器和 Kubernetes，这样本地开发和生产部署之间的运行环境不需要完全不同。

## Waypoint

HashiCorp，开发了像 Vault、Terraform 和 Consul 这样流行工具的开发者，已经构建了一个新工具，专注于简化端到端的应用开发体验。[Waypoint](https://www.waypointproject.io)引入了一个声明性的*waypoint.hcl*文件，允许您定义应用的构建、部署、配置和发布步骤，无论其使用的运行时或平台如何。使用像 Waypoint 这样的标准包装器意味着您的 CI/CD 工具和流程可以标准化，即使您的应用程序使用不同的构建和部署步骤。开发者可以检出一个应用程序仓库，运行`waypoint build`—而不必考虑这个特定的应用程序是否使用 Dockerfile 或类似[CloudNative Buildpack](https://buildpacks.io)，并期望得到用于运行应用程序的正确构件。同样，`waypoint deploy`抽象了这个特定应用程序使用 Kubernetes、Helm 或者它可能作为 AWS Lambda 函数运行的细节。为开发者和 CI/CD 工具提供标准的 CLI 体验可以极大地简化不同类型应用程序的构建和部署流程。

## Knative

虽然我们看到的其他工具侧重于加速本地开发循环，[Knative](https://oreil.ly/SXohX)则更为雄心勃勃。它旨在提供一种标准机制，用于将各种工作负载部署到 Kubernetes：不仅限于容器化应用程序，还包括*无服务器*风格的函数。

Knative 与 Kubernetes 和 Istio 集成（参见“Istio”）以提供完整的应用/函数部署平台，包括设置构建流程、自动化部署以及*事件*机制，标准化应用程序使用消息传递和队列系统的方式（例如 Pub/Sub、Kafka 或 RabbitMQ）。

## OpenFaaS

[OpenFaas](https://www.openfaas.com)是另一个项目，可以在 Kubernetes 上运行无服务器函数。它包括用于构建和部署函数到集群的 CLI 工具，并能触发 HTTP 事件、定时任务、MQTT、Kafka 和 RabbitMQ 消息以及其他流行的事件驱动系统和消息队列。这些无服务器架构与微服务思维方式结合得很好，并且可以更有效地利用服务器资源。

来自 OpenFaaS 开发团队的一个有趣的旁支项目被称为[*arkade*](https://oreil.ly/XBbiD)，它类似于一个应用商店或市场，用于安装流行的云原生工具（例如 OpenFaaS 本身）的统一工具。通过一个`arkade`命令，您可以在您的 Kubernetes 集群上安装 OpenFaaS 并开始部署您的无服务器工作负载。

## Crossplane

[Crossplane](https://crossplane.io) 是一个旨在将 Kubernetes 的标准化引入非 Kubernetes 云资源的工具。许多应用程序需要在 Kubernetes 之外生活的基础设施组件，例如 AWS S3 存储桶、RDS 数据库实例或 Azure Cosmos DB 实例。通常，这意味着开发人员最终需要使用像[Terraform](https://www.terraform.io)这样的工具先部署这些外部依赖，然后再切换到使用 Dockerfile、Kubernetes 清单和 Helm 图表部署应用程序。

Crossplane 使用 CRD（参见“Operators and Custom Resource Definitions (CRDs)”）来定义那些生活在 Kubernetes 之外的基础设施类型，使它们就像本地的 Kubernetes 对象一样。它使用云供应商的 API 实际创建和部署这些云资源，就像您使用 Terraform CLI 或您的云提供商的 CLI 和 SDK 工具一样。想象一下，您可以像这样声明一个您的应用程序需要运行的 PostgreSQL RDS 实例：

```
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xpostgresqlinstances.aws.database.example.org
  labels:
    provider: aws
    vpc: default
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  compositeTypeRef:
    apiVersion: database.example.org/v1alpha1
    kind: XPostgreSQLInstance
  resources:
    - name: rdsinstance
      base:
        apiVersion: database.aws.crossplane.io/v1beta1
        kind: RDSInstance
        spec:
          forProvider:
            region: us-east-1
            dbInstanceClass: db.t2.small
            masterUsername: postgres-admin
            engine: postgres
            engineVersion: "12"
            skipFinalSnapshotBeforeDeletion: true
            publiclyAccessible: false
          writeConnectionSecretToRef:
            namespace: crossplane-system
...
```

现在，您可以将应用程序的 RDS 实例清单与其他 Kubernetes 清单一起打包并一起部署。然后，Kubernetes 协调循环将确保此 RDS 实例被配置，就像为您的应用程序 Pods 做的那样。另一个好处是，当基础架构与应用程序一起管理和部署时，您可以轻松地在它们之间传递配置。例如，Crossplane CRD 可以在创建数据库时自动为密码创建一个 Kubernetes Secret，并将其传递给您的应用程序以供连接使用。我们预计随着 Kubernetes 的广泛采用，这种扩展 Kubernetes 管理其他类型基础设施的方式将变得流行起来。

# 部署策略

如果您手动升级运行中的应用程序（没有使用 Kubernetes），一种方法是只需关闭应用程序，安装新版本，然后重新启动。但这意味着会有停机时间。

如果您的应用程序在多个服务器上分布有多个实例，一个更好的方式是依次升级每个实例，这样就不会中断服务：所谓的*零停机部署*。

并非所有应用都需要零停机时间。例如，消费消息队列的内部服务是幂等的，因此它们可以一次性进行升级。这意味着升级速度更快，但对于用户界面的应用程序来说，我们通常更关心避免停机时间，而不是追求快速升级。

在 Kubernetes 中，您可以选择最合适的策略。`RollingUpdate`是逐个 Pod 进行的零停机时间选项，而`Recreate`是快速一次性升级所有 Pod 的选项。此外，还有一些字段可以调整，以获得您应用程序所需的确切行为。

在 Kubernetes 中，应用程序的部署策略在部署清单中定义。默认为`RollingUpdate`，如果不指定策略，就会使用这个。要将策略更改为`Recreate`，请像这样设置：

```
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 1
  strategy:
    type: Recreate
```

现在让我们更仔细地看看这些部署策略，看看它们是如何工作的。

## 滚动更新

在滚动更新中，Pod 会逐个升级，直到所有副本都被新版本替换。

例如，假设我们运行一个具有三个副本的应用程序，每个副本都运行`v1`。开发人员使用`kubectl apply...`或`helm upgrade...`命令启动了升级到`v2`的过程。会发生什么？

首先，三个`v1` Pod 中的一个将被终止。Kubernetes 将标记它为未就绪，并停止向其发送流量。将会启动一个新的`v2` Pod 来替代它。与此同时，剩余的`v1` Pod 继续接收传入的请求。在等待第一个`v2` Pod 变为就绪时，我们只剩下两个 Pod，但仍在为用户提供服务。

当`v2` Pod 准备就绪时，Kubernetes 将开始向其发送用户流量，同时还会发送到其他两个`v1` Pod。现在我们又回到了三个 Pod 的完整组合。

这个过程会一直进行，逐个 Pod 地，直到所有的`v1` Pod 都被替换成了`v2` Pod。尽管有时候会出现比平时更少的 Pod 用于处理流量，但整个应用实际上从未真正下线。这就是零停机部署。

###### 注意

在滚动更新期间，旧版本和新版本的应用将同时处于服务状态。虽然通常这不会成为问题，但你可能需要采取措施来确保安全；例如，如果你的更改涉及数据库迁移（见“使用 Helm 处理迁移”），普通的滚动更新可能就不可行了。

如果你的 Pod 有时候在就绪状态后短时间内可能会崩溃或失败，可以使用`minReadySeconds`字段让滚动更新等待每个 Pod 稳定（参见`minReadySeconds`）。

## 重新创建

在`Recreate`模式下，所有运行中的副本会同时被终止，然后重新创建新的副本。

对于不直接处理请求的应用程序来说，这是可以接受的。`Recreate`的一个优点是它避免了同时运行两个不同版本的应用程序的情况（参见“滚动更新”）。

## maxSurge 和 maxUnavailable

随着滚动更新的进行，有时候会有比名义上的`replicas`值更多或更少的 Pod 在运行。两个重要的设置管理这种行为：`maxSurge`和`maxUnavailable`：

+   `maxSurge`设置了额外 Pod 的最大数量。例如，如果你有 10 个副本并将`maxSurge`设置为 30%，那么任何时候运行的 Pod 数量不会超过 13 个。

+   `maxUnavailable`设置了不可用 Pod 的最大数量。对于名义上有 10 个副本并且`maxUnavailable`设置为 20%的 Kubernetes，不会让可用 Pod 的数量少于 8 个。

这些可以设置为整数或百分比：

```
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 20%
      maxUnavailable: 3
```

通常，对于这两个设置，默认值`25%`是可以接受的，您可能不需要调整它们，但根据负载要求，您可能需要对其进行微调，以确保应用程序在升级期间能够保持可接受的容量。在非常大的规模上，您可能会发现在部署期间以`75%`的可用性运行不足以处理您的传入流量，因此您需要稍微减少`maxUnavailable`。

`maxSurge`的值越大，您的发布速度越快，但这会增加您集群资源的负载。较大的`maxUnavailable`值也加快了发布速度，但会减少应用程序的容量。

另一方面，较小的`maxSurge`和`maxUnavailable`值减少了对您集群和用户的影响，但可能会使您的发布过程时间更长。只有您可以决定对您的应用程序来说，正确的权衡是什么。

## 蓝/绿部署

在*蓝/绿*部署中，与逐个杀死和替换 Pods 不同，会创建一个全新的 Deployment，并启动一个新的运行`v2`的 Pods 堆栈，放置在现有的`v1` Deployment 旁边。

其中一个优点是您无需同时处理旧版本和新版本的应用程序处理请求。另一方面，您的集群需要足够大，以运行所需应用程序副本的两倍，这可能很昂贵，并且大部分时间都会有很多未使用的容量（除非根据需要调整集群的规模）。

我们在“服务资源”中了解到，Kubernetes 使用标签来决定哪些 Pods 应该从服务接收流量。实施蓝/绿部署的一种方法是在旧 Pods 和新 Pods 上设置不同的标签（见“标签”）。

通过对示例应用程序的服务定义进行小调整，您可以仅将流量发送到标记为`deployment: blue`的 Pods：

```
apiVersion: v1
kind: Service
metadata:
  name: demo
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: demo
    `deployment``:` `blue`
  type: ClusterIP
```

在部署新版本时，您可以将其标记为`deployment: green`。即使完全运行，它也不会接收任何流量，因为 Service 仅将流量发送到`blue` Pods。您可以对其进行测试，并确保它准备就绪，然后再进行切换。

要切换到新的`Deployment`，编辑`Service`以更改选择器为`deployment: green`。现在新的`green` Pods 将开始接收流量，一旦所有旧的`blue` Pods 处于空闲状态，您可以关闭它们。

## 彩虹部署

在一些罕见情况下，特别是当 Pods 具有非常长寿的连接（例如 websockets）时，仅仅使用蓝色和绿色可能不够。您可能需要同时维护三个或更多版本的应用程序。

有时这被称为*彩虹部署*。每次部署更新时，您都会获得一组新的颜色 Pods 集合。随着连接最终从您最老的 Pods 集合中排干，它们可以关闭。

Brandon Dimcheff 在详细描述了一个 [彩虹部署示例](https://oreil.ly/fuFoi)。

## 金丝雀部署

蓝/绿（或彩虹）部署的优点在于，如果你决定不喜欢新版本或者它的行为不正确，你可以简单地切换回仍在运行的旧版本。然而，这种方法很昂贵，因为你需要集群能力同时运行两个版本，这意味着你可能需要运行更多的节点。

避免这个问题的一种替代方法是金丝雀部署。像煤矿中的金丝雀一样，一小部分新的 Pods 暴露在生产环境中，观察它们的反应。如果它们能够生存下来，升级过程就可以继续进行。如果出现问题，影响范围则严格限制。

就像蓝/绿部署一样，你可以使用标签来做到这一点（参见 “标签”）。在 Kubernetes 文档中有一个详细的金丝雀部署示例，可以参考 [文档](https://oreil.ly/PqUe0)。

更复杂的方法是使用服务网格，例如 Istio、Linkerd 或我们在 “服务网格” 中提到的其他工具之一，它们允许你随机路由变量比例的流量到一个或多个服务版本。这也使得像 A/B 测试之类的操作变得更加容易。

# 使用 Helm 处理迁移

无状态应用容易部署和升级，但涉及到数据库时情况可能更复杂。通常情况下，对数据库模式的更改需要在新应用版本运行之前运行迁移任务。例如，在 Rails 应用中，你需要在启动新的应用 Pods 之前运行 `rails db:migrate`。

在 Kubernetes 中，你可以使用 Job 资源来执行此操作（参见 “Jobs”）。另一个选择是使用 `initContainer`（参见 “Init Containers”）。你也可以在部署过程中使用 `kubectl` 命令来编写脚本，或者如果你使用 Helm，那么你可以使用一个名为 *hooks* 的内置功能。

## Helm 钩子

Helm 钩子允许你控制部署过程中发生事情的顺序。它们还允许你在升级过程中遇到问题时中止操作。

这里是一个使用 Helm 部署的 Rails 应用程序数据库迁移 Job 示例：

```
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Values.appName }}-db-migrate
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  activeDeadlineSeconds: 60
  template:
      name: {{ .Values.appName }}-db-migrate
    spec:
      restartPolicy: Never
      containers:
      - name: {{ .Values.appName }}-migration-job
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        command:
          - rails
          - db:migrate
```

`helm.sh/hook` 属性在 `annotations` 部分中定义：

```
annotations:
  "helm.sh/hook": pre-upgrade
  "helm.sh/hook-delete-policy": hook-succeeded
```

`pre-upgrade` 设置告诉 Helm 在执行升级之前应用此 Job 清单。该 Job 将运行标准的 Rails 迁移命令。

`"helm.sh/hook-delete-policy": hook-succeeded` 告诉 Helm 如果任务成功完成（即退出状态为 0），则删除该 Job。

## 处理失败的钩子

如果 Job 返回非零退出代码，这表明出现错误，迁移过程未能成功完成。Helm 将保留 Job 处于失败状态，以便你可以调试出错原因。

如果发生这种情况，发布过程将停止，应用程序将不会升级。运行 `kubectl get pods` 将显示失败的 Pod，允许您检查日志并查看发生了什么。

问题解决后，您可以删除失败的作业（`kubectl delete job <job-name>`），然后再次尝试升级。另一个自动清理失败作业的选项是添加我们在 “清理已完成的作业” 中介绍的 `ttlSecondsAfterFinished` 字段。

## 其他钩子

钩子除了 `pre-upgrade` 之外还有其他阶段。您可以在发布的以下任何阶段使用钩子：

+   `pre-install` 在渲染模板后，但在创建任何资源之前执行。

+   `post-install` 在所有资源加载后执行。

+   `pre-delete` 在删除请求前执行，而不是删除任何资源。

+   `post-delete` 在删除请求后，释放的所有资源都已删除。

+   `pre-upgrade` 在渲染模板后，但在加载任何资源之前执行（例如，在 `kubectl apply` 操作之前）。

+   `post-upgrade` 在所有资源升级后执行。

+   `pre-rollback` 在回滚请求后，但在回滚任何资源之前执行渲染模板。

+   `post-rollback` 在回滚请求后，所有资源已被修改时执行。

## 钩子链

Helm 钩子还具有将它们按特定顺序链在一起的能力，使用 `helm.sh/hook-weight` 属性。这些钩子将按从低到高的顺序运行，因此具有 `hook-weight` 为 0 的作业将在具有 `hook-weight` 为 `1` 的作业之前运行：

```
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ .Values.appName }}-stage-0
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook-delete-policy": hook-succeeded
    "helm.sh/hook-weight": 0
```

您可以在 Helm 的 [文档](https://oreil.ly/NHKvD) 中找到关于钩子的所有信息。

另一种实现更复杂部署情况的方法是使用 Kubernetes 的 `Operators`。这使您能够构建适合您用例的自定义任务集，包括作为部署应用程序新版本一部分需要发生的任何特殊逻辑或自定义步骤。一个常见的使用 `Operator` 的例子是升级数据库，通常需要作为升级到新版本的一部分进行额外的维护或准备任务。

# 总结

如果您必须构建、推送和部署容器映像以测试每个小的代码更改，那么开发 Kubernetes 应用程序可能会很繁琐。像 Skaffold 和 Telepresence 这样的工具可以加快这个循环，加快开发速度。

特别是，与传统服务器相比，使用 Kubernetes 更容易将更改推广到生产环境，前提是您理解基本概念以及如何自定义它们以适应您的应用程序：

+   Kubernetes 中默认的 `RollingUpdate` 部署策略每次升级少量 Pod，等待每个替换 Pod 就绪后再关闭旧的 Pod。

+   滚动更新避免了停机时间，但代价是使部署时间变长。这也意味着在部署期间您的应用的新旧版本将同时运行。

+   您可以调整`maxSurge`和`maxUnavailable`字段来优化滚动更新。取决于您使用的 Kubernetes API 版本，预设值可能适合也可能不适合您的情况。

+   `Recreate`策略会直接清除所有旧的 Pod 并一次性启动新的 Pod。这很快，但会导致停机时间，因此不适合面向用户的应用程序。

+   在蓝/绿部署中，所有新的 Pod 都会启动并准备就绪，但不接收任何用户流量。然后在一次性切换所有流量到新的 Pod 之前，将所有流量转移到新的 Pod 上，然后淘汰旧的 Pod。

+   彩虹部署类似于蓝/绿部署，但同时有多个版本在服务中。

+   您可以通过调整 Pod 的标签并更改前端 Service 上的选择器来在 Kubernetes 中实现蓝/绿和彩虹部署，以将流量引导到适当的 Pod 集合。服务网格工具还增加了将流量分配到应用的不同运行版本的能力。

+   Helm 钩子提供了一种在部署的特定阶段应用某些 Kubernetes 资源（通常是作业）的方法，例如运行数据库迁移。钩子可以定义部署期间应用资源的顺序，并在某些操作失败时导致部署停止。

+   Knative 和 OpenFaaS 允许您在集群上运行“无服务器”函数，使得在可以运行 Kubernetes 的任何地方（几乎任何地方！）部署事件驱动架构变得更加容易。
