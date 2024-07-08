# 第十四章：Kubernetes 中的持续部署

> 道不远，无不为，无不作。
> 
> 老子

在本章中，我们将探讨一个关键的 DevOps 原则——*持续集成* 和 *持续部署* (CI/CD)，并看看如何在云原生、基于 Kubernetes 的环境中实现这一点。我们概述了一些设置持续部署流水线以与 Kubernetes 配合工作的选项，并展示了一个完整的示例，使用了 Google 的 Cloud Build。我们还将涵盖 GitOps 的概念，并演示如何使用名为 Flux 的 GitOps 工具自动部署到 Kubernetes。

# 什么是持续部署？

*持续部署* (CD) 是将成功构建自动部署到生产环境。与测试套件一样，部署也应该是集中管理和自动化的。开发人员可以通过点击按钮、合并合并请求或推送 Git 发布标签来部署新版本。

CD 通常与*持续集成* (CI) 相关联：开发人员的更改会自动集成和测试到主线分支。这个想法是，如果你在一个分支上进行的更改会在合并到主线时破坏构建，持续集成会立即让你知道这一点，而不是等到你完成分支并做最终合并时才发现。持续集成和部署的组合通常被称为*CI/CD*。

持续部署的机制通常被称为*流水线*：一系列自动化操作，通过一系列测试和验收阶段，将代码从开发人员的工作站传送到生产环境。

容器化应用程序的典型流水线可能如下所示：

1.  开发人员将其代码更改推送到存储库。

1.  构建系统会自动构建当前版本的代码并运行测试。

1.  如果所有测试通过，容器图像将被发布到中央容器注册表。

1.  新构建的容器会自动部署到预备环境。

1.  预备环境会经历一些自动化和/或手动接受测试。

1.  验证过的容器映像将被部署到生产环境。

一个关键点是通过您的各个环境测试和部署的工件不是*源代码*，而是*容器*。在源代码和运行的二进制文件之间存在许多错误产生的方式，而测试容器而不是代码可以帮助捕捉到其中很多错误。

CD 的最大好处是*在生产中没有意外*；除非确切的二进制图像已在预备环境成功测试过，否则不会部署任何东西。

您可以在“使用 Cloud Build 的 CI/CD 流水线”中看到此类 CD 流水线的详细示例。

# 我应该使用哪个 CD 工具？

和往常一样，问题并不是可用工具的短缺，而是选择的范围之广。有几个专门为云原生应用程序设计的新的 CI/CD 工具，而长期存在的传统构建工具如 Jenkins 也现在有插件允许它们与 Kubernetes 和容器一起工作。

因此，如果您已经在进行 CI/CD，您可能不需要切换到全新的系统。如果您正在将现有应用程序迁移到 Kubernetes，您可能只需对现有的流水线进行少量调整即可。

在下一节中，我们将简要介绍一些受欢迎的托管和自托管 CI/CD 工具选项。我们当然不可能覆盖所有工具，但这里有一个快速列表，应该能帮助您开始搜索。

# 托管 CI/CD 工具

如果您正在寻找一个开箱即用的解决方案用于您的 CI/CD 流水线，而您不需要维护底层基础设施，那么您应该考虑使用托管服务。主要的云提供商都提供了与其生态系统良好集成的 CI/CD 工具，因此首先探索一下已经包含在您云账户中的工具是值得的。

## Azure 流水线

Microsoft 的 Azure DevOps 服务（以前称为 Visual Studio Team Services）包括一个持续交付管道工具，称为 [Azure 流水线](https://oreil.ly/pVbMb)，类似于 Google Cloud Build。

## Google Cloud Build

如果您在 Google Cloud Platform 上运行您的基础设施，那么您应该看看 [Cloud Build](https://cloud.google.com/build)。它作为各种构建步骤运行容器，并且流水线的配置 YAML 存储在您的代码仓库中。

您可以配置 Cloud Build 来监视您的 Git 仓库。当触发预设条件时，比如推送到特定分支或标签时，Cloud Build 将运行您指定的流水线，比如构建新的容器镜像，运行您的测试套件，发布镜像，甚至部署新版本到 Kubernetes。

要查看 Cloud Build 中完整的工作示例，请参阅 “使用 Cloud Build 的 CI/CD 流水线”。

## Codefresh

[Codefresh](https://codefresh.io) 是一个用于测试和部署应用程序到 Kubernetes 的托管服务。一个有趣的特性是能够为每个功能分支部署临时的 staging 环境。

使用容器，Codefresh 可以按需构建、测试和部署环境，然后您可以配置如何将您的容器部署到集群中的各种环境中。

## GitHub Actions

[GitHub Actions](https://oreil.ly/eeWSL) 已经整合到流行的托管 Git 仓库站点中。通过 GitHub 仓库共享 Actions，可以非常容易地混合和匹配并在不同应用程序之间共享构建工具。Azure 发布了一个用于部署到 Kubernetes 集群的 [热门 GitHub Action](https://oreil.ly/GcHam)。

GitHub 还提供了在您自己的服务器上本地运行 GitHub Action runners 的选项，以保持构建在您的网络内部进行。

## GitLab CI

GitLab 是一个流行的替代品，用于托管 Git 仓库，与 GitHub 类似。您可以使用他们的托管服务，或在自己的基础设施上运行 GitLab。它自带强大的内置 CI/CD 工具，[GitLab CI](https://oreil.ly/e5f11)，可用于测试和部署代码。如果您已经在使用 GitLab，那么使用 GitLab CI 来实现您的持续部署流水线是有意义的。

# 自托管的 CI/CD 工具

如果您更倾向于拥有更多基础设施的管控权以用于您的流水线，那么还有几个很好的选择可供您在任何地方运行。其中一些工具在 Kubernetes 出现之前就存在，一些则是专门为基于 Kubernetes 的 CI/CD 流水线开发的。

## Jenkins

[Jenkins](https://jenkins.io) 是一个非常广泛采纳的 CI/CD 工具，存在多年。它有几乎可以想到的所有工作流程插件，包括 Docker、`kubectl` 和 Helm。还有一个专门用于在 Kubernetes 中运行 Jenkins 的新项目，称为 [JenkinsX](https://jenkins-x.io)。

## Drone

[Drone](https://oreil.ly/qtXoR) 是一个使用容器构建的工具。它简单轻量，流水线由单个 YAML 文件定义。由于每个构建步骤都是运行容器，这意味着您可以在 Drone 上运行可以在容器中运行的任何东西。

## Tekton

[Tekton](https://tekton.dev) 引入了一个有趣的概念，即 CI/CD 组件实际上由 Kubernetes CRD 组成。因此，您可以使用原生 Kubernetes 资源构建您的构建、测试和部署步骤，并像管理 Kubernetes 集群中的其他内容一样管理流水线。

## Concourse

[Concourse](https://concourse-ci.org) 是一个用 Go 编写的开源 CD 工具。它也采用了声明式流水线的方法，类似于 Drone 和 Cloud Build，使用 YAML 文件来定义和执行构建步骤。Concourse 提供了一个[官方的 Helm chart](https://oreil.ly/UCvPz)，可以在 Kubernetes 上部署它，从而轻松快速地启动一个容器化的流水线。

## Spinnaker

[Spinnaker](https://spinnaker.io) 非常强大和灵活，但乍一看可能有点令人生畏。最初由 Netflix 开发，特别擅长大规模和复杂的部署，例如蓝/绿部署（参见“蓝/绿部署”）。有一本关于 Spinnaker 的免费电子书，名为 [*Continuous Delivery with Spinnaker*](https://oreil.ly/hkb64)（O’Reilly），可以帮助您了解 Spinnaker 是否符合您的需求。

## Argo

[Argo CD](https://oreil.ly/8FmGn)是一个类似于 Flux 的 GitOps 工具（参见“GitOps”），它通过将运行在 Kubernetes 中的内容与存储在中央 Git 仓库中的清单进行同步，自动化部署。Argo 不像使用`kubectl`或`helm`“推送”变更，而是持续从 Git 仓库中“拉取”变更，并在集群内应用它们。Argo 还提供了一个[流行的流水线工具](https://argoproj.github.io)，用于运行各种流水线工作流，不仅仅是用于 CI/CD。

## Keel

[Keel](https://keel.sh)并不是一个完整的端到端 CI/CD 工具，它仅关注于当发布新的容器镜像到容器注册表时部署它们。它可以配置响应 webhook、发送和接收 Slack 消息，并在部署到新环境之前等待批准。如果你已经有一个适合你的 CI 流程，但只需要自动化 CD 部分的方法，那么 Keel 可能值得评估。

# 使用 Cloud Build 的 CI/CD 流水线

现在你已经了解了 CI/CD 的一般原则，并学习了一些工具选项，让我们来看一个完整的端到端演示流水线的例子。

我们的想法并不是你一定要像我们这里使用完全相同的工具和配置；相反，我们希望你能够了解如何将所有东西组合起来，并可以调整这个示例的某些部分以适应你自己的环境。

在这个例子中，我们将使用 GitHub、Google Kubernetes Engine (GKE)集群和 Google Cloud Build，但我们不依赖于这些产品的任何特定功能。你可以使用你喜欢的任何工具复制这种类型的流水线。

如果你想使用自己的 GCP 账户来完成这个示例，请记住它会使用一些可计费资源。你会希望在测试完云资源后删除和清理它们，以确保不会意外收费。

如果你宁愿在本地尝试一个 CI/CD 示例而不使用任何 Google Cloud 资源，请跳到“GitOps”。

## 设置 Google Cloud 和 GKE

如果你是第一次注册新的 Google Cloud 账户，你将有资格获得一些免费的积分，这些积分可以让你在几个月内运行一个 Kubernetes 集群和其他云资源，而不会被收费。但是，在尝试任何云服务时，你一定要监控你的使用情况，以确保不会产生任何意外的费用。你可以在[Google Cloud 平台网站](https://cloud.google.com/free)了解更多关于免费套餐的信息并创建一个账户。

一旦您注册并登录到您自己的 Google Cloud 项目中，请按照 [这些说明](https://oreil.ly/zcLVd) 创建 GKE 集群。对于此示例来说，Autopilot 集群将是合适的选择，并选择一个靠近您位置的区域。您还需要在新项目中启用 [Cloud Build](https://oreil.ly/O8ZNV) 和 [Artifact Registry](https://oreil.ly/b86m9) API，因为我们将与 GKE 一起使用这些服务。

接下来，我们将引导您完成以下准备创建流水线的步骤：

1.  Fork demo 仓库到您自己的个人 GitHub 帐户中。

1.  在 Artifact Registry 中创建一个容器仓库。

1.  验证 Cloud Build 以使用 Artifact Registry 和 GKE。

1.  为在推送到任何 Git 分支时构建和测试创建 Cloud Build 触发器。

1.  创建基于 Git 标签部署到 GKE 的触发器。

## Fork Demo Repository

使用您的 GitHub 帐户，在 GitHub 界面上 fork [demo repo](https://oreil.ly/LAI8f)。如果您对 repo fork 不熟悉，您可以在 [GitHub docs](https://oreil.ly/S37E1) 中了解更多信息。

## 创建 Artifact Registry 容器仓库

GCP 提供了一个名为 Artifact Registry 的私有 artifact 仓库工具，可存储 Docker 容器、Python 包、npm 包和其他类型的 artifact。我们将使用它来托管我们将构建的 demo 容器映像。

在 [Google Cloud web 控制台](https://oreil.ly/aIZet) 的 Artifact Registry 页面创建名为 `demo` 的新 Docker 仓库，按照 [这些说明](https://oreil.ly/sBY0O)。在创建 GKE 集群的同一 Google Cloud 区域中创建。

您还需要授权 Cloud Build 服务帐户具有权限更改您的 Kubernetes Engine 集群。在 GCP 的 IAM 部分，按照 [GCP 文档中的说明](https://oreil.ly/tMtAR) 为 Cloud Build 服务帐户授予 *Kubernetes Engine Developer* 和 *Artifact Registry Repository Administrator* IAM 角色。

## 配置 Cloud Build

现在让我们看看构建流水线中的步骤。在许多现代 CI/CD 平台中，流水线的每个步骤都是运行一个容器。使用 YAML 文件在您的 Git repo 中定义构建步骤。每个步骤使用容器意味着您可以轻松地打包、版本化和共享不同流水线之间的通用工具和脚本。

在 demo 仓库内，有一个名为 *hello-cloudbuild-v2* 的目录。在该目录中，您会找到定义我们 Cloud Build 流水线的 *cloudbuild.yaml* 文件。

###### 注意

注意在 *hello-cloudbuild-v2* 目录名中的 *-v2*。这里有一些第二版书籍的更改，与第一版不匹配。为了避免破坏第一版中使用的任何示例，我们在此示例中使用了完全不同的目录。

让我们依次查看此文件中的每个构建步骤。

## 构建测试容器

这是第一步：

```
- id: build-test-image
  dir: hello-cloudbuild-v2
  name: gcr.io/cloud-builders/docker
  entrypoint: bash
  args:
    - -c
    - |
      docker image build --target build --tag demo:test .
```

与所有 Cloud Build 步骤一样，这由一组 YAML 键值对组成：

+   `id`为构建步骤提供了一个人性化标签。

+   `dir`指定要在 Git 仓库中工作的子目录。

+   `name`标识要为此步骤运行的容器。

+   `entrypoint`指定在容器中运行的命令，如果不是默认值。

+   `args`提供了传递给入口点命令的必要参数。（我们在这里使用`bash -c |`的小技巧，将我们的`args`保持在一行上，这样更容易阅读。）

第一步的目的是构建一个我们可以用来运行应用程序测试的容器。由于我们使用了多阶段构建（参见“理解 Dockerfile”），我们现在只想构建第一个阶段。因此，我们使用`--target build`参数，告诉 Docker 只构建 Dockerfile 中`FROM golang:1.17-alpine AS build`下的部分，并在继续下一步之前停止。

这意味着生成的容器仍然安装有 Go，以及标记为`...AS build`的步骤中使用的任何包或文件，这意味着我们可以使用此镜像来运行测试套件。通常情况下，您需要在容器中安装运行测试所需的包，但不希望这些包出现在最终的生产镜像中。在这里，我们实际上正在构建一个一次性容器，仅用于运行测试套件，之后将其丢弃。

## 运行测试

这是我们`cloudbuild.yaml`中的下一步：

```
- id: run-tests
  dir: hello-cloudbuild-v2
  name: gcr.io/cloud-builders/docker
  entrypoint: bash
  args:
    - -c
    - |
      docker container run demo:test go test
```

由于我们将我们的一次性容器标记为`demo:test`，在 Cloud Build 中此临时镜像将在本次构建的其余过程中仍然可用。此步骤将对该容器运行`go test`命令。如果任何测试失败，此步骤将失败，并且构建将退出。否则，它将继续进行到下一步。

## 构建应用程序容器

在这里，我们再次运行`docker build`，但没有使用`--target`标志，因此我们运行整个多阶段构建，最终得到最终的应用程序容器：

```
- id: build-app
  dir: hello-cloudbuild-v2
  name: gcr.io/cloud-builders/docker
  args:
    - docker
    - build
    - --tag
    - ${_REGION}-docker.pkg.dev/$PROJECT_ID/demo/demo:$COMMIT_SHA
    - .
```

## 替换变量

为了使 Cloud Build 管道文件可重用和灵活，我们使用变量，或者 Cloud Build 称之为*替换*。任何以`$`开头的内容在管道运行时都将被替换。例如，`$PROJECT_ID`将插入为 Google Cloud 项目，在特定构建运行时，`$COMMIT_SHA`是触发此构建的特定 Git 提交 SHA。在 Cloud Build 中，用户定义的替换必须以下划线字符(`_`)开头，并且只能使用大写字母和数字。我们将在创建构建触发器时使用`${_REGION}`替换变量。

## Git SHA 标签

您可能会问为什么我们在容器映像标签中使用了`$COMMIT_SHA`。在 Git 中，每个提交都有一个唯一标识符，称为 SHA（以生成它的安全哈希算法命名）。SHA 是一串长的十六进制数字，如`5ba6bfd64a31eb4013ccaba27d95cddd15d50ba3`。

如果您使用此 SHA 标记您的镜像，它将提供一个链接到生成它的确切 Git 提交的链接，这也是容器中代码的完整快照。使用带有源 Git SHA 标记的构建工件的好处在于，您可以同时构建和测试许多特性分支，而不会出现任何冲突。

## 验证 Kubernetes 清单

在管道的这一步骤中，我们已经构建了一个通过测试的新容器，并且准备部署。但在我们部署之前，我们还希望快速检查一下我们的 Kubernetes 清单是否有效。在构建的最后一步中，我们将运行`helm template`生成我们的 Helm 图表的渲染版本，然后将其传输到`kubeval`工具（参见`kubeval`）以检查是否存在任何问题：

```
- id: kubeval
  dir: hello-cloudbuild-v2
  name: cloudnatived/helm-cloudbuilder
  entrypoint: bash
  args:
    - -c
    - |
      helm template ./k8s/demo/ | kubeval
```

###### 注意

请注意，我们在这里使用自己的 Helm 容器镜像（`cloudnatived/helm-cloudbuilder`），其中包含 `helm` 和 `kubeval`，但您也可以创建和使用自己的“构建镜像”，其中包含您使用的任何其他构建或测试工具。只需记住，保持构建镜像的小巧和精简很重要（参见“最小化容器镜像”）。当您每天运行数十甚至数百个构建时，大容器的拉取时间会累积增加。

## 发布镜像

假设管道中的每个步骤都成功完成，Cloud Build 然后可以将生成的容器镜像发布到您之前创建的 Artifact Registry 存储库中。要指定要发布的构建中的哪些镜像，请在 Cloud Build 文件的`images`下列出它们：

```
images:
  - ${_REGION}-docker.pkg.dev/$PROJECT_ID/demo/demo:$COMMIT_SHA
```

## 创建第一个构建触发器

现在您已经了解了管道的工作原理，请在 Google Cloud 中创建实际执行管道的构建触发器，基于我们指定的条件。Cloud Build 触发器指定要监视的 Git 存储库，激活的条件（例如推送到特定分支或标签），以及要执行的管道文件。

现在继续创建新的 Cloud Build 触发器。登录到您的 Google Cloud 项目，并浏览到[Cloud Build 触发器页面](https://oreil.ly/LLuWt)。

单击“添加触发器”按钮以创建新的构建触发器，并选择 GitHub 作为源代码库。系统将要求您授权 Google Cloud 访问您的 GitHub 存储库。选择`YOUR_GITHUB_USERNAME/demo`，Google Cloud 将链接到您 fork 的演示存储库的副本。您可以随意命名触发器。

在分支部分下，选择`.*`以匹配任何分支。

在配置部分，选择云构建配置文件，并将位置设置为*hello-cloudbuild-v2/cloudbuild.yaml*，这是演示存储库中文件所在的位置。

最后，我们需要创建一些替换变量，以便我们可以为不同的构建重复使用相同的*cloudbuild.yaml*文件。

对于此示例，您需要向触发器添加以下替换变量：

+   `_REGION` 应该是您部署 Artifact Registry 和 GKE 集群的 GCP 区域，如 `us-central1` 或 `southamerica-east1`。

当您完成时，点击创建触发器按钮。现在，您可以测试触发器并查看发生了什么！

## 测试触发器

继续对您分叉的演示库进行更改。编辑 *main.go* 和 *main_test.go*，用 `Hola` 替换 `Hello` 或您喜欢的任何内容，并保存这两个文件（我们将在下面的示例中使用 `sed`）。如果您安装了 Golang，您还可以在本地运行测试，以确保测试套件仍然通过。准备好后，提交并推送更改到您分叉的 Git 存储库：

```
cd hello-cloudbuild-v2
sed -i *s/Hello/Hola/g* main.go
sed -i *s/Hello/Hola/g* main_test.go
go test
PASS
ok  	github.com/cloudnativedevops/demo/hello-cloudbuild	0.011s
git commit -am "Update greeting"
git push
...
```

如果您查看 [Cloud Build Web UI](https://oreil.ly/YAc9a)，您会看到项目中最近构建的列表。您应该看到列表顶部有一个当前刚刚推送的当前更改的构建。它可能仍在运行中，也可能已经完成。

希望您会看到一个绿色勾 indicating that all steps passed。如果没有，请检查构建中的日志输出，看看是什么失败了。

假设它通过了，一个容器应该已经发布到您的私有 Google Artifact Registry，并标记有您更改的 Git 提交 SHA。

## 从 CI/CD 管道部署

现在您可以通过 Git 推送触发构建、运行测试并发布最终容器到注册表，您已准备好将该容器部署到 Kubernetes。

对于此示例，我们将假设有两个环境，一个用于 `production`，一个用于 `staging`，并将它们部署到单独的命名空间：`staging-demo` 和 `production-demo`。两者将在同一个 GKE 集群中运行（尽管您可能希望为您的真实应用程序使用单独的集群）。

为了简化事务，我们将使用 Git 标签 `production` 和 `staging` 触发对每个环境的部署。您可能有自己的版本管理过程，如使用语义化版本（[SemVer](https://semver.org)）、发布标签或在更新主分支或主干分支时自动部署到暂存环境。请随意根据您自己的情况调整这些示例。

我们将配置 Cloud Build 在将 `staging` Git 标签推送到仓库时部署到暂存，将 `production` 标签推送时部署到生产。这需要一个使用不同 Cloud Build YAML 文件 *cloudbuild-deploy.yaml* 的新管道。让我们看看我们部署管道中的步骤：

### 获取 Kubernetes 集群的凭据

要使用 Cloud Build 部署到 Kubernetes，构建将需要一个有效的 `KUBECONFIG`，我们可以使用 `kubectl` 获取它：

```
- id: get-kube-config
  dir: hello-cloudbuild-v2
  name: gcr.io/cloud-builders/kubectl
  env:
  - CLOUDSDK_CORE_PROJECT=$PROJECT_ID
  - CLOUDSDK_COMPUTE_REGION=${_REGION}
  - CLOUDSDK_CONTAINER_CLUSTER=${_CLOUDSDK_CONTAINER_CLUSTER}
  - KUBECONFIG=/workspace/.kube/config
  args:
    - cluster-info
```

### 部署到集群

一旦构建经过身份验证，它可以运行 Helm 实际在集群中升级（或安装）应用程序：

```
- id: deploy
  dir: hello-cloudbuild-v2
  name: cloudnatived/helm-cloudbuilder
  env:
    - KUBECONFIG=/workspace/.kube/config
  args:
    - helm
    - upgrade
    - --create-namespace
    - --install
    - ${TAG_NAME}-demo
    - --namespace=${TAG_NAME}-demo
    - --values
    - k8s/demo/${TAG_NAME}-values.yaml
    - --set
    - container.image=${_REGION}-docker.pkg.dev/$PROJECT_ID/demo/demo
    - --set
    - container.tag=${COMMIT_SHA}
    - ./k8s/demo
```

我们向 `helm upgrade` 命令传递了一些额外的标志：

`namespace`

应部署应用程序的命名空间

`values`

用于此环境的 Helm 值文件

`set container.image`

设置部署容器的名称

`set container.tag`

使用具体标签（源 Git SHA）部署图像

## 创建一个部署触发器

现在让我们为我们想象中的`staging`环境创建一个新的 Cloud Build 触发器。

在 Cloud Build Web UI 中创建一个新的触发器，就像您在“创建第一个构建触发器”中所做的那样。仓库将保持不变，但这次配置它在推送*tag*而不是分支时触发，并设置标签名称与*staging*匹配。

此次构建将不再使用*cloudbuild.yaml*文件，而是使用`*hello-cloudbuild-v2/cloudbuild-deploy.yaml*`。

在`替代变量`部分，我们将设置一些特定于部署构建的值：

+   `_REGION`将与您在第一个触发器中使用的相同。它应该与您创建 GKE 集群和 Artifact Registry 仓库的 GCP 可用区域匹配。

+   `_CLOUDSDK_CONTAINER_CLUSTER`是您的 GKE 集群的名称。

在这里使用这些变量意味着，即使这些环境位于不同的集群或不同的 GCP 项目中，我们也可以使用相同的 YAML 文件来部署分别用于临时和生产环境。

一旦为`staging`标签创建了触发器，请继续通过将`staging`标签推送到仓库来尝试它：

```
`git tag -f staging`
`git push -f origin refs/tags/staging`
Total 0 (delta 0), reused 0 (delta 0) To github.com:domingusj/demo.git
 * [new tag]         staging -> staging
```

与以往一样，您可以在云构建 UI 中查看[构建进度](https://oreil.ly/UQmGq)。如果一切顺利，Cloud Build 应能够成功验证到您的 GKE 集群，并将应用程序的临时版本部署到`staging-demo`命名空间。您可以通过检查[GKE 仪表板](https://oreil.ly/LbLjU)（或使用`helm status`）来验证此操作。

最后，按照相同的步骤创建一个单独的 Cloud Build 触发器，以便在推送到`production`标签时部署到生产环境。如果一切顺利，您将在新的`production-demo`命名空间中运行该应用程序的另一份副本。在此示例中，我们将两个环境都部署到同一个 GKE 集群，但对于真实应用程序，您可能希望将它们分开。

## 调整示例流水线

当您完成尝试演示流水线时，您将希望删除为测试创建的任何 GCP 资源，包括 GKE 集群、`demo`Artifact Registry 仓库和您的 Cloud Build 触发器。

我们希望此示例演示了 CI/CD 流水线的关键概念。如果您使用 Cloud Build，则可以将这些示例用作设置自己流水线的起点。如果您使用其他工具，希望您能轻松地调整我们在此展示的模式，以适应您自己的环境。自动化应用程序的构建、测试和部署步骤将极大地提高所有参与者创建和部署软件的体验。

# GitOps

正如我们在“基础设施即代码”章节中提到的，作为行业向 DevOps 转变的一部分，管理基础设施需要通过代码和源代码控制来进行。 “GitOps” 是一个较新的术语，根据不同的理解，其含义可能有所不同。但在高层次上，GitOps 使用源代码控制（Git 是较流行的源代码控制工具之一）来以自动化方式跟踪和管理基础设施。想象一下 Kubernetes 协调循环，但应用于更广泛的范围，即通过推送变更到 Git 存储库来配置和部署任何和所有基础设施。一些现有的 CI/CD 工具已经重新定位自己为“GitOps”工具，我们预计这个概念将在未来几年内在软件行业中快速发展和演变。

在本节中，我们将使用一个名为 Flux 的工具，通过将更改推送到 GitHub 存储库，自动将演示应用程序部署到本地 Kubernetes 集群。

## Flux

Weaveworks（[`eksctl`](https://eksctl.io)的创建者）可能是首先创造[GitOps 术语](https://oreil.ly/Bjd3D)的人。他们还构建了一个名为[Flux](https://oreil.ly/Eoqs6)的较受欢迎的 GitOps 工具。它可以通过轮询 Git 存储库、监视任何更改并自动应用 Flux Pods 内部运行的更改来自动将更改部署到 Kubernetes 集群。让我们尝试一个示例，看看它如何工作。

### 设置 Flux

`flux` 是与 Flux 进行交互的 CLI 工具，也可用于在 Kubernetes 中安装 Flux 组件。请按照您操作系统的[安装说明](https://oreil.ly/erpNZ)设置，并将您的 `kubectl` 指向测试 Kubernetes 集群。

您需要创建一个 GitHub 个人访问令牌，以便 Flux 可以安全地与 GitHub 进行通信。您可以在 GitHub 个人资料的设置页面的开发者设置部分生成一个。它将需要 `repo` 权限，并且您可以决定是否希望它定期自动过期，或者设置为永不过期。对于本示例，任何一种方式都可以，但在真实的生产系统中，您应始终具有定期轮换凭据的流程。

请按照[GitHub 说明](https://oreil.ly/amjIN)生成个人访问令牌，并将其导出到您的环境，同时附上您的用户名，然后使用 `flux check` 命令检查 Flux 是否准备好安装：

```
export GITHUB_TOKEN=*YOUR_PERSONAL_ACCESS_TOKEN*
export GITHUB_USER=*YOUR_GITHUB_USERNAME*
flux check --pre
checking prerequisites
Kubernetes 1.21.4 >=1.19.0-0
prerequisites checks passed
...
```

### 安装 Flux

假设检查通过，您现在可以安装 Flux 了！作为过程的一部分，它将使用您的个人访问令牌自动在您的帐户中创建一个新的 GitHub 存储库，然后将该存储库用于管理您的集群。

```
`flux bootstrap github \`
  `--owner=$GITHUB_USER \`
  `--repository=flux-demo \`
  `--branch=main \`
  `--path=./clusters/demo-cluster \`
  `--personal`
connecting to github.com ... cloned repository ... all components are healthy
```

此外，您应该看到 Flux 成功连接到您的仓库。您可以浏览到您的新仓库，并看到在这个仓库内，Flux 创建了一个名为*clusters/demo-cluster/flux-system*的目录，其中包含所有在新`flux-system`命名空间中运行的 Flux Kubernetes 清单。

### 使用 Flux 创建新部署

现在让我们使用 Flux 自动部署一个新的命名空间和部署到您的集群中。按照适当的 GitOps 方式，我们只需将更改推送到 Git 仓库即可完成此操作。您需要克隆 Flux 创建的新仓库，这意味着您需要设置 GitHub 凭据以推送新的提交。如果您尚未完成此操作，可以按照[GitHub 提供的这些说明](https://oreil.ly/UVcoA)进行操作。克隆仓库后，在`flux-system`旁边为我们的新`flux-demo`部署创建一个新目录：

```
`git clone git@github.com:_$GITHUB_USER_/flux-demo.git`
`cd flux-demo`
`mkdir clusters/demo-cluster/flux-demo`
`cd clusters/demo-cluster/flux-demo`
```

接下来，我们将使用`kubectl`和`--dry-run`标志生成一个新命名空间和名为`flux-demo`的部署所需的 YAML。保存这些到新的清单文件后，我们将其提交并推送到仓库：

```
`kubectl create namespace flux-demo -o yaml --dry-run=client > namespace.yaml`
`kubectl create deployment flux-demo -n flux-demo \`
`-o yaml --dry-run=client --image=cloudnatived/demo:hello > deployment.yaml`
`git add namespace.yaml deployment.yaml`
`git commit -m "Create flux-demo Deployment"`
`git push`
```

由于 Flux 定期轮询 Git 仓库并监视任何更改，当它检测到新文件时，它将自动创建和部署您的新`flux-demo`清单：

```
`kubectl get pods -n flux-demo`
NAME                        READY   STATUS    RESTARTS   AGE flux-demo-c77cc8d6f-zfzql   1/1     Running   0          39s
```

除了普通的 Kubernetes 清单外，Flux 还可以管理 Helm 发布和使用 kustomize 的清单。您还可以配置 Flux 轮询您的容器注册表，并自动部署新的镜像。然后，它将向仓库进行新的 Git 提交，跟踪它部署的镜像版本，使您的 Git 仓库与实际在集群中运行的内容保持同步。

此外，就像 Kubernetes 的协调循环一样，Flux 持续监视其管理的资源是否有任何手动更改。它将尝试保持集群与 Git 仓库中的内容同步。这样，任何使用 Flux 管理的手动更改应该会自动回滚，使您更加信任 Git 仓库作为集群中应该运行的最终来源或真实数据源。

这是 GitOps 的主要目标之一：您应该能够使用在 Git 中跟踪的代码自动管理 Kubernetes。在使用 Flux 时，将更改推送到此仓库是您唯一应该对集群进行更改的方式。使用 Git 来管理您的基础设施意味着您将在提交历史中有所有更改的记录，同时所有更改也可以作为团队合并请求流程的一部分进行同行评审。

# 概述

为您的应用程序设置持续部署流水线可以让您以一致、可靠和快速的方式部署软件。理想情况下，开发人员应该能够将代码推送到源代码控制库，所有构建、测试和部署阶段都在一个集中的流水线中自动完成。

由于有许多 CI/CD 软件和技术选择，我们无法给出适用于所有人的单一配方。相反，我们的目标是向您展示 CD 为何有益，并在您组织内实施时提供一些重要的思考要点：

+   在构建新管道时，决定使用哪些 CI/CD 工具是一个重要的过程。我们在本书中提到的所有工具都可能被整合到几乎任何现有环境中。

+   Jenkins、GitHub Actions、GitLab、Drone、Cloud Build 和 Spinnaker 只是一些与 Kubernetes 配合良好的热门 CI/CD 工具。

+   使用代码定义构建管道步骤使您能够与应用代码一起跟踪和修改这些步骤。

+   容器使开发人员能够将构建产物推广至各种环境，例如测试、预发布，最终到生产环境，理想情况下无需重新构建新的容器。

+   我们使用 Cloud Build 的示例管道应该很容易适应其他工具和应用程序类型。无论使用的工具或软件类型如何，CI/CD 管道的总体构建、测试和部署步骤基本相同。

+   GitOps 是谈论 CI/CD 管道时使用的较新术语。其主要理念是部署和基础设施变更应使用在源代码控制（Git）中跟踪的代码进行管理。

+   Flux 和 Argo 是流行的 GitOps 工具，可以在您将代码更改推送到 Git 存储库时自动应用更改到您的集群。
