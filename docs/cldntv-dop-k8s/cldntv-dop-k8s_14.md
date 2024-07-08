# 第十二章：部署 Kubernetes 应用程序

> 我躺在背上，惊讶地感受到自己是如何平静和专注的，绑在四百五十万磅的炸药上。
> 
> 罗恩·加兰，宇航员

在本章中，我们将讨论如何将您的清单文件转换为运行中的应用程序。我们将学习如何为您的应用程序构建 Helm 图表，并查看一些替代清单管理工具：Tanka、kustomize、Kapitan 和 kompose。

# 使用 Helm 构建清单

我们在 第二章 中看到如何使用从 YAML 清单创建的 Kubernetes 资源部署和管理应用程序。您完全可以使用这种方式管理所有 Kubernetes 应用程序的原始 YAML 文件，但这并不理想。维护这些文件不仅困难，而且存在分发问题。

假设您想要让其他人在他们自己的集群中运行您的应用程序。您可以向他们分发清单文件，但他们必然需要根据自己的环境定制一些设置。

为了做到这一点，他们将不得不复制 Kubernetes 配置的自己的副本，找到定义各种设置的位置（可能在几个地方重复），并对其进行编辑。

随着时间的推移，他们将需要维护文件的自己的副本，并在您提供更新后，他们将不得不手动拉取并与其本地更改进行协调。

最终，这开始变得痛苦起来。我们希望能够将原始清单文件与您或应用程序任何用户可能需要调整的特定设置和变量分离开来。理想情况下，我们可以将这些内容以标准格式提供给任何人下载并安装到 Kubernetes 集群中。

一旦我们拥有了这些，那么每个应用程序都可以暴露出不仅是配置值，还包括它对其他应用程序或服务的任何依赖关系。一个智能的包管理工具可以通过单个命令安装和运行应用程序及其所有依赖项。

在 “Helm：一个 Kubernetes 包管理器” 中，我们介绍了 Helm 工具，并展示了如何使用它安装公共图表。现在让我们更详细地看看 Helm 图表，以及如何创建我们自己的图表。

## Helm 图表的内部构成是什么？

在演示资料库中，打开 *hello-helm3/k8s* 目录，看看我们的 Helm 图表里面有什么。

每个 Helm 图表都有一个标准结构。首先，图表包含在一个与图表同名的目录中（在这种情况下是 `demo`）：

```
demo
├── Chart.yaml
├── production-values.yaml
├── staging-values.yaml
├── templates
│   ├── deployment.yaml
│   └── service.yaml
└── values.yaml
```

### Chart.yaml 文件

接下来，它包含一个名为 *Chart.yaml* 的文件，其中指定了图表的名称和版本：

```
name: demo
sources:
  - https://github.com/cloudnativedevops/demo
version: 1.0.1
```

*Chart.yaml* 中有许多可选字段供您提供，包括指向项目源代码的链接，如此处所示，但唯一必需的信息是名称和版本。

### *values.yaml* 文件

还有一个名为 *values.yaml* 的文件，其中包含图表作者公开的可修改用户设置：

```
environment: development
container:
  name: demo
  port: 8888
  image: cloudnatived/demo
  tag: hello
replicas: 1
```

这看起来有点像 Kubernetes YAML 清单，但有一个重要的区别。*values.yaml* 文件是完全自由格式的 YAML，没有预定义的模式：由您选择定义什么变量，它们的名称和值。

您的 Helm chart 中完全可以没有任何变量，但如果有的话，您可以将它们放在 *values.yaml* 中，然后在图表的其他地方引用它们。

暂时忽略 *production-values.yaml* 和 *staging-values.yaml* 文件；我们稍后会解释它们的用途。

## Helm 模板

那么这些变量被引用在哪里？如果你看 *templates* 子目录，你会看到几个看起来很熟悉的文件：

```
`ls k8s/demo/templates`
deployment.yaml service.yaml
```

这些与前面示例中的部署和服务清单文件完全相同，只是现在它们是*模板*：不再直接引用诸如容器名称之类的东西，而是包含一个 Helm 将从 *values.yaml* 中实际值替换的占位符。

下面是模板部署的样子：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.container.name }}-{{ .Values.environment }}
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: {{ .Values.container.name }}
  template:
    metadata:
      labels:
        app: {{ .Values.container.name }}
        environment: {{ .Values.environment }}
    spec:
      containers:
        - name: {{ .Values.container.name }}
          image: {{ .Values.container.image }}:{{ .Values.container.tag }}
          ports:
            - containerPort: {{ .Values.container.port }}
          env:
            - name: ENVIRONMENT
              value: {{ .Values.environment }}
```

###### 提示

花括号表示 Helm 应该替换变量值的位置，但它们实际上是 *Go 模板语法* 的一部分。

（没错，Go 到处都是。Kubernetes 和 Helm 本身都是用 Go 编写的，所以 Helm charts 使用 Go 模板并不奇怪。）

## 插值变量

此模板中引用了多个变量：

```
...
metadata:
  name: `{``{` `.Values.container.name` `}``}``-{{` `.Values.environment` `}}`
```

这整段文字，包括花括号，都将被*插值*（即替换）为从 *values.yaml* 中取得的 `container.name` 和 `environment` 的值。生成的结果将看起来像这样：

```
...
metadata:
  name: `demo-development`
```

这非常强大，因为诸如 `container.name` 这样的值在模板中被多次引用。当然，它也在 Service 模板中被引用：

```
apiVersion: v1
kind: Service
metadata:
  name: `{``{` `.Values.container.name` `}``}`-service-{{ .Values.environment }}
  labels:
    app: `{``{` `.Values.container.name` `}``}`
spec:
  ports:
  - port: {{ .Values.container.port }}
    protocol: TCP
    targetPort: {{ .Values.container.port }}
  selector:
    app: `{``{` `.Values.container.name` `}``}`
  type: ClusterIP
```

例如，您可以看到 `.Values.container.name` 被引用了多少次。即使在这样一个简单的图表中，您需要多次重复相同的信息片段。使用 Helm 变量可以消除这种重复。例如，要更改容器名称，您只需编辑 *values.yaml* 并重新安装图表，更改将在所有模板中传播。

Go 模板格式非常强大，你可以用它来做远不止简单的变量替换：它支持循环、表达式、条件语句，甚至调用函数。Helm charts 可以利用这些特性从输入值生成相当复杂的配置，不像我们示例中的简单替换。

您可以在 Helm [文档](https://oreil.ly/4u06b)中阅读更多有关如何编写 Helm 模板的信息。

## 引用模板中的值

您可以在 Helm 中使用 `quote` 函数自动引用模板中的值：

```
name: {{.Values.MyName | quote }}
```

只有字符串值应该用引号括起来 —— 不要对数值（比如端口号）使用 `quote` 函数。

## 指定依赖关系

如果你的图表依赖于其他图表怎么办？例如，如果你的应用程序使用 Redis，那么你的应用 Helm 图表可能需要指定`redis`图表作为一个依赖项。

你可以使用`Chart.yaml`中的`dependencies`部分来实现这一点：

```
dependencies:
  - name: redis
    version: ~15.4.1
    repository: https://charts.bitnami.com/bitnami
```

现在运行`helm dependency update`命令，Helm 将下载那些图表，准备与你自己的应用程序一起安装。这些依赖项可以是本地图表，也可以是托管在公共 Helm 存储库中的图表。

你也可以覆盖任何作为依赖项引入的图表的默认值（通常称为子图表）。有关更多详情，请参阅[Helm 文档上的子图表值](https://oreil.ly/rF4rq)。

版本中的`~`符号告诉 Helm，你愿意自动升级到图表的更新版本，直到下一个次要版本发布。例如，`~15.4.1`表示 Helm 将安装最新的`15.4.x`版本，在`15.5.x`之前停止。Helm，像许多其他包管理器一样，还使用.lock 文件，因此更新后，你应该看到生成一个*Chart.lock*文件，记录了当时使用的确切版本。

# 部署 Helm 图表

现在让我们看看如何实际使用 Helm 图表来部署应用程序。Helm 最有价值的功能之一是能够指定、更改、更新和覆盖配置设置。在本节中，我们将看到它的工作原理。如果你想跟着做示例，这个例子在[cloudnativedevops GitHub 演示库](https://oreil.ly/xeLdl)的目录中。

## 设置变量

我们已经看到，Helm 图表的作者可以将所有用户可修改的设置放在`values.yaml`中，以及这些设置的默认值。那么，一个图表的*用户*如何改变或覆盖这些设置以适应他们的本地站点或环境呢？`helm install`和`helm upgrade`命令允许你在命令行上指定额外的值文件，这些文件将覆盖*values.yaml*中的任何默认值。让我们看一个例子。

### 创建环境变量

假设你想在一个分段环境中部署应用程序的一个版本。对于我们的示例目的，实际上这意味着什么并不重要，但假设应用程序根据名为`ENVIRONMENT`的环境变量的值知道它是在分段环境还是在生产环境，并相应地改变其行为。那个环境变量是如何创建的？

再次查看*deployment.yaml*模板，这个环境变量是通过以下代码提供给容器的：

```
...
env:
  - name: ENVIRONMENT
    value: {{ .Values.environment }}
```

`environment`的值来自*values.yaml*，这是你所期望的：

```
environment: development
...
```

因此，使用默认值安装图表将导致容器的`ENVIRONMENT`变量包含`development`。假设你想将其更改为`staging`。你可以像我们看到的那样编辑*values.yaml*文件，但更好的方法是创建一个额外的 YAML 文件，仅包含该变量的一个值：

```
environment: staging
```

您会在 *k8s/demo/staging-values.yaml* 文件中找到这个值，该文件不是 Helm 图表的一部分——我们只是提供它来节省一点打字时间。

## 在 Helm 发布中指定值

要在`helm upgrade`命令中指定额外的值文件，请使用`--values`标志，如下所示：

```
`helm upgrade --install demo-staging` `--values=./k8s/demo/staging-values.yaml ./k8s/demo`
```

###### 注意

请注意，在这些示例中，我们使用`helm upgrade --install`而不是`helm install`。这样做可以让您无论 Helm 发布是否已安装，都可以使用相同的命令。如果您更喜欢首先使用`helm install`，然后在后续部署时使用`helm upgrade`，那也可以。

这将创建一个新的发布，名称为`demo-staging`，运行容器的`ENVIRONMENT`变量将设置为`staging`而不是`development`。我们指定的额外值文件中列出的变量（使用`--values`标志）与默认值文件（*values.yaml*）中的变量组合在一起。在这种情况下，只有一个变量（`environment`），*staging-values.yaml*中的值将覆盖默认值文件中的值。

你还可以直接在命令行上使用`--set`标志指定数值。这对于诸如更改镜像标签版本或测试快速更改等情况非常有用。通常，你应该创建一个单独的 YAML 文件，包含特定环境所需的任何值覆盖，例如示例中的 *staging-values.yaml* 文件，并将该文件跟踪到源代码控制存储库中，然后使用`--values`标志应用它。

虽然您自然会希望以这种方式设置配置值来安装自己的 Helm 图表，但您也可以对公共图表进行操作。要查看图表提供的可设置值列表，请运行带有图表名称的`helm get values`：

```
`helm inspect values ./k8s/demo`
```

## 使用 Helm 更新应用程序

你已经学会了如何使用默认值和自定义值文件安装 Helm 图表，但是如何更改某些正在运行的应用程序的值呢？

`helm upgrade`命令将为您执行此操作。假设您希望更改演示应用程序的副本数（Kubernetes 应运行的 Pod 的副本数）。默认情况下为 1，您可以从 *values.yaml* 文件中看到：

```
replicas: 1
```

您已经知道如何使用自定义值文件覆盖此设置，因此编辑 *staging-values.yaml* 文件以添加适当的设置：

```
environment: staging
`replicas``:` `2`
```

运行与之前相同的命令，将您的更改应用于*现有*的`demo-staging`部署，而不是创建一个新的部署：

```
`helm upgrade --install demo-staging` `--values=./k8s/demo/staging-values.yaml ./k8s/demo`
Release "demo-staging" has been upgraded. Happy Helming!
```

您可以随意运行`helm upgrade`多次以更新正在运行的部署，Helm 将乐意为您效劳。

## 回滚到之前的版本

如果你决定不喜欢刚刚部署的版本，或者出现了问题，使用`helm rollback`命令回滚到之前的版本非常容易，只需指定之前发布的编号（如`helm history`输出中所示）即可：

```
`helm rollback demo-staging 1`
Rollback was a success! Happy Helming!
```

实际上，回滚不一定要回到先前的版本；假设你回滚到版本 1，然后决定要向前回滚到版本 2。如果运行`helm rollback demo-staging 2`，这正是会发生的事情。

### 使用 helm 进行自动回滚

你可以让 Helm 自动回滚一个不成功的部署。如果在`helm upgrade`命令中加入`--atomic`标志，Helm 将等待你的部署成功。如果进入`FAILED`状态，它将自动回滚到上一个成功的发布。你应该确保在此情况下设置了失败部署的警报，以便你能够调试发生了什么，否则你可能不会注意到你的部署实际上并没有成功！

## 创建 Helm Chart 库

到目前为止，我们已经使用 Helm 从本地目录安装了图表。你不需要拥有自己的图表库来使用 Helm，因为将应用程序的 Helm Chart 存储在应用程序自己的库中是很常见的。

但是，如果你确实想要维护自己的 Helm chart 库，这非常简单。图表需要通过 HTTP 访问，并且有多种方式可以实现这一点：将它们放在云存储桶中、托管自己的[ChartMuseum 服务器](https://chartmuseum.com)，使用 Artifactory，使用 GitHub Pages，或者如果有现成的 Web 服务器，则可以使用现有的 Web 服务器。

一旦所有图表都集中在一个单独的目录下，运行`helm repo index`命令来创建包含库元数据的*index.yaml*文件。

你的图表库已经准备就绪！详细了解管理图表库的更多细节，请参阅 Helm 的[文档](https://oreil.ly/glEyY)。

要从你的库中安装图表，首先需要将库添加到 Helm 的列表中：

```
`helm repo add myrepo http://myrepo.example.com`
`helm install myrepo/myapp`
```

## 使用 Sops 管理 Helm Chart 中的秘密

我们在“Kubernetes Secrets”中看到了如何在 Kubernetes 中存储秘密数据，以及如何通过环境变量或挂载文件将其传递给应用程序。如果需要管理超过一两个秘密，可能会更容易创建一个包含所有秘密的单个文件，而不是每个文件都包含一个秘密。如果你正在使用 Helm 部署你的应用程序，你可以将该文件作为值文件，并使用 Sops 进行加密（参见“使用 Sops 加密秘密”）。

我们在演示库的*hello-sops*目录中为你构建了一个示例：

```
`cd hello-sops`
`tree`
. ├── k8s │   └── demo │       ├── Chart.yaml │       ├── production-secrets.yaml │       ├── production-values.yaml │       ├── staging-secrets.yaml │       ├── staging-values.yaml │       ├── templates │       │   ├── deployment.yaml │       │   └── secrets.yaml │       └── values.yaml └── temp.yaml 
3 directories, 9 files
```

这是与我们早期示例类似的 Helm Chart 布局（参见“Helm Chart 的内部是什么？”）。在这里，我们定义了一个`Deployment`和一个`Secret`。但在这个例子中，我们添加了一个变化，使得管理不同环境中的多个秘密变得更加容易。

让我们来看看我们的应用程序将需要的秘密：

```
`cat k8s/demo/production-secrets.yaml`
secret_one: ENC[AES256_GCM,data:ekH3xIdCFiS4j1I2ja8=,iv:C95KilXL...1g==,type:str] secret_two: ENC[AES256_GCM,data:0Xcmm1cdv3TbfM3mIkA=,iv:PQOcI9vX...XQ==,type:str] ...
```

在这里，我们使用 Sops 加密了多个应用程序使用的秘密的值。

现在看一下 Kubernetes 的*secrets.yaml*文件：

```
`cat k8s/demo/templates/secrets.yaml`
apiVersion: v1 kind: Secret metadata:
 name: {{ .Values.container.name }}-secrets type: Opaque data:
 {{ $environment := .Values.environment }} app_secrets.yaml: {{ .Files.Get (nospace (cat $environment "-secrets.yaml")) | b64enc }}
```

在最后两行中，我们在 Helm 图表中添加了一些 Go 模板，以便根据 *values.yaml* 文件中设置的 `environment` 从 *production-secrets.yaml* 或 *staging-secrets.yaml* 文件中读取秘密。

最终结果将是一个名为 *demo-secrets* 的单个 Kubernetes `Secret`，其中包含在任一秘密文件中定义的所有键值对。此 `Secret` 将作为一个名为 *secrets.yaml* 的单个文件挂载到 Deployment 中供应用程序使用。

我们还在最后一行末尾添加了 `...| b64enc`。这是使用 Helm 的 Go 模板的另一个方便的快捷方式，自动将秘密数据从明文转换为 `base64`，这是 Kubernetes 默认期望秘密的格式（参见 `base64`）。

我们需要首先使用 Sops 临时解密文件，然后将更改应用到 Kubernetes 集群。这是一个命令管道，用于部署演示应用程序的暂存版本及其暂存秘密：

```
sops -d k8s/demo/staging-secrets.yaml > temp-staging-secrets.yaml && \
helm upgrade --install staging-demo --values staging-values.yaml \
--values temp-staging-secrets.yaml ./k8s/demo && rm temp-staging-secrets.yaml
```

这是它的工作原理：

1.  Sops 解密 *staging-secrets* 文件，并将解密后的输出写入 *temp-staging-secrets*。

1.  Helm 使用来自 *staging-values* 和 *temp-staging-secrets* 的数值来安装 `demo` 图表。

1.  *temp-staging-secrets* 文件被删除。

因为所有这些操作都在一步中完成，所以我们不会留下包含明文秘密的文件，供错误的人发现。这与 [`helm-secrets`](https://oreil.ly/BX6uf) 插件的工作方式非常相似，因此如果您喜欢这种管理秘密的工作流程，那么值得查看这个项目。

# 使用 Helmfile 管理多个图表

当我们在 “Helm: Kubernetes 包管理器” 中介绍 Helm 时，我们向您展示了如何将演示应用程序 Helm 图表部署到 Kubernetes 集群。尽管 Helm 非常有用，但它一次只能操作一个图表。您如何知道应该在集群中运行哪些应用程序，以及您在安装它们时应用的自定义设置？

有一个很棒的工具叫做 [Helmfile](https://oreil.ly/pptjH)，它可以帮助你完成这些操作。就像 Helm 通过模板和变量使您能够部署单个应用程序一样，Helmfile 使您能够使用单个命令部署应该安装在您的集群上的所有内容。

## Helmfile 的内容是什么？

在 `demo` 仓库中有一个使用 Helmfile 的示例。在 *hello-helmfile* 文件夹中，您会找到 *helmfile.yaml*：

```
repositories:
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts

releases:
  - name: demo
    namespace: demo
    chart: ../hello-helm3/k8s/demo
    values:
      - "../hello-helm3/k8s/demo/production-values.yaml"

  - name: kube-state-metrics
    namespace: kube-state-metrics
    chart: prometheus-community/kube-state-metrics

  - name: prometheus
    namespace: prometheus
    chart: prometheus-community/prometheus
    set:
      - name: rbac.create
        value: true
```

`repositories` 部分定义了我们将要引用的 Helm 图表仓库。在这种情况下，唯一的仓库是 `prometheus-community`，官方的 Prometheus Helm 仓库。如果您正在使用自己的 Helm 图表仓库（参见 “创建 Helm 图表仓库”），请在这里添加。

接下来，我们定义了一组 `releases`：我们希望部署到集群的应用程序。每个发布指定了以下一些元数据：

+   `name` 用于部署的 Helm 图表名称

+   `namespace` 用于部署

+   `chart` 是图表本身的 URL 或路径

+   `values`给出了与部署一起使用的*values.yaml*文件的路径

+   `set`在值文件中设置任何额外的值

我们在这里定义了三个发布版本：演示应用程序，加上 Prometheus（请参阅“Prometheus”）和`kube-state-metrics`（请参阅“Kubernetes Metrics”）。

## 图表元数据

请注意，我们已经指定了相对路径到`demo`图表和值文件：

```
- name: demo
  namespace: demo
  chart: ../hello-helm3/k8s/demo
  values:
    - "../hello-helm3/k8s/demo/production-values.yaml"
```

因此，您的图表不需要在图表存储库中以便 Helmfile 管理它们； 例如，您可以将它们全部保存在同一个源代码存储库中。

对于`prometheus`图表，我们已指定了`prometheus-community/prometheus`。 由于这不是文件系统路径，Helmfile 知道要在仓库的 URL 上查找图表，我们在`repositories`部分中定义了该 URL：

```
- name: prometheus-community
  url: https://prometheus-community.github.io/helm-charts/
```

所有图表都在各自的*values.yaml*文件中设置了各种默认值。 在 Helmfile 的`set:`部分，您可以指定在安装应用程序时想要覆盖的任何值。

在这个例子中，对于`prometheus`发布，我们想确保`rbac.create`的值设置为`true`：

```
- name: prometheus
  namespace: prometheus
  chart: prometheus-community/prometheus
  set:
    - name: rbac.create
      value: true
```

## 应用 Helmfile

*helmfile.yaml*然后以声明方式指定了集群中应该运行的所有内容（或者至少是其中的一个子集），就像 Kubernetes 清单一样。 当您应用这个声明性清单时，Helmfile 将使集群符合您的规范。

要做到这一点，请运行：

```
`helmfile sync`
Adding repo prometheus- community https://prometheus-community.github.io/helm-charts "prometheus-community" has been added to your repositories 
Building dependency release=demo, chart=../hello-helm3/k8s/demo Affected releases are:
 demo (../hello-helm3/k8s/demo) UPDATED kube-state-metrics (prometheus-community/kube-state-metrics) UPDATED prometheus (prometheus-community/prometheus) UPDATED prometheus-node- exporter (prometheus-community/prometheus-node-exporter) UPDATED ...
```

就好像您依次为您定义的每个 Helm 图表运行了`helm install`/`helm upgrade`一样。

例如，您可能希望作为连续部署管道的一部分自动运行`helm sync`（请参阅第十四章）。 而不是手动运行`helm install`以将新应用程序添加到集群中，您可以只需编辑您的 Helmfile，将其检入源代码控制，并等待自动化推出您的更改。 在“GitOps”中，我们还将介绍使用集中工具管理多个 helm 发布的另一种方法。

###### 提示

使用单一的真相源。 不要混合手动使用 Helm 部署单个图表，并使用 Helmfile 或 GitOps 工具在集群中声明性地管理所有图表。 如果您应用了 Helmfile，然后还使用 Helm 在带外部署或修改应用程序，您将不再拥有集群的单一真相源。 这肯定会导致问题，因此选择一种管理部署的流程，并在使用方式上保持一致。

如果 Helmfile 不完全符合您的口味，[Helmsman](https://oreil.ly/DizPj)是一个类似的工具，做的事情或多或少相同。

与任何新工具一样，我们建议仔细阅读文档，比较各种选项，尝试它们，然后决定哪个适合您。

# 高级清单管理工具

虽然 Helm 是一个很棒的工具，并且被广泛使用，但它确实有一些局限性。编写和编辑 Helm 模板并不是一件很有趣的事情。Kubernetes 的 YAML 文件复杂、冗长且重复。因此，Helm 模板也如此。

正在开发几种新工具来解决这些问题，并使得处理 Kubernetes 清单变得更加容易：要么通过比 YAML 更强大的语言描述，例如 [Jsonnet](https://jsonnet.org)，要么将 YAML 文件分组到基本模式并使用覆盖文件进行定制。

## kustomize

[kustomize](https://oreil.ly/nc9B6) 与 Helm 并列，可能是另一个最受欢迎的清单管理工具。实际上，从 Kubernetes 版本 1.14 开始，kustomize 已经包含在 `kubectl` CLI 工具中。您还可以按照他们的 [说明](https://oreil.ly/ePdt1) 单独使用 `kustomize` 通过安装二进制文件。与模板化不同，kustomize 使用普通的 YAML 文件和覆盖来允许替换和重用。您从 *base* YAML 清单开始，并使用 *overlays* 来修补不同环境或配置的清单。kustomize 将从基本文件加上覆盖生成最终的清单。

我们在 [cloudnativedevops GitHub 演示库](https://oreil.ly/LAI8f) 的 *hello-kustomize* 目录中有一个工作示例：

```
`cd hello-kustomize/demo/base`
`kubectl kustomize ./`
apiVersion: v1 kind: Service metadata:
 labels: app: demo org: acmeCorporation name: demo-service ... --- apiVersion: apps/v1 kind: Deployment metadata:
 labels: app: demo org: acmeCorporation name: demo spec:
 replicas: 1 ...
```

在这个例子中，kustomize 读取我们的 *kustomization.yaml* 文件，在部署（Deployment）和服务（Service）的 `app` 和 `org` 上设置了 `commonLabels`，并显示了更新清单的最终渲染输出。与 Helm 的不同之处在于，这里的原始 *deployment.yaml* 和 *service.yaml* 文件是完全有效且可用的 YAML，但我们仍然可以通过 kustomize 添加一些灵活性，例如在我们的清单中使用多个公共标签。

`patchesStrategicMerge` kustomize 设置允许我们覆盖字段，例如按环境更改副本计数：

```
`kubectl kustomize ../overlays/production`
... apiVersion: apps/v1 kind: Deployment metadata:
 labels: app: demo org: acmeCorporation name: prod-demo spec:
 replicas: 3 ...
```

再次强调，您应该查看渲染后的部署（Deployment）和服务（Service）清单的最终结果，但请注意我们如何基于 *overlays/production* 目录中的 *replicas.yaml* 文件，将生产环境的 `replicas` 字段更改为 `3`。

我们在这里应用的另一个有用修改是 `namePrefix` 设置。我们的新部署现在命名为 `name: prod-demo`，而不是在 *base* 目录中定义的默认 `name: demo`，我们的服务被重命名为 `prod-demo-service`。Kustomize 还配有其他类似的辅助工具，用于在不同情况下对清单进行调整，同时保持原始的 YAML 清单处于可用且有效的状态。

要应用您的 kustomize 定制清单，您可以使用 `kubectl apply -k`（而不是 `kubectl apply -f`），`kubectl` 将使用 kustomize 读取 YAML 文件，渲染任何覆盖或修改，然后将最终结果应用到集群中。

如果不喜欢模板化 YAML 文件，那么 kustomize 值得一试。您可能会遇到社区项目，这些项目提供使用它们的 Helm 图表或 kustomize 安装其应用程序的选项，因此熟悉它的工作方式是值得的。

## Tanka

有时候，仅使用声明性 YAML 是不够的，特别是对于需要能够使用计算和逻辑的大型和复杂部署。例如，您可能希望根据集群的大小动态设置副本数。为此，您需要一个真正的编程语言。

[Tanka](https://oreil.ly/MBVrB) 允许您使用名为 Jsonnet 的语言编写 Kubernetes 清单，Jsonnet 是 JSON 的扩展版本（它是等同于 YAML 的声明性数据格式，Kubernetes 也可以理解 JSON 格式的清单）。Jsonnet 为 JSON 添加了重要的功能——变量、循环、算术、条件语句、错误处理等等。

## Kapitan

[Kapitan](https://oreil.ly/ekar1) 是另一个清单管理工具，专注于在多个应用程序甚至集群之间共享配置值。它还可以用于其他类型的工具，如 terraform 代码、Dockerfiles 和 Jinja2 模板文档。Kapitan 具有配置值的分层数据库（称为 *inventory*），允许您通过插入不同的值来重用清单模式，具体取决于环境或应用程序，并且可以使用 Jsonnet 或基于 Python 的 Kadet 后端引擎生成它们：

```
local kube = import "lib/kube.libjsonnet";
local kap = import "lib/kapitan.libjsonnet";
local inventory = kap.inventory();
local p = inventory.parameters;

{
    "00_namespace": kube.Namespace(p.namespace),
    "10_serviceaccount": kube.ServiceAccount("default")
}
```

## kompose

如果您一直在 Docker 容器中运行生产服务，但尚未使用 Kubernetes，则可能熟悉 Docker Compose。

Compose 允许您定义和部署一组共同工作的容器：例如，一个 web 服务器，一个后端应用程序以及像 Redis 这样的数据库。一个单独的 *docker-compose.yml* 文件可以用来定义这些容器如何相互通信。

[kompose](https://oreil.ly/vh2AU) 是一个将 *docker-compose.yml* 文件转换为 Kubernetes 清单的工具，帮助您从 Docker Compose 迁移到 Kubernetes，而无需从头编写自己的 Kubernetes 清单或 Helm 图表。

## Ansible

您可能已经熟悉 Ansible，这是一款流行的基础设施自动化工具。它不是专门针对 Kubernetes 的，但它可以使用扩展模块管理许多不同类型的资源，类似于 Puppet（参见“Puppet Kubernetes Module”）。

除了安装和配置 Kubernetes 集群外，Ansible 还可以直接管理 Kubernetes 资源，如部署和服务，使用 [`k8s` 模块](https://oreil.ly/dJTLA)。

像 Helm 一样，Ansible 可以使用其标准模板语言（Jinja）模板化 Kubernetes 清单，并且它具有更复杂的变量查找概念，使用分层系统。例如，您可以为一组应用程序或部署环境（如 `staging`）设置公共值。

如果你的组织已经在使用 Ansible，那么评估一下是否应该也将其用于管理 Kubernetes 资源是非常值得的。如果你的基础设施仅基于 Kubernetes，那么 Ansible 可能会比你需要的功能更多，但对于混合基础设施来说，使用一个工具来管理所有内容会非常有帮助：

```
kube_resource_configmaps:
  my-resource-env: "{{ lookup('template', template_dir +
'/my-resource-env.j2') }}"
kube_resource_manifest_files: "{{ lookup('fileglob', template_dir +
'/*manifest.yml') }}"
- hosts: "{{ application }}-{{ env }}-runner"
  roles:
    - kube-resource
```

## kubeval

与本节讨论的其他工具不同，[`kubeval`](https://oreil.ly/87Nh5) 不是用于生成或模板化 Kubernetes 配置文件的工具，而是用于验证它们的有效性。

每个 Kubernetes 版本都有一个不同的 YAML 或 JSON 配置文件的模式，能够自动检查你的配置文件是否符合模式是非常重要的。例如，`kubeval` 将检查你是否为特定对象指定了所有必需的字段，以及这些值是否为正确的类型。

当应用配置文件时，`kubectl` 也会验证它们，并在尝试应用无效配置文件时给出错误。但事先验证它们也非常有用。`kubeval` 不需要访问集群，也可以验证针对任何 Kubernetes 版本的配置文件。

将 `kubeval` 添加到你的持续部署流水线中是个好主意，这样可以在对配置文件进行更改时自动验证它们。你也可以使用 `kubeval` 测试，例如在实际升级之前，你的配置文件是否需要任何调整以适用于最新版本的 Kubernetes。

# 总结

尽管你可以仅使用原始的 YAML 配置文件将应用部署到 Kubernetes，但这样做并不方便。Helm 是一个强大的工具，可以帮助你解决这个问题，前提是你要了解如何充分利用它。

未来将有很多不同的工具可以使 Kubernetes 配置文件的管理变得更加简单。了解使用 Helm 的基础知识非常重要，因为许多流行的工具都以 Helm Charts 的形式提供给你，用于在你的集群中安装：

+   Chart 是 Helm 的一个包规范，包括关于包的元数据，一些用于配置它的配置值，以及引用这些值的模板 Kubernetes 对象。

+   安装一个 Chart 会创建一个 Helm 发布（release）。每次你安装一个 Chart 的实例时，都会创建一个新的发布。当你使用不同的配置值更新一个发布时，Helm 会增加发布的修订版本号。

+   要根据自己的需求定制一个 Helm Chart，可以创建一个自定义的 values 文件，仅覆盖你关心的设置，并将其添加到 `helm install` 或 `helm upgrade` 命令行中。

+   你可以使用一个变量（例如 `environment`）来选择不同的值或密钥集，具体取决于部署环境：预备环境、生产环境等。

+   使用 Helmfile，你可以声明性地指定一组 Helm Charts 和要应用到你的集群的值，并通过一个命令安装或更新它们全部。

+   Helm 可与 Sops 一起用于处理图表中的秘密配置。它还可以使用函数自动将您的秘密进行 base64 编码，这是 Kubernetes 所期望的。

+   Helm 不是管理 Kubernetes 清单的唯一工具。Kustomize 是另一个强大的工具，甚至内置在 `kubectl` 中。Kustomize 采用与 Helm 不同的方法，而不是插入变量，而是使用 YAML 覆盖来调整清单。

+   Tanka 和 Kapitan 是使用 Jsonnet 的替代清单管理工具。

+   使用 `kubeval` 快速测试和验证清单的有效语法和常见错误。
