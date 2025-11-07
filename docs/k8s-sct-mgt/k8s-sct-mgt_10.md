# 7 Kubernetes-native continuous integration and Secrets

本章涵盖

+   使用持续集成方法对任何更改进行应用程序集成

+   使用 Kubernetes 本地的 Tekton 实现持续集成管道

+   使用 Kubernetes 本地的 CI 管道测试、构建并将 Linux 容器推送到外部注册库

在上一章中，您学习了如何从密钥存储库向容器注入机密，而在前面的章节中，您也了解了如何在应用程序生命周期的不同阶段保持机密的安全。现在，是时候将这些概念结合起来并开始应用它们了。

我们将演示如何使用 Tekton 实现一个 Kubernetes 本地的持续集成管道，以持续和自动地发布应用程序或服务，同时保持机密的安全。在本章中，我们希望快速交付高质量的应用程序，以便更快更好地进入市场，在整个管道中正确管理机密，以确保在开发阶段不会泄露任何机密。

## 7.1 持续集成简介

开发软件不是一项个人任务，而是一项团队任务，许多人共同且并行工作以创建一个应用程序。在过程结束时整合每个开发者的所有工作可能不是最佳策略，因为可能会出现一些问题，包括合并地狱、组件无法正确集成以及工作部分崩溃。最佳的集成策略是尽可能早地尽可能多地集成，这样任何错误都可以快速检测到，并且可以更容易地定位和修复。

持续集成（或 CI）是一组实践，它自动化了来自多个开发者的代码更改到单个存储库的集成。对存储库的提交必须频繁发生（通常每天几次），并且必须触发一个自动化的过程来验证新代码的正确性。

CI 的最终目标是建立一个稳定和自动化的方式来构建、打包和测试应用程序，以便任何源代码的更改都可以快速集成，无需等待数周，并在提交后进行验证。因此，任何集成过程中的中断都会在早期阶段被发现。

对于每个提交，代码应在以下阶段运行：

+   *构建*—代码被编译和打包。输出取决于开发应用程序所使用的平台和语言——在 Java 的情况下，它可以是 JAR 或 WAR 文件，而在 Go 的情况下，它将是一个二进制可执行文件。

+   *测试*—应用程序运行第一轮测试。这些测试不是端到端测试或长时间测试，而是单元测试、一些组件测试以及验证核心业务功能绿色路径的最小子集的端到端测试。

+   *安全检查*—在这个阶段，代码将进行分析以查找漏洞和不良实践，通常使用静态代码分析工具。

+   *发布*—交付工件被发布在工件存储库中。它可以是一个 JAR 文件、Go 可执行文件或 Linux 容器。

图 7.1 总结了持续集成管道中的每个步骤。

![图 7.1 Sotobueno3](img/CH07_F01_Sotobueno3.png)

图 7.1 组成持续集成管道的常见步骤

持续集成的优点包括以下内容：

+   集成错误在早期阶段就被检测到，并且很容易修复，因为原始代码没有对其进行太多更改。

+   应用程序持续集成，因此不需要几周或几个月来集成所有组件。

+   当检测到测试失败时，更容易找到原因（因为只有少量更改），并且在回滚到先前版本的情况下，只有少量功能会丢失。

+   由于应用程序经常集成，总有一个版本准备好部署（或发布）到任何环境（测试、预生产和生产）。

重要 A CI 管道必须提供快速反馈，这意味着它不应超过 10 分钟，因为该管道的主要目标是快速向开发者提供反馈，并尽快通知他们任何集成错误。

在对持续集成进行简要介绍之后，现在是时候以 Kubernetes 原生的方式使用 Tekton 来实现它了。

## 7.2 Tekton

你将使用 Tekton，一个用于创建云原生 CI 管道的 Kubernetes 原生框架，为 Java 应用程序实现一个简单的 CI 管道。此管道将构建应用程序，将其容器化，并将容器推送到容器注册库。

Tekton ([`tekton.dev/`](https://tekton.dev)) 是一个开源的 Kubernetes 原生项目，用于构建持续集成/持续交付 (CI/CD) 管道，它提供了自定义资源定义 (CRDs)，这些定义了您可以在管道中创建和重用的构建块。Tekton CRDs 可以分为两大块：一组代表定义管道的任何元素，另一组代表管道执行。

如果这种元素分离对你来说似乎很困惑，可以考虑编程语言中类和实例的类比：类对象是一个概念的定义，而类的实例是在内存中的实际对象，具有特定的参数，并且可以被实例化多次。开发者类定义和创建两个类实例的过程如图 7.2 所示。

![图 7.2 Sotobueno3](img/CH07_F02_Sotobueno3.png)

图 7.2 类定义与类实例

在安装 Tekton 之前，你需要创建一个 Kubernetes 集群并部署一个 Git 服务器和一个容器注册库。

### 7.2.1 安装先决条件

在终端窗口中运行以下命令以启动一个新的 minikube 实例。

列表 7.1 启动 minikube

```
minikube start -p argo --kubernetes-version='v1.19.0' --vm-driver='virtualbox' --memory=8196                        ①
```

① 在 argo 配置文件下创建一个 Kubernetes 集群

在本章中，您需要一个具有对某些仓库写权限的 Git 仓库。为了避免依赖于外部服务（例如 GitHub 或 GitLab），您需要在 Kubernetes 集群中部署一个 Git 服务器 ([`gitea.io/en-us/`](https://gitea.io/en-us/))。

列表 7.2 部署 Gitea

```
kubectl apply -f https://gist.githubusercontent.com/
➥lordofthejars/1a4822dd16c2dbbafd7250bcb5880ca2/
➥raw/65ecee01462426252d124410ca0cc19afac382c3/
➥gitea-deployment.yaml                          ①
```

① 应用 Gitea 部署脚本

等待 Gitea 部署启动并运行。

列表 7.3 等待 Gitea 准备就绪

```
kubectl wait --for=condition=ready pod -l app=gitea-demo --timeout=90s    ①
```

① 等待 Gitea 部署完成

Git 服务器可以通过 Kubernetes 集群中的 `gitea` DNS 名称访问。使用用户名 `gitea` 和密码 `gitea1234`，以推送章节中使用的源代码的权限注册新用户到系统中。

列表 7.4 创建 Gitea 用户

```
kubectl exec svc/gitea > /dev/null
-- gitea admin create-user --username gitea
➥--password gitea1234 --email gitea@gitea.com
➥--must-change-password=false                   ①
```

① 在 Gitea 容器中执行用户创建操作

最后，使用的源代码已从 GitHub 迁移到内部 Git 服务器。

列表 7.5 将应用程序迁移到 Gitea

```
kubectl exec svc/gitea > /dev/null -- curl -i -X POST -H
➥"Content-Type:application/json" -d '{"clone_addr":
➥"https://github.com/lordofthejars/kubernetes-secrets
➥-source.git","private": false,"repo_name": "kubernetes
➥-secrets-source","uid": 1}'http://gitea:gitea1234@localhost:3000/
➥api/v1/repos/migrate        ①
```

① 在 Gitea 容器中执行 Git 仓库迁移

此外，还需要一个容器注册表来存储 CI 阶段构建的容器。为了避免依赖于外部服务，您需要在 Kubernetes 集群中部署一个容器注册表 ([`docs.docker.com/registry/`](https://docs.docker.com/registry/))。

列表 7.6 安装 Docker 注册表

```
kubectl apply -f https://gist.githubusercontent.com/
➥lordofthejars/d386a28c07a54a6fd8717ce78a652b8b/raw/
➥a03b22afd549f8164dca2e38d6fab4fecfbc318a/
➥registry-deployment.yaml                     ①
```

① 应用 Docker 注册表部署脚本

等待注册表部署启动并运行。

列表 7.7 等待注册表准备就绪

```
kubectl wait --for=condition=ready pod -l
➥app=registry-demo --timeout=90s         ①
```

① 等待 Docker 注册表部署完成

容器镜像由 Kubernetes 节点拉取，这意味着在 Kubernetes 服务中使用的 DNS 名称在物理机器（例如，在本例中的 minikube 节点）中是无效的。为了使推送到注册表的容器可以从节点拉取，您需要向 Kubernetes 节点添加一个包含 DNS 名称和 `registry` 服务 IP 的 `/etc/hosts` 条目。通过运行以下命令获取 `registry` 服务 IP。

列表 7.8 获取注册表 IP

```
kubectl get service/registry -o jsonpath=
➥'{.spec.clusterIP}'      ①

10.111.129.197
```

① 获取服务 IP

然后访问 minikube 机器，并将以下条目添加到 `/etc/hosts`。

列表 7.9 将注册表 IP 注册到主机

```
minikube ssh -p argo

sudo -i
echo "10.111.129.197 registry" >> /etc/hosts    ①

exit
exit
```

① 用正确的值替换 IP。

现在，您已准备好在 Kubernetes 集群中安装 Tekton。

### 7.2.2 安装 Tekton

通过应用列表 7.10 中的代码安装 Tekton 0.20.1。此命令将安装所有基于角色的访问控制 (RBAC)、自定义资源定义 (CRD)、ConfigMaps 和部署以使用 Tekton。

列表 7.10 安装 Tekton

```
kubectl apply --filename https://github.com/tektoncd/
➥pipeline/releases/download/v0.20.1/release.yaml       ①

namespace/tekton-pipelines created
podsecuritypolicy.policy/tekton-pipelines created
clusterrole.rbac.authorization.k8s.io/
➥tekton-pipelines-controller-cluster-access created
clusterrole.rbac.authorization.k8s.io/
➥tekton-pipelines-controller-tenant-access created
clusterrole.rbac.authorization.k8s.io/
➥tekton-pipelines-webhook-cluster-access created
role.rbac.authorization.k8s.io/tekton-pipelines-controller created
role.rbac.authorization.k8s.io/tekton-pipelines-webhook created
role.rbac.authorization.k8s.io/tekton-pipelines-leader-election created
serviceaccount/tekton-pipelines-controller created
serviceaccount/tekton-pipelines-webhook created
clusterrolebinding.rbac.authorization.k8s.io/
➥tekton-pipelines-controller-cluster-access created
clusterrolebinding.rbac.authorization.k8s.io/
➥tekton-pipelines-controller-tenant-access created
clusterrolebinding.rbac.authorization.k8s.io/
➥tekton-pipelines-webhook-cluster-access created
rolebinding.rbac.authorization.k8s.io/tekton-pipelines-controller created
rolebinding.rbac.authorization.k8s.io/tekton-pipelines-webhook created
rolebinding.rbac.authorization.k8s.io/
➥tekton-pipelines-controller-leaderelection created
rolebinding.rbac.authorization.k8s.io/
➥tekton-pipelines-webhook-leaderelection created
customresourcedefinition.apiextensions.k8s.io/
➥clustertasks.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/conditions.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/
➥images.caching.internal.knative.dev created
customresourcedefinition.apiextensions.k8s.io/pipelines.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/
➥pipelineruns.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/
➥pipelineresources.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/runs.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/tasks.tekton.dev created
customresourcedefinition.apiextensions.k8s.io/taskruns.tekton.dev created
secret/webhook-certs created
validatingwebhookconfiguration.admissionregistration.k8s.io/
➥validation.webhook.pipeline.tekton.dev created
mutatingwebhookconfiguration.admissionregistration.k8s.io/
➥webhook.pipeline.tekton.dev created
validatingwebhookconfiguration.admissionregistration.k8s.io/
➥config.webhook.pipeline.tekton.dev created
clusterrole.rbac.authorization.k8s.io/tekton-aggregate-edit created
clusterrole.rbac.authorization.k8s.io/tekton-aggregate-view created
configmap/config-artifact-bucket created
configmap/config-artifact-pvc created
configmap/config-defaults created
configmap/feature-flags created
configmap/config-leader-election created
configmap/config-logging created
configmap/config-observability created
configmap/config-registry-cert created
deployment.apps/tekton-pipelines-controller created
service/tekton-pipelines-controller created
horizontalpodautoscaler.autoscaling/tekton-pipelines-webhook created
poddisruptionbudget.policy/tekton-pipelines-webhook created
deployment.apps/tekton-pipelines-webhook created
service/tekton-pipelines-webhook created
```

① 应用 Tekton 部署脚本

提示：Tekton CLI 是用于与 Tekton 资源交互的命令行实用程序。尽管安装它不是强制性的，但它非常有帮助，尤其是在查看管道中发生的事情时。要安装它，请访问 [`github.com/tektoncd/cli/releases/tag/v0.16.0`](https://github.com/tektoncd/cli/releases/tag/v0.16.0)，下载您平台的包，解压缩它，并将 `tkn` 文件复制到 `PATH` 目录中，以便在任何地方都可以访问。

到目前为止你安装的整体情况如图 7.3 所示。

![图 7.4 Tekton 元素之间的关系](img/CH07_F03_Sotobueno3.png)

图 7.3 集群内部部署的服务（Tekton、SCM 和容器注册库）

到目前为止，你已经准备好开始在 Kubernetes 集群中学习和使用 Tekton 了。

### 7.2.3 Tekton 管道

总结来说，Tekton 提供了两组 Kubernetes 对象来定义和执行管道。第一组是用于定义组成 CI 管道的`tasks`和`steps`的 Kubernetes 对象集合。最重要的对象是`Pipelines`，它由`Tasks`组成，而`Tasks`又由`Steps`组成，如图 7.4 所示。

![图 7.4 Tekton 元素之间的关系](img/CH07_F04_Sotobueno3.png)

图 7.4 Tekton 元素之间的关系

第二组是用于实例化`tasks`和`pipelines`的 Kubernetes 对象集合。最重要的对象是`PipelineRun`、`TaskRun`和`Triggers`。`Triggers`在本书中没有涉及，因为它们超出了本书的范围，但可以简单地说，触发器可以通过外部事件（例如，对源仓库的提交）来执行`pipeline`。

PipelineResource

`PipelineResource` 是一个 Kubernetes 对象，它定义了一组用作`task`输入和输出参数的资源。输入资源的例子包括 Git 仓库和容器镜像。输出资源的例子包括容器镜像和文件。

要设置 Git 仓库的 URL，创建一个`PipelineResource`，将类型设置为`git`并将`url`参数设置为 Git 仓库位置，如下所示。

列表 7.11 build-resources.yaml

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: git-source
spec:
  type: git          ①
  params:
    - name: url      ②
      value: https://github.com/lordofthejars/kubernetes-secrets-source.git
```

① 将 PipelineResource 设置为 git 类型

② Git 资源有 url 配置参数。

新创建的`PipelineResource`命名为 git-source；你稍后会用到它。

Steps

`step` 代表`pipeline`中的操作；例如，一些`steps`包括编译应用程序、运行测试和构建 Linux 容器镜像。每个`step`都在提供的容器镜像内执行，并且任何`step`都可以挂载卷或使用环境变量。

`step` 在`steps`部分定义，其中设置`step`的名称、在`step`中使用的容器镜像以及在该容器内执行的命令。此外，你可以通过使用`workingDir`属性来设置命令运行的目录。以下是一个使用 Apache Maven 构建 Java 应用程序的示例。

列表 7.12 build-app-task.yaml

```
steps:
    - name: maven-build
      image: docker.io/maven:3.6-jdk-11-slim                         ①
      command:                                                       ②
        - mvn
      args:
        - clean
        - package
      workingDir: "/workspace/source/$(inputs.params.contextDir)"    ③
```

① 使用 Maven docker 镜像来构建应用程序。

② 在容器内调用 Maven 命令。

③ 工作目录通过外部输入参数`inputs.params.contextDir`设置。

Tasks

`task` 是一个由一系列按顺序排列的`steps`组成的 Kubernetes 对象。每个`task`都在一个 Kubernetes Pod 中执行，在该 Pod 中运行一个容器。

由于 Pod 内的所有容器共享资源（例如，CPU、磁盘和内存），并且一个`task`由在同一个 Pod 中运行的几个`步骤`（容器）组成，因此一个`步骤`写入磁盘的内容可以在`task`的任何`步骤`内部访问。图 7.5 显示了所有这些元素是如何相互关联的。

![图片](img/CH07_F05_Sotobueno3.png)

图 7.5 Tekton `任务`、Pod 和容器之间的关系

在`spec`部分配置了一个`task`，在那里你设置要执行的`步骤`列表和可选的配置参数，如输入参数、`task`所需的输入和输出资源以及卷。以下列表展示了如何注册上一节中定义的`步骤`，为`workingDir`属性定义输入参数，并在执行任何`步骤`之前定义一个 Git 类型的输入资源来克隆仓库的示例。

列表 7.13 build-app-task.yaml

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-app
spec:
  params:                   ①
    - name: contextDir
      description: the context dir within source
      default: .            ②
  resources:
    inputs:
      - name: source        ③
        type: git
  steps:
    - name: maven-build     ④
    ...
```

① 注册了一个输入参数列表。在这种情况下，contextDir 参数由 maven-build 步骤需要来设置工作目录。

② 如果未在外部设置参数，则设置默认值

③ 使用名称 source 定义了 Git 输入资源。

④ 步骤定义

到达这一点后，你可能会有两个疑问：

1.  你在哪里设置 Git 项目仓库？

1.  项目被克隆在哪里？

为了回答第一个问题，Git 仓库是在一个`PipelineResource`对象中外部配置的。第二个问题更容易回答。内容被克隆到/workspace/<name>目录中，其中`name`是在`git`类型中给出的输入名称值。因此，之前定义的 Git 资源被克隆到/workspace/source 目录中。

一个`Task`只是定义，或者执行`步骤`的描述。要执行它，你需要创建一个`TaskRun`对象。

TaskRun

`TaskRun`是一个 Kubernetes 对象，它在集群上实例化和执行 Tekton `Task`。`TaskRun`按照定义的顺序执行`Task`中定义的每个`步骤`，直到所有步骤都执行完毕。

要执行之前创建的`build-appTask`，你需要一个`TaskRun`对象。此对象将引用`Task`并使用特定值设置输入参数和资源，如下面的列表所示。

列表 7.14 build-app-task-run.yaml

```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-app-run
spec:
  params:
    - name: contextDir         ①
      value: name-generator
  resources:
    inputs:
      - name: source           ②
        resourceRef:
          name: git-source
  taskRef:
    name: build-app            ③
```

① 将 contextDir 参数设置为 name-generator 目录。

② 通过引用之前创建的 git-source PipelineResource 设置源资源。

③ 对 Task 的引用

注意：你可能想知道为什么`contextDir`需要设置为一个特定的值，而不是保留其默认值（仓库的根目录）。这是由于[`github.com/lordofthejars/kubernetes-secrets-source.git`](https://github.com/lordofthejars/kubernetes-secrets-source.git)仓库的组织方式。如果你仔细查看目录层次结构，你会注意到仓库中包含在目录中的服务（名称和欢迎信息）：

```
.
├── greetings
│     ├── README.md
│     ├── mvnw
│     ├── mvnw.cmd
│     ├── pom.xml
│     ├── src
│     └── target
├── name-generator     ①
│     ├── README.md
│     ├── mvnw
│     ├── mvnw.cmd
│     ├── pipelines
│     ├── pom.xml
│     ├── src
│     └── target
└── welcome-message
     ├── README.md
     ├── mvnw
     ├── mvnw.cmd
     ├── pom.xml
     ├── src
     ├── target
     └── vault-init
```

① 服务位置正在建设中

由于你正在构建名称生成器服务，因此将 Maven 的工作目录设置为 `name-generator`。

`TaskRun` 是执行单个任务的方式。有时你可能使用它们来执行或测试特定的任务，但大多数时候，你希望执行包含在其上的所有 `tasks` 的完整 `pipeline`。

Pipeline

`Pipeline` 是一个由一系列 `tasks` 组成的 Kubernetes 对象，这些 `tasks` 以有向无环图的形式连接。在 `Pipeline` 定义中，你可以完全控制 `tasks` 的执行顺序和条件，这使得可以设置并行运行 `tasks` 的扇入/扇出场景，或者设置在执行之前应该满足的条件。

现在创建一个简单的 `Pipeline`，使用上一节中创建的 `build-appTask`。与 `tasks` 一样，`Pipeline` 也可以有输入参数和输入资源，这使得 `pipeline` 可扩展。对于这个特定的例子，只有输入参数（Git 资源）可以从 `pipeline` 外部进行配置，而 `contextDir` 参数的值是在 `task` 中硬编码的。最后，`build-apptask` 被注册为具有输入参数和资源的 `pipeline` `task`。`Pipeline` 定义应类似于以下列表所示。

列表 7.15 pipeline-name-app.yaml

```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: name-generator-deployment
spec:
  resources:
    - name: appSource           ①
      type: git
  tasks:
    - name: build-app
      taskRef:
        name: build-app         ②
      params:                   ③
        - name: contextDir
          value: name-generator
      resources:
        inputs:                 ④
          - name: source
            resource: appSource
```

① 定义一个 Git 类型的输入资源

② 将 build-app 任务注册到当前 pipeline

③ 为任务输入参数设置一个静态值

④ 将 Pipeline 定义中的 Git 资源链接到 Task

到目前为止，你已经看到了如何使用 Tekton 定义 CI 管道，但还没有发生执行，因为管道需要被实例化，并且需要提供输入参数和资源。在下一节中，你将看到如何执行 Tekton `pipeline`。

PipelineRun

`PipelineRun` 是一个 Kubernetes 对象，用于在集群上实例化和执行 Tekton `Pipeline`。`PipelineRun` 执行 `Pipeline` 中定义的每个 `tasks`，并为每个 `tasks` 自动创建一个 `TaskRun`，如下所示。

列表 7.16 pipeline-run-name-app.yaml

```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: build-app-pipeline
spec:
  resources:
    - name: appSource                  ①
      resourceRef:
        name: git-source
  pipelineRef:
    name: name-generator-deployment    ②
```

① 通过引用 git-source PipelineResource 设置 appSource 资源。

② 对 pipeline 的引用

图 7.6 总结了基本的 Tekton 元素以及它们之间的关系。

![图片](img/CH07_F06_Sotobueno3.png)

图 7.6 `PipelineRun` 与 Tekton 资源之间的关系

你现在已经看到了构建基本 CI 管道最重要的 Tekton 资源，但这还远非一个真正的管道。

## 7.3 欢迎消息的持续集成

在 Kubernetes 中，一个真正的 CI 管道至少需要以下步骤：

+   使用 Tekton Git 资源检出代码

+   在 Tekton `步骤` 中定义一个 Apache Maven 容器以构建和测试应用程序

+   将容器注册表的凭证设置为 Kubernetes Secrets，并在 Tekton `步骤` 中定义一个 Buildah 容器以构建和推送容器

本章中使用的应用程序是一个由两个服务组成的简单服务架构，这两个服务生成欢迎信息：

+   *名称生成服务*——一个从名称列表中随机选择名称的服务，如下面的列表所示。

列表 7.17 NameGeneratorResource.java

```
@Path("/generate")
public class NameGeneratorResource {

  private static final String[] NAMES = new String[] {
    "Ada", "Alexandra", "Burr", "Edson", "Kamesh", "Sebi", "Anna", "Gavina"①
  }; 

  private static final Random r = new Random();

  @GET
  @Path("/name")
  @Produces(MediaType.TEXT_PLAIN)
  @RolesAllowed("Subscriber")                                              ②
  public String generate() {
    return NAMES[generateRandomIndex()];                                   ③
  }
}
```

① 名称列表

② 保护该方法，以确保只有具有订阅者角色的用户才能访问

③ 生成随机名称((CO18-3))

+   *欢迎信息服务*——一个从数据库中随机选择欢迎信息并将问候的人的姓名委托给名称服务的服务，如下面的列表所示。

列表 7.18 WelcomeResource.java

```
@Path("/welcome")
public class WelcomeResource {

  @RestClient
  NameService nameService;                        ①

  @ConfigProperty(name = "name-service-token")    ②
  String token;

  private static Random r = new Random();

  @GET
  @Path("/message")
  @Produces(MediaType.TEXT_PLAIN)
  public String hello() {
    String welcomeMessage = randomMessage
    ➥(Welcome.listAll());                        ③
    String name = nameService.generateName
    ➥("Bearer " + token);                        ④

    return welcomeMessage + " " + name;
  }
}
```

① 与名称生成服务通信的接口

② 访问名称生成服务所需的 API 密钥

③ 从数据库中查找所有欢迎信息

④ 通过传递 API 密钥向名称生成服务发送请求

图 7.7 显示了应用程序的概述。

![图片](img/CH07_F07_Sotobueno3.png)

图 7.7 名称服务和欢迎服务之间交互的概述

为了最大化安全性，以下考虑因素被采纳：

+   您需要提供一个 API 密钥来访问名称生成服务。此 API 密钥是秘密的，并存储在 HashiCorp Vault 实例中。

+   欢迎信息服务的数据库凭据由 HashiCorp Vault 动态数据库凭据管理。

+   服务使用 Kubernetes 身份验证方法对 HashiCorp Vault 进行身份验证。

图 7.8 显示了这些元素的概述。

![图片](img/CH07_F08_Sotobueno3.png)

图 7.8 安全元素

我们假设您对 CI/CD 有一些经验，以及 Git 和 Linux 容器的基本知识。本章中描述的原则适用于您最终可能选择的技术。

Tekton 和 Kubernetes 元素的管道执行在图 7.9 中显示。创建了一个包含三个容器的 Pod；第一个容器从 Gitea 服务器克隆项目，项目在第二个容器中打包，服务所在的 Linux 容器在第三个容器中构建并推送到容器注册库。执行命令的三个容器在图 7.9 中显示。

![图片](img/CH07_F09_Sotobueno3.png)

图 7.9 欢迎信息服务 Pod 内运行的容器列表

这些步骤都实现为 Tekton `step`。让我们在以下章节中实现它们。

### 7.3.1 编译和运行测试

您已经在上一节中看到了如何使用 Apache Maven 编译和运行测试。欢迎信息服务是用 Java 开发的，Apache Maven 用作构建工具。

列表 7.19 构建服务 Tekton `step`

```
- name: maven-build
  image: docker.io/maven:3.6.3-jdk-11-slim
  command:
    - mvn
  args:
    - -DskipTests
    - clean
    - install
  workingDir: "/workspace/source/$(inputs.params.contextDir)"
```

### 7.3.2 构建和推送容器镜像

在运行中的容器内构建容器镜像（记住每个 `step` 都是在容器内执行的）有些复杂，因为需要 Docker 守护进程来构建容器镜像。为了避免处理 Docker-inside-Docker 问题，或者在不方便运行 Docker 主机（如 Kubernetes 集群）的环境中构建容器镜像，有一些无 Docker 工具允许在不依赖 Docker 守护进程的情况下构建容器镜像。例如，*Buildah* ([`buildah.io/`](https://buildah.io/)) 是一个在容器内从 `Dockerfile` 构建容器镜像的工具，无需 Docker 守护进程。

在列表 7.20 中显示的 `step` 定义中，使用 Buildah 构建并推送欢迎信息容器到容器注册库。容器名称以 `registry:group:name:tag` 的形式提供，`Dockerfile` 的位置作为参数提供。

列表 7.20 构建并推送容器镜像 Tekton `step`

```
- name: build-and-push-image
  image: quay.io/buildah/stable
  script: |                                                     ①
    #!/usr/bin/env bash
    buildah bud --layers -t $DESTINATION_IMAGE $CONTEXT_DIR     ②
    buildah push --tls-verify=false $DESTINATION_IMAGE docker:
    ➥//$DESTINATION_IMAGE                                      ③
  env:                                                          ④
    - name: DESTINATION_IMAGE
      value: "$(inputs.params.imageName)"
    - name: CONTEXT_DIR
      value: "/workspace/source/$(inputs.params.contextDir)"
  securityContext:                                              ⑤
    runAsUser: 0
    privileged: true
  volumeMounts:                                                 ⑥
    - name: varlibc
      mountPath: /var/lib/containers
```

① 可以在步骤定义中嵌入脚本。

② 构建容器镜像

③ 推送容器镜像

④ 参数设置为环境变量，因此在脚本执行期间可以访问它们。

⑤ 运行 Buildah 需要以 root 用户身份运行并启用权限提升。

⑥ Linux 容器层保存在本地文件系统中的 /var/lib/containers 目录下。

创建一个名为 `welcome-service-task.yaml` 的新文件，包含之前定义的 `steps`，如下所示。

列表 7.21 welcome-service-task.yaml

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: welcome-service-app                                ①
spec:
  params:                                                  ②
    - name: contextDir                                     ③
      description: the context dir within source
      default: .
    - name: imageName                                      ④
      description: the container name url-group-artifact-tag
  resources:
    inputs:
      - name: source
        type: git                                          ⑤
  steps:
    - name: maven-build                                    ⑥
      image: docker.io/maven:3.6.3-jdk-11-slim
      command:
        - mvn
      args:
        - clean
        - install
      workingDir: "/workspace/source/$(inputs.params.contextDir)"

    - name: build-and-push-image                           ⑦
      image: quay.io/buildah/stable
      script: |
        #!/usr/bin/env bash
        buildah bud --layers -t $DESTINATION_IMAGE $CONTEXT_DIR
        buildah push --tls-verify=false $DESTINATION_IMAGE docker:
          ➥//$DESTINATION_IMAGE
      env:
        - name: DESTINATION_IMAGE
          value: "$(inputs.params.imageName)"
        - name: CONTEXT_DIR
          value: "/workspace/source/$(inputs.params.contextDir)"
      securityContext:
        runAsUser: 0
        privileged: true
      volumeMounts:
        - name: varlibc
          mountPath: /var/lib/containers
  volumes:
    - name: varlibc
      emptyDir: {}
```

① 任务名称

② 任务参数

③ 仓库内的源代码相对路径

④ 容器镜像名称

⑤ 从 Git 服务器克隆仓库

⑥ 使用 Maven 打包步骤

⑦ 创建容器镜像步骤

执行以下命令将 `Task` 注册到 Kubernetes 集群。

列表 7.22 注册任务

```
kubectl apply -f welcome-service-task.yaml   ①
```

① 注册 Tekton 任务定义

### 7.3.3 管道资源

欢迎信息服务的仓库存储在 Kubernetes 集群中部署的本地 Git 服务器（Gitea）上。在 `PipelineResource` 中设置服务的 Git 位置。创建一个名为 `welcome-service-resource.yaml` 的新文件，如下所示。

列表 7.23 welcome-service-resource.yaml

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: welcome-service-git-source
spec:
  type: git
  params:
    - name: url
      value: http://gitea:3000/gitea/
      ➥kubernetes-secrets-source.git     ①
```

① 指向内部仓库

执行以下命令将 `PipelineResource` 注册到 Kubernetes 集群。

列表 7.24 注册 `pipeline` 资源

```
kubectl apply -f welcome-service-resource.yaml    ①
```

① 注册 Tekton 管道资源

### 7.3.4 管道

最后一步是定义一个 `pipeline` 来实现欢迎信息服务的 CI 管道。创建一个名为 `welcome-service-pipeline.yaml` 的新文件，如下所示。

列表 7.25 welcome-service-pipeline.yaml

```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: welcome-deployment
spec:
  resources:
    - name: appSource
      type: git
  params:
    - name: imageTag
      type: string
      description: image tag
      default: v1
  tasks:
    - name: welcome-service-app
      taskRef:
        name: welcome-service-app
      params:
        - name: contextDir
          value: welcome-message
        - name: imageName
          value: "registry:5000/k8ssecrets/
          ➥welcome-message:$(params.imageTag)"
      resources:
        inputs:
          - name: source
            resource: appSource
```

执行以下命令将 `Pipeline` 注册到 Kubernetes 集群。

列表 7.26 注册 `Pipeline`

```
kubectl apply -f welcome-service-pipeline.yaml      ①
```

① 注册 Tekton 管道定义

图 7.10 显示了 `Pipeline` 和 `Task` 参数之间的关系。

![图片](img/CH07_F10_Sotobueno3.png)

图 7.10 `Pipeline` 和 `Task` 参数之间的关系

### 7.3.5 PipelineRun

创建一个 `PipelineRun` 来触发之前步骤中定义的 `welcome-deploymentpipeline`。在这个 `PipelineRun` 中，除了设置 Git 仓库位置外，还提供了容器镜像标签。

列表 7.27 welcome-service-pipeline-run.yaml

```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: welcome-pipeline-run
spec:
  params:
    - name: imageTag
      value: "1.0.0"     ①
  resources:
    - name: appSource
      resourceRef:
        name: welcome-service-git-source
  pipelineRef:
    name: welcome-deployment
```

① 将镜像标签设置为版本 1.0.0。

图 7.11 显示了 `PipelineRun` 和 `PipelineResource` 参数之间的关系。

![图片](img/CH07_F11_Sotobueno3.png)

图 7.11 `PipelineRun` 和 `PipelineResource` 之间的关系

执行以下命令以触发 `Pipeline` 进入 Kubernetes 集群。

列表 7.28 注册 `PipelineRun`

```
kubectl apply -f welcome-service-pipeline-run.yaml    ①
```

① 启动管道

在这一点上，对于 `Pipeline` 对象 `tasks` 部分中定义的每个 `task`，都会自动创建并执行一个 `TaskRun`。要列出它们，请在终端窗口中运行以下命令。

列表 7.29 列出所有 `TaskRuns`

```
tkn tr list    ①
```

① 列出所有 TaskRuns

输出提供了一个列表，列出了在 Kubernetes 集群中执行的所有 `TaskRuns` 及其状态：

```
NAME                                           STARTED      DURATION STATUS
welcome-pipeline-run-welcome-service-app-l2zns 1 minute ago ---      Running
```

`tkn` 允许我们检查 `TaskRun` 的日志，并在出现故障时找到错误原因。在终端中运行以下命令，使用 `-f` 选项以流式传输当前执行的实时日志。

列表 7.30 从 `PipelineRun` 流式传输日志

```
tkn tr logs welcome-pipeline-run-welcome-service-app-l2zns -f    ①
```

① 将 TaskRun ID 更改为前一个命令中显示的正确 ID

你将在控制台看到 `pipeline` 的日志：

```
[maven-build] Downloaded from central: https://repo.maven.apache.org/    ①
➥maven2/io/quarkus/quarkus-narayana-jta-deployment/1.11.3.Final/
➥quarkus-narayana-jta-deployment-1.11.3.Final.jar (8.4 kB at 19 kB/s)
Downloaded from central: https://repo.maven.apache.org/maven2/io/
➥quarkus/quarkus-agroal-deployment/1.11.3.Final/
➥quarkus-agroal-deployment-1.11.3.Final.jar (13 kB at 30 kB/s)
➥[maven-build] Downloading from central: https://repo.maven.apache.org/
➥maven2/io/quarkus/quarkus-hibernate-orm-deployment/1.11.3.Final/
➥quarkus-hibernate-orm-deployment-1.11.3.Final.jar

...                                                                      ②
[build-and-push-image] STEP 11: ENTRYPOINT [ "/deployments/run-java.sh" ]
[build-and-push-image] STEP 12: COMMIT test.org/k8ssecrets/
➥welcome-message:1.0.0
[build-and-push-image] --> abab5f4192b
[build-and-push-image] abab5f4192b3a5d9317419d61553d91baf0dfc4df16
➥b9ad58d2489f71ee0a30a                                                  ③
[build-and-push-image] Getting image source signatures
[build-and-push-image] Copying blob sha256:f0b7ce40f8b0d5a8e10eecc86
➥06f43a8bfbb48255da7d1ddc5e3281434f33b20
[build-and-push-image] Copying blob sha256:ba89bf93365092f038be159229ea
➥fbbc083ff8ffdfd2007e24c4c612e82871ee
[build-and-push-image] Copying blob sha256:04a05557bbadc648beca5cf01b71
➥b152ce7890a454381877144ac7e63b968874
[build-and-push-image] Copying blob sha256:821b0c400fe643d0a9f146c9ab8
➥ec12d8abe59eddd00796673b3154005515b26
[build-and-push-image] Copying blob sha256:7a6b87549e30f9dd8d25021fef3
➥c15626617941f83322ba5f6b1988cade6b1cf
[build-and-push-image] Copying config sha256:abab5f4192b3a5d9317419d61
➥553d91baf0dfc4df16b9ad58d2489f71ee0a30a
[build-and-push-image] Writing manifest to image destination
[build-and-push-image] Storing signatures
```

① 启动 Maven 进程以构建服务。

② Buildah 构建容器镜像。

③ 将容器镜像推送到容器注册库

记住，`Task` 作为 Pod 执行，每个 `step` 都在该 Pod 内的容器中执行。这可以在运行以下命令时看到。

列表 7.31 获取所有 Pods

```
kubectl get pods        ①

NAME                                 READY       STATUS      RESTARTS  AGE
welcome-pipeline-run-welcome-        0/3         Completed   0         6m22s
➥service-app-l2zns-pod-98b2l
```

① 获取管道执行期间创建的所有 Pods

由于 `welcome-service-apptask` 由三个 `steps`（Git 克隆、Maven 构建、Docker 构建/推送）组成，在 `task` 执行期间创建了三个容器，如 `READY` 列中所示。

当容器镜像发布到容器注册库时，CI 管道周期结束。但服务尚未部署，也尚未发布到 Kubernetes 集群。在下一章中，你将看到如何使用持续部署和 GitOps 方法将服务部署和发布到集群。

## 摘要

+   Kubernetes Secrets 既可以用于应用程序代码（例如，用户名、密码和 API 密钥），也可以用于 CI 管道（例如，外部服务的用户名和密码）。

+   持续集成密钥需要得到保护，就像其他任何密钥一样。你可以在 Tekton 和 Argo CD 中使用 `SealSecrets` 将加密的密钥存储在 Git 中。启用 Kubernetes 数据加密以存储 Kubernetes 中的加密密钥。

+   Tekton 是实现 CI 管道的 Kubernetes 原生平台。

+   Git 被用作单一的事实来源——不仅用于源代码，也用于 `pipeline` 脚本。
