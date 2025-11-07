# 11 GitOps：配置即代码

本章涵盖

+   使用命名空间和配置文件来复制环境

+   将 Kubernetes 工作负载配置视为源代码的好处

+   使用 Git pull 请求来驱动操作

+   在版本控制中不存储明文形式的机密信息

你可能已经注意到，在这本书中，我们一直在编写大量的 YAML 配置文件。在不编写配置文件的情况下，你可以使用命令式 `kubectl` 命令（如 `kubectl run`）与大多数 Kubernetes 对象进行交互，这些命令的学习难度相对较低。那么，为什么我一直在使用基于声明的配置方法呢？一个原因是当你将应用程序部署到生产环境时，你可以开始将配置视为代码，使用版本控制和代码审查。

另一个原因是它允许你轻松地使用相同的配置启动多个环境。比如说，你想要一个尽可能相似的开发测试环境和生产环境，以便进行更好的测试。在你的工作负载定义在配置文件中时，可以轻松地复制这些环境。Kubernetes 的命名空间功能使得无需担心名称冲突即可实现这一点。

## 11.1 使用命名空间的生产和开发测试环境

当你为应用程序准备生产环境时，你可能会想要创建一个开发测试环境，以便在更新实时生产应用程序之前测试更改。Kubernetes 通过命名空间使这变得简单。

命名空间，正如其名所示，在单个逻辑空间内提供名称的唯一性。因此，你可以设置一个生产命名空间和一个开发测试命名空间，并在每个命名空间中设置 Deployment `foo-deployment` 和 Service `foo-service`（如图 11.1）。这避免了你需要过度修改配置以适应不同环境的需求，例如创建不同名称的 `foo-staging-deployment` 和 `foo-staging-service`，并且提供了一些防止意外更改的保护，因为默认情况下，`kubectl` 命令仅应用于当前活动的命名空间。

![11-01](img/11-01.png)

图 11.1 一个具有两个命名空间的 Kubernetes 集群。请注意，这些命名空间中的 Deployment 和 Service 对象名称相同。

你的生产环境和开发测试环境之间的主要配置差异通常是诸如规模（副本数量）和任何未作为命名空间一部分部署的服务的外部服务凭证等事项。

提示：随着你的应用程序工作负载的复杂性或不同配置环境的数量增加，你可能希望考虑使用模板引擎，如 Kustomize 或 Helm。

要创建一个名为 `staging` 的命名空间以托管应用程序的新实例，你可以运行

```
kubectl create namespace staging
```

要与此命名空间交互，你可以在运行的每个`kubectl`命令中添加`--namespace` `staging`（或简写为`-n` `staging`）或更改`kubectl`上下文，以便所有命令都在这个新命名空间中运行。我强烈推荐后者，因为你不想忘记`-n`标志并意外在错误的命名空间中运行命令。最好每次都切换上下文。你可以使用`kubectl` `get` `namespace`列出可用的命名空间，然后设置上下文为你的选择：

```
kubectl config set-context --current --namespace=staging
```

当列出命名空间时，你可能注意到 Kubernetes 自带了一些命名空间。`kube-system`是系统 Pod 所在的地方——除非你清楚自己在做什么，否则最好不要触碰这个。`default`是默认用户命名空间，使用它是可以的，但我建议创建你自己的专用命名空间，每个应用环境一个。

我觉得`kubectl`上下文设置命令很繁琐，并强烈推荐设置你的 shell，使用一个工具使其更简单。我使用的是`kubectx` + `kubens`。¹ 安装`kubens`后，你可以运行`kubens`来列出命名空间，并使用以下命令设置上下文：

```
kubens staging
```

另一个包含的实用工具`kubectx,`可以用来在完全不同的集群之间快速切换。这些脚本只是较长`kubectl` `config` `set-context`命令的简写，所以一旦设置了上下文，你就可以像平常一样使用`kubectl`。

### 11.1.1 部署到我们的新命名空间

一旦创建了命名空间，你就可以轻松地从配置中部署应用程序。这就是为什么这本书在每种情况下都使用配置文件的原因。你不必重新运行一系列强制命令来重新创建你的工作负载，你只需从包含你的配置文件的文件夹中运行以下命令：

```
kubectl apply -f .
```

如果你需要更改配置或将其部署到另一个命名空间，你只需每次重新运行该命令即可推出你的更改。

实际上，在 Kubernetes 中使用命名空间创建新环境非常简单，以至于如果你过去在其他平台上共享单个预发布环境，你可能看到拥有许多不同环境的一些好处。你可以为每个开发者或团队创建一个命名空间，一个用于预发布，另一个用于集成测试，等等。通常，命名空间是免费的（当然，复制你的 Pod 所使用的计算资源除外）。

### 11.1.2 从集群同步变更

但对于任何在配置之外强制执行的更改怎么办？也许你使用`kubectl` `scale`扩展了工作负载，使用`kubectl` `set-image`更改了镜像，或者使用`kubectl` `run`创建了 Deployment。这种情况是会发生的；我不会评判。

Kubernetes 允许你使用`--output`参数（简写为`-o`）在任意的`get`请求中查看和导出配置。例如，要获取 Deployment 的最新 YAML 配置，可以使用以下命令：

```
kubectl get deploy timeserver -o yaml                             ❶

kubectl get deploy timeserver -o yaml > timeserver-exported.yaml  ❷
```

❶ 以 YAML 格式查看 Deployment。

❷ 将 Deployment YAML 配置管道传输到文件。

但问题是 Kubernetes 添加了许多你实际上不希望在磁盘配置中出现的额外字段，比如状态消息等等。曾经有一个方便的 `--export` 选项可以去除这些字段，但遗憾的是它已被弃用。因此，确定哪些行可以删除以及哪些需要保留需要一点技巧。但你可以将这种方式得到的 YAML 文件与本书中的文件进行比较，以查看哪些行是重要的。

如果你计划在多个命名空间中使用配置，这是常见的，你肯定希望删除 `metadata` `→` `namespace` 字段。移除它将允许你在当前命名空间中部署配置（保留它将意味着任何更改都会更新在指定命名空间中的对象）。我看到保持命名空间的风险是你可能会意外地将一些配置设置在生产命名空间中的预发布文件夹中。第 11.3 节讨论了关于在不同命名空间中滚动部署的一些安全策略，但它依赖于*不*在资源对象中直接指定命名空间。

为了保持清洁，可以考虑移除的其它字段来自 `metadata` 部分，包括字段 `uid`、`resourceVersion`、`generation`、`creationTimestamp` 以及整个 `status` 部分。这些字段不会阻止你在其他命名空间或集群中重用配置，但它们在其部署上下文之外并没有实际意义，因此最好将其排除在版本控制之外，以避免混淆。

## 11.2 Kubernetes 方式的配置即代码

当你在源代码中遇到错误时，你可以检查版本历史记录以查看代码何时被更改（例如使用 `git log` 和 `git blame`），有时可能会回滚提交以回到之前的工作状态。当你将配置视为代码（通过将其提交到版本控制系统）时，你可以执行类似操作，但针对的是生产系统。

如果你有一个代码审查流程，你可以用同样的流程来审查 Kubernetes 配置。毕竟，配置对运行系统的影响和代码一样大。在配置仓库上进行代码审查可以帮助在它们被部署之前捕捉到错误。例如，如果你在配置更改中意外删除了 Deployment 的所有副本，你的同事在代码审查过程中有机会捕捉到这个错误。

你会发现这种模式在所有主要的互联网公司中都有应用。例如，大多数 Google 服务都是在单个代码仓库中开发和部署的，² 因此服务配置紧挨着代码。代码和服务配置遵循完全相同的代码审查实践，尽管可以批准合并更改的所有者（工程师）名单可能不同。

虽然没有义务像谷歌一样将配置存储在与代码相同的仓库中，但这主要是一个品味（以及无休止的技术辩论）的问题。我将在这里展示的用于在 Git 中存储 Kubernetes 配置的模型只是我找到对我有效的一个例子，但你应该根据自己的工程实践进行调整。

我使用单个 Git 仓库来表示单个集群中部署的所有 Kubernetes 对象。在这个仓库中，每个 Kubernetes 命名空间都有一个文件夹，这些文件夹中包含了命名空间中对象的 YAML 文件（图 11.2）。另一种选择是为每个命名空间使用单独的分支，这有一些很好的特性，比如能够将更改从预发布合并到生产。然而，由于可能有一些你不想合并的更改，这可能会变得混乱（例如，你不会希望不小心将仅用于预发布的更改合并到生产中）。

![11-02](img/11-02.png)

图 11.2 Git 仓库文件夹结构和与 Kubernetes 命名空间的关系

这里是一个示例目录布局：

```
/_debug        ❶
/_cluster      ❷
/staging       ❸
/production    ❸
```

❶ 任何你希望存储供所有开发者使用的调试脚本目录

❷ 集群配置（例如，命名空间配置文件）。这些文件仅在集群创建期间使用。

❸ 任何你希望存储供所有开发者使用的调试脚本目录

该仓库中的每个目录都映射到一个 Kubernetes 命名空间。这种 1:1 映射的优点是它允许你自信地执行`kubectl apply -f .`命令，将目录中的所有更改部署到活动命名空间。克隆环境就像复制整个文件夹并将其部署到其自己的命名空间一样简单。

对于较小规模的工作负载部署，共享多个环境命名空间的集群是很常见的。共享集群可以减少管理多个集群的直接成本和运营开销，并允许工作负载共享一个共同的计算资源池。随着部署规模的扩大，可能需要将环境分离到它们自己的集群中，以提供额外的访问控制和资源隔离级别（图 11.3）。好消息是配置仓库并不关心这些命名空间的位置；它们存在于不同的集群中是完全没问题的。

![11-03](img/11-03.png)

图 11.3 多集群环境配置的配置仓库

一旦你的配置仓库设置完成，开发过程看起来就像这样：

1.  对所需环境的配置进行更改。

1.  提交这些更改。

1.  通过设置当前命名空间上下文并运行`kubectl apply -f .`在匹配的目录中来更新实时状态。

通过这种方式，您正在遵循配置即代码的模式，但您还可以做更多。到目前为止的设置中存在的一个危险是您可能会意外地将配置从一个文件夹滚动到错误的命名空间。接下来的几节将介绍如何安全地滚动发布并避免这个问题，以及如何提升到完整的 GitOps 流程，其中配置仓库的 `git` `push` 自动触发滚动发布。

## 11.3 安全地滚动发布

当您的配置作为代码仓库设置好后，现在有一个问题，那就是如何最好地在仓库中滚动更改。当然，您可以简单地检出仓库并运行 `kubectl` `apply` `-f` `.`，就像我们之前做的那样，但这可能很危险。您可能会意外地将错误的配置部署到错误的命名空间。由于我们在多个环境中重复使用对象名称，这确实可能很糟糕。此外，没有阻止您直接更改实时集群状态而不将配置更改提交到仓库。

要解决错误的命名空间问题，我建议设置一些防护措施以避免意外将错误的配置部署到错误的命名空间。我们之前简单地运行 `kubectl` `apply` `-f` `.`，现在将其包装在一个脚本中，该脚本执行检查以确保您正在部署到正确的命名空间。如果我们命名文件夹与命名空间相同，那么检查就很简单：如果当前命名空间等于文件夹名称，则部署；否则，不部署。以下列表提供了一个示例脚本，该脚本比较当前目录名称与当前命名空间，如果不匹配则退出错误状态。

列表 11.1 第十一章/gitops/gitops_check.sh

```
#! /bin/bash

CURRENT_DIR=`echo "${PWD##*/}"`                                      ❶
CURRENT_NAMESPACE=`kubectl config view --minify                      ❷
➥ -o=jsonpath='{.contexts[0].context.namespace}'`                   ❷

if [ "$CURRENT_DIR" != "$CURRENT_NAMESPACE" ]; then                  ❸
    >&2 echo "Wrong namespace (currently $CURRENT_NAMESPACE but" \   ❸
             "$CURRENT_DIR expected)"                                ❸
    exit 1                                                           ❸
fi                                                                   ❸

exit 0                                                               ❹
```

❶ 获取当前路径的最后一个目录组件。

❷ 获取当前命名空间。

❸ 如果此脚本从与命名空间名称不匹配的目录中运行，则退出错误。

❹ 否则，以成功状态退出。

然后，您可以在任何其他脚本中使用这个脚本，如下面的滚动发布脚本所示。您不会直接运行 `kubectl` `apply` `-f` `.`，而是运行这个脚本，该脚本会验证正确的目录/命名空间组合。

列表 11.2 第十一章/gitops/production/rollout.sh

```
#! /bin/sh

if [ $(../gitops_check.sh; echo $?) != 0 ]; then exit 1; fi   ❶

kubectl apply -f .                                            ❷
```

❶ 验证目录名称是否与命名空间匹配。

❷ 正常运行 kubectl apply。

在 *第十一章/gitops* 中的示例中提供了一个完整的 GitOps 文件夹结构，包括这些脚本。

当然，这并不是唯一的选择。另一种方法是在您的滚动发布脚本中设置所需的命名空间，然后进行部署。只是确保如果 `set` `namespace` 步骤失败，整个过程将退出。

虽然这些脚本可以工作，但您需要确保您的配置文件中没有直接指定 `metadata` `→` `namespace` 字段。如果它们设置了命名空间，它将忽略当前上下文，因此脚本不会阻止该情况下的更新。

要真正遵循 GitOps 方法，你将需要添加一个额外的保证，即始终部署的配置与存储库中实际存在的配置相匹配。解决这个问题的最佳方法是完全移除人工干预，并配置一个部署管道或 GitOps 操作员。

### 11.3.1 部署管道

部署管道简单来说是一组基于代码存储库触发器运行的函数——例如，“当代码被推送到配置存储库时，将配置部署到 Kubernetes 集群”（图 11.4）。使用管道可以保证正在部署的配置与提交的内容相匹配。如果操作员在部署后需要做出额外的更改（例如，更正错误），他们将在配置代码存储库中像平常一样进行更改：推送更改并再次触发管道部署。

![11-04](img/11-04.png)

图 11.4 Kubernetes 的持续部署管道

配置好你的管道后，你可以通过将 Git 存储库中的代码合并到生产环境中来推送生产内容（即 Git 驱动的操作，或 GitOps）。关键是不直接在集群上做出任何更改；所有更改都通过配置存储库和持续部署管道进行。

### 11.3.2 使用 Cloud Build 进行持续部署

在实践中实现部署管道，市场上有很多产品。对于 Google Kubernetes Engine (GKE)用户来说，一个选项是 Cloud Build。你可以设置一个触发器，以便当你的配置存储库被推送到时，它会运行`kubectl` `apply` `-f` `.`。

要设置它，请按照以下步骤操作：

1.  为 Cloud Build 服务账户配置 IAM 权限³，以授予其在你的 GKE 集群上执行操作的权限。

1.  创建一个新的触发器（设置为在配置存储库被推送到时触发）。

在你的存储库中添加一个 Cloud Build 配置文件，例如以下列表中的文件，并在触发器中引用它。

列表 11.3 第十一章/11.3.2_CloudBuild/cloudbuild-deploy.yaml

```
steps:
- name: 'gcr.io/cloud-builders/kubectl'
  id: Deploy
  args:
  - 'apply'
  - '-f'
  - '$FOLDER'
  env:
  - 'CLOUDSDK_COMPUTE_REGION=us-west1'
  - 'CLOUDSDK_CONTAINER_CLUSTER=my-cluster'
```

这只是对持续交付的表面探讨。如果你使用 Cloud Build，你可以参考优秀的“使用 Cloud Build 进行 GitOps 风格的持续交付”指南⁴，它进一步深入并设置了一个完整的端到端 CI/CD 流程。

持续校准

这里描述的方法可以通过使用 GitOps 操作员进一步改进。这是一个在集群中运行的控制系统循环，它不断将集群中运行的内容与配置存储库中存在的内容进行校准。最终结果是类似于之前描述的事件驱动管道，其优点是当出现偏差时可以执行额外的校准，而管道方法依赖于 Git 推送事件来触发。Flux ([`fluxcd.io/`](https://fluxcd.io/))就是这样一种 GitOps 操作员。

## 11.4 密钥

Git 仓库是一个存储 Kubernetes 配置的好地方，但有一些数据可能不应该存储在那里：如数据库密码和 API 密钥这样的机密值。如果这些机密被嵌入到代码本身或环境变量中，这意味着任何可以访问你的源代码的人都可以获得这些机密。一个改进的方法是，只有那些可以访问你的生产系统的人才能访问这些数据。当然，你可以更进一步，但在本章关于 GitOps 的背景下，我将专注于如何将你的机密与代码和配置仓库分离。

Kubernetes 实际上有一个用于存储机密的对象，恰当地命名为 Secrets。这些对象提供了一种方式，可以将诸如凭证和密钥等信息以与工作负载本身配置分离的方式提供给工作负载。

### 11.4.1 基于字符串（密码）的机密

如果你一直将密码等机密像明文环境变量一样嵌入到工作负载配置中，现在就是迁移到 Secrets 的好时机。假设我们有一个值为`secret_value`的机密（实际上，这可能是从你的云服务提供商那里获得的一个密钥）。我们可以将我们的`secret_value`封装到一个 Kubernetes Secret 对象中，如下所示。

列表 11.4 Chapter11/11.4.1_StringSecrets/secret.yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: secrets-production
type: Opaque
stringData:
  SECRET_KEY: secret_value
  ANOTHER_KEY: another_secret_value
```

机密可以作为文件挂载到容器中或作为环境变量提供给 Pods。你会使用文件方法来访问作为配置文件（例如，私有 SSL 密钥）的机密数据，而使用环境变量来处理数据库密码等项。由于列表 11.4 中的机密是一个简单的字符串，我们将使用环境变量方法在下面的列表中引用它（有关基于文件的机密示例，请参阅 11.4.3 节）。

列表 11.5 Chapter11/11.4.1_StringSecrets/deploy.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: timeserver
spec:
  replicas: 1
  selector:
    matchLabels:
      pod: timeserver-pod
  template:
    metadata:
      labels:
        pod: timeserver-pod
    spec:
      containers:
      - name: timeserver-container
        image: docker.io/wdenniss/timeserver:5
        env:
        - name: AVATAR_ENDPOINT            ❶
          value: http://robohash-internal  ❶
       - name: SECRET_KEY ❷
 valueFrom: ❷
 secretKeyRef: ❷
 name: secrets-production ❷
 key: SECRET_KEY ❷
```

❶ 一个普通的纯文本环境变量

❷ 从机密中填充的环境变量

为了验证一切是否正确工作，请在 Chapter11/11.4.1_StringSecrets 文件夹中创建 Secret 和 Deployment 对象，并运行

```
$ kubectl exec deploy/timeserver -- env
SECRET_KEY=secret_value
```

你应该在输出中看到这个机密。我们的应用程序现在可以通过`SECRET_KEY`环境变量访问这个机密。

### 11.4.2 Base64 编码的机密

你可能会遇到 Kubernetes 文档和其他资源，它们展示了机密值以 base64 编码的形式（使用`data`字段而不是`stringData`）。这不是为了安全起见（base64 是一种编码，而不是加密），而是为了能够表示那些难以在 YAML 中定义的数据。

我通常不会默认使用 base64 编码密钥，因为我觉得这主要只是模糊了数据，并没有增加多少价值。然而，如果你有一个难以在 YAML 中表示的字符串，比如你的密钥是一个完整的文件，那么对数据进行 base64 编码是有意义的。以下列表提供了 11.4 列表中显示的 `SECRET_KEY` 密钥的等效 base64 编码表示。

列表 11.6 第十一章/11.4.2_Base64Secrets/secret-base64.yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: secrets-production
type: Opaque
data:
  SECRET_KEY: c2VjcmV0X3ZhbHVlCg==
```

要进行编码和解码，在任何类 Unix 系统上，你可以执行以下操作：

```
$ echo "secret_value" | base64
c2VjcmV0X3ZhbHVlCg==

$ echo "c2VjcmV0X3ZhbHVlCg==" | base64 -d
secret_value
```

如果你有一些需要 base64 编码的值和一些不需要的值，你可以在同一个配置文件中包含 `data` 和 `stringData`。你还可以在每个 Kubernetes Secret 对象中存储多个密钥（每行一个）。以下列表提供了一个示例，定义了三个密钥，其中两个使用纯文本，一个使用 base64。

列表 11.7 第十一章/11.4.2_Base64Secrets/secrets-multiple.yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: secrets-production
type: Opaque
stringData:
  SECRET_KEY: secret_value
  ANOTHER_KEY: another_value
data:
  ENCODED_KEY: 
    VGhpcyBzdHJpbmcKbWlnaHQgYmUgaGFyZCB0byByZXByZXNlbnQgaW4gWUFNTCDwn5iFCg==
```

如果你通过命令行从服务器检索密钥，你会得到 base64 编码的形式，并且需要解码它们以查看纯文本值（然而，它们已经解码并提供给应用程序代码）。

我个人为每个命名空间创建一个密钥对象，每个对象包含多个密钥。然而，我将它们存储在与配置文件的其他部分分开的仓库中。在第 11.4.4 节中，我将讨论一些选项，如何在仍然使用 GitOps 方法的同时，将密钥存储在主配置仓库之外。

### 11.4.3 基于文件的密钥

有时你可能会处理一些密钥，你希望将它们作为文件而不是环境变量的字符串从应用程序中访问。Kubernetes 在这里也为你提供了支持。创建密钥实际上是一样的，但我将提供一个多行文本文件的全新示例，因为这种数据在 YAML 中的表示有一些细微差别。

假设我们需要存储一个私钥。以下是一个使用 `openssl genrsa 256 -out example.key` 生成的示例（通常你会使用 2048 位或更高的密钥，但为了简洁，我将使用 256 位）：

列表 11.8 第十一章/11.4.3_ 文件密钥/example.key

```
-----BEGIN RSA PRIVATE KEY-----
MIGsAgEAAiEA4TneQFg/UMsVGrAvsm1wkonC/5jX+ykJAMeNffnlPQkCAwEAAQIh
ANgcs+MgClkXFQAP0SSvmJRmnRze3+zgUbN+u+rrYNRlAhEA+K0ghKRgKlzVnOxw
qltgTwIRAOfb8LCVNf6FAdD+bJGwHycCED6YzfO1sONZBQiAWAf6Am8CEQDIEXI8
fVSNHmp108UNZcNLAhEA3hHFV5jZppEHHHLy4F9Dnw==
-----END RSA PRIVATE KEY-----
```

该文件的数据可以用以下方式在 YAML 中表示。注意至关重要的管道字符，它将保留数据值中的行结束符。

列表 11.9 第十一章/11.4.3_ 文件密钥/secret_file.yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: secret-files
type: Opaque
stringData:
  example.key: |
    -----BEGIN RSA PRIVATE KEY-----
    MIGsAgEAAiEA4TneQFg/UMsVGrAvsm1wkonC/5jX+ykJAMeNffnlPQkCAwEAAQIh
    ANgcs+MgClkXFQAP0SSvmJRmnRze3+zgUbN+u+rrYNRlAhEA+K0ghKRgKlzVnOxw
    qltgTwIRAOfb8LCVNf6FAdD+bJGwHycCED6YzfO1sONZBQiAWAf6Am8CEQDIEXI8
    fVSNHmp108UNZcNLAhEA3hHFV5jZppEHHHLy4F9Dnw==
    -----END RSA PRIVATE KEY-----
```

如果你现在已经厌倦了与 YAML 语法搏斗，你可以通过使用 `cat example.key | base64` 对文件数据进行 base64 编码，并像以下列表（为了可读性，数据已截断）那样表示。注意，整个 base64 字符串放在一行上（没有换行符！）。

列表 11.10 第十一章/11.4.3_ 文件密钥/secret_file_base64.yaml

```
apiVersion: v1
kind: Secret
metadata:
  name: secret-files
type: Opaque
data:
  example.key: LS0tLS1CRUdJTiBSU0EgUFJJVk...U0EgUFJJVkFURSBLRVktLS0tLQo=
```

手动创建这些密钥配置文件确实有点繁琐。一个更自动化的方法是使用 `kubectl` 为你创建文件。以下命令将创建相同的功能输出（注意，为了可读性，base64 字符串已截断）：

```
$ cd Chapter11/11.4.3_FileSecrets
$ kubectl create secret generic secret-files \
      --from-file=example.key=./example.key --dry-run=client -o yaml

apiVersion: v1
data:
  example.key: LS0tLS1CRUdJTiBSU0EgUFJJVk...U0EgUFJJVkFURSBLRVktLS0tLQo=
kind: Secret
metadata:
  creationTimestamp: null
  name: secret-files
```

`--dry-run=client` `-o` `yaml`部分的意思是实际上你不会在服务器上创建机密，而是将其作为 YAML 输出（供你放入配置文件中，稍后用`kubectl apply -f secret.yaml`应用到服务器上）。省略`--dry-run`将直接在集群上创建机密（即创建 Kubernetes 对象的命令式风格）。实际上，本节中给出的每个示例都可以写成命令式的`kubectl`命令，但声明式、配置驱动的集群操作方法具有持久的好处，包括本章前面提到的那些。

一旦创建，你就可以将 Secret 中的所有文件挂载为容器中的一个文件夹。以下列表将我们的`secret-files` Secret 挂载到位置`/etc/config`。每个数据键都挂载为其自己的文件。在我们的例子中，只有一个：`example.key`。

列表 11.11 第十一章/11.4.3_ 文件机密/deploy.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: timeserver
spec:
  replicas: 1
  selector:
    matchLabels:
      pod: timeserver-pod
  template:
    metadata:
      labels:
        pod: timeserver-pod
    spec:
      containers:
      - name: timeserver-container
        image: docker.io/wdenniss/timeserver:5
        volumeMounts:
        - name: secret-volume
          mountPath: "/etc/config"   ❶
          readOnly: true
      volumes:
      - name: secret-volume          ❷
        secret:                      ❷
          secretName: secret-files   ❷
```

❶ 文件 secrets 将被挂载的路径。

❷ 对 Secret 对象的引用

为了验证一切是否正确工作，创建 Secret 和 Deployment 对象，并使用`exec`列出目录。你应该看到我们的文件`example.key`：

```
$ kubectl exec deploy/timeserver -- ls /etc/config
example.key
```

要查看文件本身，将`ls`命令替换为`cat /etc/config/example.key`。现在你可以将代码指向这个文件，就像指向系统上的任何其他文件一样。

### 11.4.4 Secrets 和 GitOps

使用 Secrets 只是方程的一部分。现在，你需要弄清楚如何存储它们。如果你将它们放在同一个配置仓库中，那么你实际上可以直接使用普通的环境变量，并跳过上一节中的步骤。这个问题没有银弹，但这里有一些想法，按复杂度递增的顺序列出。

独立仓库

一个简单的选择是为你的机密信息创建一个独立的配置仓库，其用户权限少于常规仓库。你仍然可以享受配置代码的所有好处（例如，代码审查、回滚等），但可以限制受众。如果你操作一个具有细粒度访问控制的仓库，你可以在该仓库的一个受控文件夹中放置机密信息。

这个仓库的一个合理位置是与你的云提供商的生产资源一起，具有与生产环境相同的安全控制。由于任何有权访问你的生产环境的人实际上已经有了这些机密，因此这种模型不会为账户被破坏的人提供任何额外的访问权限。

密封机密

Sealed Secrets⁵ 项目采用了一种有趣的方法：你在将秘密存储到 Git 之前对其进行加密（这样就没有人能读取它们），然后由集群中的控制器使用私钥进行解密。虽然你最终还是面临着存储私钥的问题，但这意味着加密的秘密可以包含在主配置仓库中，从而获得所有相关的优势，例如回滚。

秘密服务

另一个选项是运行一个独立的服务，该服务可以将秘密注入到你的集群中。HashiCorp 的 Vault⁶ 是这个概念的流行实现，如果你希望自行运行，它也是开源的。

## 摘要

+   使用命名空间来区分不同的环境，如生产环境和预发布环境，以及不同的应用程序。

+   将配置视为代码允许你轻松地复制和维护多个环境。

+   通过将 Kubernetes 配置存储在版本控制中，就像存储代码一样（包括进行代码审查，如果你这样做的话），遵循配置即代码的方法论。

+   不要直接对实时集群进行更改，而是首先更改配置，提交到版本控制，然后应用这些更改。

+   可以使用部署管道在将更改提交并推送到配置仓库时自动部署这些更改。

+   可以使用像 Flux 这样的 GitOps 操作员来提供配置仓库的持续协调。

+   使用 Kubernetes Secrets 存储敏感信息，如数据库密钥，在单独的 Kubernetes 对象中。以限制访问的方式存储该配置。

* * *

[`github.com/ahmetb/kubectx`](https://github.com/ahmetb/kubectx)

[`research.google/pubs/pub45424/`](https://research.google/pubs/pub45424/)

[`cloud.google.com/build/docs/securing-builds/configure-access-for-cloud-build-service-account`](https://cloud.google.com/build/docs/securing-builds/configure-access-for-cloud-build-service-account)

[`cloud.google.com/kubernetes-engine/docs/tutorials/gitops-cloud-build`](https://cloud.google.com/kubernetes-engine/docs/tutorials/gitops-cloud-build)

[`github.com/bitnami-labs/sealed-secrets`](https://github.com/bitnami-labs/sealed-secrets)

[`www.vaultproject.io/`](https://www.vaultproject.io/)
