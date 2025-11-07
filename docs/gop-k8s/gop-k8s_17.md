# 附录 B. 设置 GitOps 工具

本附录将逐步说明设置第三部分教程所需工具的步骤。

## B.1 安装 Argo CD

Argo CD 支持多种安装方法。您可能使用基于 Kustomize 的官方安装、社区维护的 Helm 图表^(1)，甚至 Argo CD 操作员^(2) 来管理您的 Argo CD 部署。最简单的安装方法只需要使用单个 YAML 文件。请继续使用以下命令将 Argo CD 安装到您的 minikube 集群中：

```
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

此命令使用适用于大多数用户的默认设置安装所有 Argo CD 组件。出于安全原因，Argo CD UI 和 API 默认情况下不会在集群外部暴露。在 minikube 上完全打开访问权限是绝对安全的。

通过在单独的终端中运行以下命令，在您的 minikube 集群中启用负载均衡器访问^(3)：

```
$ minikube tunnel
```

使用以下命令打开对 `argocd-server` 服务的访问并获取访问 URL：

```
$ kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Argo CD 提供了基于 Web 的用户界面和命令行界面 (CLI)。为了简化说明，我们将在此教程中使用 CLI 工具。让我们继续安装 CLI 工具。您可以使用以下命令在 Mac 上安装 Argo CD CLI，或者遵循官方的入门指南^(4) 在您的平台上安装 CLI：

```
$ brew tap argoproj/tap
$ brew install argoproj/tap/argocd
```

一旦安装了 Argo CD，它就有一个预配置的管理员用户。初始管理员密码是自动生成的，是 Argo CD API 服务器 Pod 名称，可以使用此命令检索：

```
$ kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
```

使用以下命令获取 Argo CD 服务器 URL 并使用 Argo CD CLI 更新生成的密码：

```
$ argocd login <ARGOCD_SERVER-HOSTNAME>:<PORT>
$ argocd account update-password
```

`<ARGOCD_SERVER-HOSTNAME>:<PORT>` 是 minikube API 和服务端口，应从 Argo CD URL 获取。您可以使用以下命令检索 URL：

```
minikube service argocd-server -n argocd --url
```

命令返回 HTTP 服务 URL。请确保删除 http://，并仅使用主机名和端口号使用 Argo CD CLI 登录。

最后，登录到 Argo CD 用户界面。请在浏览器中打开 Argo CD URL，并使用管理员用户名和您的密码登录。您现在可以开始了！

## B.2 安装 Jenkins X

Jenkins X CLI 依赖于 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) ^(5) 和 Helm^(6)，并将尽力安装这些工具。然而，我们笔记本电脑上可能存在的所有可能排列组合接近无限，因此您最好自己安装这些工具。

注意：在撰写本文时，2021 年 2 月，Jenkins X 还不支持 Helm v3+。请确保您正在使用 Helm CLI v2+。

### B.2.1 前提条件

您可以使用（几乎）任何 Kubernetes 集群，但它需要是公开可访问的。这样做的主要原因在于 GitHub 触发器。Jenkins X 严重依赖于 GitOps 原则。大部分事件将由 GitHub webhook 触发。如果您的集群无法从 GitHub 访问，您将无法触发这些事件，并且您将难以跟随示例进行操作。

现在，这提出了两个重大问题。您可能更喜欢在本地使用 minikube 或 Docker for Desktop 进行练习，但这两个工具都无法从您的笔记本电脑外部访问。您可能有一个无法从外部访问的企业集群。在这些情况下，我们建议您使用 AWS、GCP 或其他地方的服务。最后，我们将使用命令`hub`执行一些 GitHub 操作。如果您还没有安装，请安装它。

注意请参考附录 A 以获取有关配置 AWS 或 GCP Kubernetes 集群的更多信息。

为了您的方便，我们将使用的所有工具列表如下：

+   Git

+   kubectl

+   Helm^(7)

+   AWS CLI

+   `eksctl`^(8)（如果使用 AWS EKS）

+   `gcloud`（如果使用 Google GKE）

+   `hub`^(9)

现在，让我们安装 Jenkins X CLI：

```
$ brew tap jenkins-x/jx
$ brew install jx
```

### B.2.2 在 Kubernetes 集群中安装 Jenkins X

我们如何以比我们通常安装软件更好的方式安装 Jenkins X？Jenkins X 的配置应该被定义为代码并存储在 Git 仓库中，这正是社区为我们创建的。它维护一个 GitHub 仓库，其中包含 Jenkins X 平台定义的结构，以及一个将安装它的管道，以及一个我们可以用它来调整以满足特定需求的 requirements 文件。

注意您还可以参考 Jenkins X 网站^(10)来了解如何在您的 Kubernetes 集群中设置 Jenkins X。

让我们看看这个仓库：

```
$ open "https://github.com/jenkins-x/jenkins-x-boot-config.git"
```

一旦您在浏览器中看到仓库，您首先需要在您的 GitHub 账户下创建一个分支。我们稍后会探索仓库中的文件。

接下来，我们将定义一个变量`CLUSTER_NAME`，它将，正如您所猜到的，保存我们刚刚创建的集群的名称。在随后的命令中，请将[...]的第一个出现替换为集群名称，第二个替换为您的 GitHub 用户：

```
$ export CLUSTER_NAME=[...]
$ export GH_USER=[...]
```

在我们分支 boot 仓库并且知道我们的集群名称后，我们可以使用一个合适的名称克隆仓库，该名称将反映我们即将安装的 Jenkins X 的命名方案：

```
$ git clone \
    https://github.com/$GH_USER/jenkins-x-boot-config.git \
    environment-$CLUSTER_NAME-dev
```

包含（几乎）所有可用于自定义设置的参数的关键文件是 jx-requirements.yml。让我们看看它：

```
$ cd environment-$CLUSTER_NAME-dev
$ cat jx-requirements.yml
cluster:
  clusterName: ""
  environmentGitOwner: ""
  project: ""
  provider: gke
  zone: ""
gitops: true
environments:
- key: dev
- key: staging
- key: production
ingress:
  domain: ""
  externalDNS: false
  tls:
    email: ""
    enabled: false
    production: false
kaniko: true
secretStorage: local
storage:
  logs:
    enabled: false
    url: ""
  reports:
    enabled: false
    url: ""
  repository:
    enabled: false
    url: ""
versionStream:
  ref: "master"
  url: https://github.com/jenkins-x/jenkins-x-versions.git
webhook: prow
```

正如您所看到的，该文件包含的值格式类似于与 Helm 图表一起使用的 requirements.yaml 文件。它被分为几个部分。

首先，有一组值定义了我们的集群。您应该能够通过查看其中的变量来了解它代表什么。您可能不会花费超过几分钟的时间就能看出，我们至少需要更改其中的一些值，所以这就是我们接下来要做的。

在您最喜欢的编辑器中打开 jx-requirements.yml 并更改以下值：

+   将 `cluster.clusterName` 设置为您的集群名称。它应该与环境变量 `CLUSTER_NAME` 的名称相同。如果您已经忘记了，请执行 `echo $CLUSTER_NAME`。

+   将 `cluster.environmentGitOwner` 设置为您的 GitHub 用户。它应该与之前声明的环境变量 `$GH_USER` 相同。

+   仅当您的 Kubernetes 集群运行在 GKE 上时，将 `cluster.project` 设置为您的 GKE 项目名称。否则，保持该值不变（为空）。

+   将 `cluster.provider` 设置为 `gke` 或 `eks` 或任何其他提供者，如果您决定您很勇敢并想尝试目前不受支持的平台。或者，自本章编写以来，事情可能已经改变，并且您的提供者现在确实受到支持。

+   将 `cluster.zone` 设置为您的集群正在运行的区域。如果您正在运行区域集群（您应该这样做），则值应该是区域，而不是区域。例如，如果您使用我们的 Gist 创建了 GKE 集群，则值应该是 `us-east1-b`。类似地，EKS 的值是 `us-east-1`。

我们已经完成了 `cluster` 部分，下一个是 `gitops` 值。它指导系统如何处理引导过程。将其更改为 `false` 没有意义，所以我们将保持它不变（true）。

下一个部分包含我们已经熟悉的环境的列表。键是后缀，最终名称将是 `environment-` 与集群名称的组合，后面跟着键。我们将保持它们不变。

`ingress` 部分定义了与集群外部访问相关的参数（域名、TLS 等）。

`kaniko` 值应该是显而易见的。当设置为 `true` 时，系统将使用 `kaniko` 而不是，比如说，Docker 来构建容器镜像。这是一个更好的选择，因为 Docker 不能在容器中运行，因此具有重大的安全风险（挂载套接字是邪恶的），并且它破坏了 Kubernetes 调度程序，因为它绕过了其 API。无论如何，`kaniko` 是使用 Tekton 构建容器镜像时唯一受支持的方式，所以我们将保持它不变（true）。

接下来，`secretStorage`当前设置为`local`。整个平台将在该仓库中定义，除了机密信息（如密码）。将它们推送到 Git 将是幼稚的，因此 Jenkins X 可以在不同的位置存储机密信息。如果你将其更改为`local`，那么该位置就是你的笔记本电脑。虽然这比 Git 仓库要好，但你可能可以想象为什么这不是正确的解决方案。将机密信息本地化会复杂化合作（它们只存在于你的笔记本电脑上），是易变的，并且仅比 Git 稍微安全一些。机密信息的一个更好的地方是 HashiCorp Vault。它是 Kubernetes（及其之外）中最常用的机密管理解决方案，Jenkins X 默认支持它。如果你已经设置了 Vault，可以将`secretStorage`的值设置为`vault`。否则，你可以保留默认值`local`。

在`secretStorage`值下方是整个部分，它定义了日志、报告和仓库的存储。如果启用，这些工件将存储在网络驱动器上。正如你所知道的那样，容器和节点是短暂的，如果我们想要保留任何这些，我们需要将它们存储在其他地方。这并不一定意味着网络驱动器是最好的地方，但这是默认的设置。稍后，你可能选择更改这一点，比如将日志发送到中央数据库，如 Elasticsearch、Papertrail、Cloudwatch、Stackdriver 等。

目前，我们将保持简单，并为所有三种类型的工件启用网络存储：

+   将`storage.logs.enabled`的值设置为`true`。

+   将`storage.reports.enabled`的值设置为`true`。

+   将`storage.repository.enabled`的值设置为`true`。

`versionStream`部分定义了包含 Jenkins X 所使用的所有包（图表）版本的仓库。你可能选择分叉该仓库并自行控制版本。在你跳入这样做之前，请注意 Jenkins X 的版本控制相当复杂，因为涉及许多包。除非你有很好的理由接管 Jenkins X 的版本控制并且准备好维护它，否则请保持现状。

正如你所知道的那样，Prow 仅支持 GitHub。如果你的 Git 提供商不是 GitHub，那么 Prow 就不适用。作为一个替代方案，你可以在 Jenkins 中设置它，但这也不是正确的解决方案。鉴于未来在 Tekton，Jenkins（没有 X）将不会得到长期支持。它仅在第一代 Jenkins X 中被使用，因为它是一个好的起点，并且几乎支持我们所能想象的一切。但社区已经接受 Tekton 作为唯一的管道引擎，这意味着静态 Jenkins 正在逐渐消失，并且它主要被用作那些习惯于“传统”Jenkins 的人的过渡解决方案。

所以，如果你不使用 GitHub，Prow 又不是一个选择，而 Jenkins 的日子又屈指可数，你该怎么办？更复杂的是，Prow 也将在未来的某个时刻（或者过去，取决于你阅读这篇文章的时间）被弃用。它将被 Lighthouse 取代，至少在开始时，它将提供与 Prow 相似的功能。与 Prow 相比，它的主要优势是 Lighthouse 将（或已经）支持所有主要的 Git 提供商（例如 GitHub、GitHub Enterprise、Bitbucket Server、Bitbucket Cloud、GitLab 等等）。在某个时刻，`webhook`的默认值将是`lighthouse`。但是，在撰写本文时（2021 年 2 月），情况并非如此，因为 Lighthouse 尚未稳定且尚未准备好投入生产。它很快就会准备好。无论如何，我们目前将继续使用 Prow 作为我们的 webhook。

只有在使用 EKS 时才执行以下命令。它们将添加与 Vault 相关的附加信息，即具有足够权限与之交互的 IAM 用户。确保用你的具有足够权限的 IAM 用户（总是作为管理员）替换`[...]`：

```
$ export IAM_USER=[...] # such as jx-boot
echo "vault:
  aws:
    autoCreate: true
    iamUserName: \"$IAM_USER\"" \
    | tee -a jx-requirements.yml
```

只有在使用 EKS 时才执行以下命令。jx-requirements.yaml 文件包含一个区域条目，对于 AWS，我们需要一个区域。此命令将替换一个为另一个：

```
$ cat jx-requirements.yml \
    | sed -e \
    's@zone@region@g' \
    | tee jx-requirements.yml
```

让我们看看现在的 jx-requirements.yml 文件看起来如何：

```
$ cat jx-requirements.yml
cluster:
  clusterName: "jx-boot"
  environmentGitOwner: "vfarcic"
  project: "devops-26"
  provider: gke
  zone: "us-east1"
gitops: true
environments:
- key: dev
- key: staging
- key: production
ingress:
  domain: ""
  externalDNS: false
  tls:
    email: ""
   enabled: false
    production: false
kaniko: true
secretStorage: vault
storage:
  logs:
    enabled: true
    url: ""
  reports:
    enabled: true
    url: ""
  repository:
    enabled: true
    url: ""
versionStream:
  ref: "master"
  url: https://github.com/jenkins-x/jenkins-x-versions.git
webhook: prow
```

现在，你可能担心我们遗漏了一些值。例如，我们没有指定域名。这意味着我们的集群将无法从外部访问吗？我们也没有指定存储的 URL。在这种情况下，Jenkins X 会忽略它吗？

事实是，我们只指定了我们知道的事情。例如，如果你使用我们的 Gist 创建了集群，那么就没有 Ingress，因此也就没有它本应创建的外部负载均衡器。结果，我们还不知道可以通过哪个 IP 地址访问集群，也无法生成.nip.io 域名。同样，我们也没有创建存储。如果我们创建了，我们就可以在 URL 字段中输入地址。

这些只是未知因素的一小部分。我们指定了我们知道的内容，我们将让 Jenkins X 的`boot`来找出未知因素。或者，更准确地说，我们将让`boot`创建缺少的资源，从而将未知因素转化为已知因素。

让我们安装 Jenkins X：

```
$ jx boot
```

现在我们需要回答很多问题。过去，我们试图通过指定所有答案作为我们执行的命令的参数来避免回答问题。这样，我们就有了记录在案的方法来做事情，而这些事情最终并没有进入 Git 仓库。其他人可以通过运行相同的命令来重现我们所做的一切。然而，这一次，我们不需要避免提问，因为我们将要做的所有事情都将存储在 Git 仓库中。

第一个输入要求输入逗号分隔的 Git 提供者用户名列表，这些用户名是开发环境存储库的审批者。这将创建一个用户列表，这些用户可以批准由 Jenkins X `boot`管理的开发存储库的拉取请求。现在，输入你的 GitHub 用户名并按 Enter 键。

我们可以看到，过了一会儿，我们收到了两条警告，指出 Vault 和 webhooks 没有启用 TLS。如果我们指定了一个“真实”的域名，`boot`将安装 Let’s Encrypt 并生成证书。但由于我们无法确定你手头是否有域名，我们没有指定它，因此我们不会得到证书。虽然这在生产环境中是不可接受的，但作为一个练习来说，这完全没问题。

由于那些警告，`boot`正在询问我们是否希望继续。输入`y`并按 Enter 键继续。

由于 Jenkins X 每天都会创建多个版本，所以你很可能没有`jx`的最新版本。如果是这种情况，`boot`会询问你是否想要升级到`jx`版本。按 Enter 键使用默认答案`Y`。结果，`boot`将升级 CLI，但会终止管道。这没关系。没有造成损害。我们只需要重复这个过程，但这次使用`jx`的最新版本：

```
$ jx boot
```

流程再次开始。我们将跳过对`jx boot`前几个问题的注释，并继续不使用 TLS。答案与之前相同（两种情况下都是`y`）。

接下来的一组问题与日志、报告和存储库的长期存储有关。对于所有三个问题都按 Enter 键，`boot`将创建具有自动生成的唯一名称的存储桶。

从现在开始，这个过程将创建机密并安装 CRDs（CustomResourceDefinitions），这些 CRDs 提供特定于 Jenkins X 的自定义资源。然后，它将安装 NGINX Ingress Controller（除非你的集群已经有一个），并将域名设置为.nip.io，因为我们没有指定一个。进一步来说，它将安装`cert-manager`，这将提供 Let’s Encrypt 证书。或者，更准确地说，如果指定了域名，它将提供证书。无论如何，它已经安装好了，以防我们改变主意，选择通过更改域名和稍后启用 TLS 来更新平台。

接下来是 Vault。`boot`将安装它并尝试用机密填充它。但由于它还不知道它们，这个过程将再次询问我们。这一组中的第一个问题是管理员用户名。请随意按 Enter 键接受默认值 admin。之后是管理员密码。输入你想要使用的任何密码（我们今天不需要它）。

此过程需要知道如何访问我们的 GitHub 仓库，因此它会询问我们的 Git 用户名、电子邮件地址和令牌。您可以使用您的 GitHub 用户名和电子邮件回答前两个问题。至于令牌，^(11)，您需要在 GitHub 中创建一个新的，并授予完整的仓库访问权限。最后，与 Secrets 相关的下一个问题是 HMAC 令牌。请随意按 Enter 键，过程将为您创建它。

最后一个问题。您想配置一个外部的 Docker 仓库吗？按 Enter 键使用默认答案（N），`boot` 将在集群内部创建它，或者在大多数云服务提供商的情况下，使用作为服务提供的仓库。在 GKE 的情况下，那就是 GCR；对于 EKS，那就是 ECR。在任何情况下，如果不配置外部 Docker 仓库，`boot` 将使用对特定提供商最有意义的选项：

```
? Jenkins X Admin Username admin
? Jenkins X Admin Password [? for help] ********
? The Git user that will perform git operations inside a pipeline. It should be a user within the Git organisation/own? Pipeline bot Git username vfarcic
? Pipeline bot Git email address vfarcic@gmail.com
? A token for the Git user that will perform git operations inside a pipeline. This includes environment repository creation, and so this token should have full repository permissions. To create a token go to https://github.com/settings/tokens/new?scopes=repo,read:user,read:org, user:email,write:repo_hook,delete_repo then enter a name, click Generate token, and copy and paste the token into this prompt.
? Pipeline bot Git token ****************************************
Generated token bb65edc3f137e598c55a17f90bac549b80fefbcaf, to use it press enter.
This is the only time you will be shown it so remember to save it
? HMAC token, used to validate incoming webhooks. Press enter to use the generated token [? for help] 
? Do you want to configure non default Docker Registry? No
```

剩余的过程将安装和配置平台的全部组件。我们不会详细介绍它们，因为它们与我们之前使用的相同。重要的是，系统将在一段时间后完全运行。

最后一步将验证安装。在过程的最后一步，您可能会看到一些警告。不要惊慌。`boot` 可能有些不耐烦。随着时间的推移，您会看到正在运行的 Pods 数量增加，而挂起的 Pods 数量减少，直到所有 Pods 都在运行。

就这些了。Jenkins X 现在已经启动并运行。我们已经将平台的完整定义（除了 Secrets）存储在 Git 仓库中：

```
verifying the Jenkins X installation in namespace jx
verifying pods
Checking pod statuses
POD                                          STATUS
jenkins-x-chartmuseum-774f8b95b-bdxfh        Running
jenkins-x-controllerbuild-66cbf7b74-twkbp    Running
jenkins-x-controllerrole-7d76b8f449-5f5xx    Running
jenkins-x-gcactivities-1594872000-w6gns      Succeeded
jenkins-x-gcpods-1594872000-m7kgq            Succeeded
jenkins-x-heapster-679ff46bf4-94w5f          Running
jenkins-x-nexus-555999cf9c-s8hnn             Running
lighthouse-foghorn-599b6c9c87-bvpct          Running
lighthouse-gc-jobs-1594872000-wllsp          Succeeded
lighthouse-keeper-7c47467555-c87bz           Running
lighthouse-webhooks-679cc6bbbd-fxw7z         Running
lighthouse-webhooks-679cc6bbbd-zl4bw         Running
tekton-pipelines-controller-5c4d79bb75-75hvj Running
Verifying the git config
Verifying username billyy at git server github at https://github.com
Found 2 organisations in git server https://github.com: IntuitDeveloper, intuit
Validated pipeline user billyy on git server https://github.com
Git tokens seem to be setup correctly
Installation is currently looking: GOOD
Using namespace 'jx' from context named 'gke_hazel-charter-283301_us-east1-b_cluster-1' on server 'https://34.73.66.41'.
```

## B.3 安装 Flux

Flux 由一个 CLI 客户端和运行在托管 Kubernetes 集群内部的守护进程组成。本节将解释如何安装 Flux CLI。守护进程的安装需要您指定带有访问凭证的 Git 仓库，这部分内容在第十一章中介绍。

### B.3.1 安装 CLI 客户端

Flux 分发包括名为 `fluxctl` 的 CLI 客户端。`fluxctl` 自动化 Flux 守护进程的安装，并允许您获取由 Flux 守护进程控制的 Kubernetes 资源的信息。

使用以下命令之一在 Mac、Linux 和 Windows 上安装 fluxctl。

macOS:

```
brew install fluxctl
```

Linux:

```
sudo snap install fluxctl
```

Windows:

```
choco install fluxctl
```

在官方安装说明中查找有关 `fluxctl` 安装细节的更多信息：[`docs.fluxcd.io/en/latest/references/fluxctl/`](https://docs.fluxcd.io/en/latest/references/fluxctl/)

* * *

1.[`github.com/argoproj/argo-helm/tree/master/charts/argo-cd`](https://github.com/argoproj/argo-helm/tree/master/charts/argo-cd).

2.[`github.com/argoproj-labs/argocd-operator`](https://github.com/argoproj-labs/argocd-operator).

3.[`minikube.sigs.k8s.io/docs/handbook/accessing/#loadbalancer-access`](https://minikube.sigs.k8s.io/docs/handbook/accessing/#loadbalancer-access).

4.[`argoproj.github.io/argo-cd/cli_installation/#download-with-curl`](https://argoproj.github.io/argo-cd/cli_installation/#download-with-curl).

5.[`kubernetes.io/docs/tasks/tools/install-kubectl/`](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

6.[`docs.helm.sh/using_helm/#installing-helm`](https://docs.helm.sh/using_helm/#installing-helm).

7.[`docs.helm.sh/using_helm/#installing-helm`](https://docs.helm.sh/using_helm/#installing-helm).

8.[`github.com/weaveworks/eksctl`](https://github.com/weaveworks/eksctl).

9.[`hub.github.com/`](https://hub.github.com/).

10.[`jenkins-x.io/docs/`](https://jenkins-x.io/docs/).

11.[`docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token`](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token).
