# 第三章：使用 Helm 进行基础以外的探索

在上一章中，我们看了 Helm 最常用的命令。在本章中，我们将探索 Helm 工具提供的其他功能。我们将深入研究提供有关发布信息的命令，测试安装的命令，并跟踪历史记录的命令。最后，我们将重新审视安装和升级，这次涵盖高级情况。

我们将开始使用一些有助于故障排除和调试的工具。

# 模板化与干运行

当 Helm 安装一个发布时，程序将通过几个阶段。它加载图表，解析传递给程序的值，读取图表元数据等。在过程的中间部分，Helm 将编译图表中的所有模板（一次性全部编译），然后通过传递值（就像我们在前一章中看到的）渲染它们。在这个中间部分期间，它执行所有模板指令。一旦模板被渲染成 YAML，Helm 通过将其解析为 Kubernetes 对象来验证 YAML 的结构。最后，Helm 序列化这些对象并将它们发送到 Kubernetes API 服务器。

大致而言，过程如下：

1.  加载整个图表，包括其依赖项。

1.  解析值。

1.  执行模板，生成 YAML。

1.  将 YAML 解析为 Kubernetes 对象以验证数据。

1.  将其发送到 Kubernetes。

举例来说，让我们来看看我们在前一章中发出的一个命令：

```
$ helm install mysite bitnami/drupal --set drupalUsername=admin
```

在第一阶段，Helm 将定位名为`bitnami/drupal`的图表并加载该图表。如果图表是本地的，则将从磁盘读取。如果给定 URL，则将从远程位置获取（可能使用插件来帮助获取图表）。

然后它将`--set drupalUsername=admin`转换为可以注入模板的值。此值将与图表的*values.yaml*文件中的默认值组合。Helm 对数据进行了一些基本检查。如果有问题解析用户输入，或者默认值损坏，它将以错误退出。否则，它将构建一个单一的大值对象，模板引擎可以用来进行替换。

生成的值对象是通过加载图表文件的所有值，叠加从文件加载的任何值（即使用`-f`标志），然后叠加使用`--set`标志设置的任何值来创建的。换句话说，`--set`值覆盖传递值文件中的设置，这反过来又覆盖图表默认的*values.yaml*文件中的任何设置。

在这一点上，Helm 将读取 Drupal 图表中的所有模板，然后执行这些模板，将合并的值传递给模板引擎。格式错误的模板将导致错误。但还有许多其他情况可能导致失败。例如，如果缺少必需的值，则在此阶段将返回错误。

需要注意的是，当执行时，某些 Helm 模板需要有关 Kubernetes 的信息。因此，在模板渲染期间，Helm *可能* 会联系 Kubernetes API 服务器。这是一个我们将在稍后讨论的重要话题。

然后，将前一步的输出从 YAML 解析为 Kubernetes 对象。此时，Helm 将执行一些模式级验证，确保对象格式良好。然后，它们将被序列化为 Kubernetes 的最终 YAML 格式。

在最后阶段，Helm 将 YAML 数据发送到 Kubernetes API 服务器。这是`kubectl`和其他 Kubernetes 工具交互的服务器。

API 服务器将对提交的 YAML 运行一系列检查。如果 Kubernetes 接受 YAML 数据，Helm 将认为部署成功。但如果 Kubernetes 拒绝 YAML，则 Helm 将退出并显示错误。

后面，我们将详细讨论一旦对象发送到 Kubernetes 后会发生什么。特别是，我们将涵盖 Helm 如何将前面描述的过程与安装和修订相关联。但现在，我们已经有足够的工作流信息来理解两个相关的 Helm 特性：`--dry-run`标志和`helm template`命令。

## `--dry-run`标志

像`helm install`和`helm upgrade`这样的命令提供了一个名为`--dry-run`的标志。当您提供此标志时，它将导致 Helm 依次执行前四个阶段（加载图表，确定值，渲染模板，格式化为 YAML）。但是当第四阶段完成时，Helm 将在标准输出中转储大量信息，包括所有渲染的模板。然后它将退出，而不会将对象发送到 Kubernetes，也不会创建任何发布记录。

例如，这是我们之前安装 Drupal 的版本，附加了`--dry-run`标志：

```
$ helm install mysite bitnami/drupal --values values.yaml --set \
drupalEmail=foo@example.com --dry-run
```

在输出的顶部，它会打印一些关于发布的信息：

```
NAME: mysite
LAST DEPLOYED: Tue Aug 11 11:42:05 2020
NAMESPACE: default
STATUS: pending-install
REVISION: 1
HOOKS:
```

前面告诉我们安装的名称是什么，上次部署时间是什么（在本例中是当前日期和时间），它将被部署到哪个命名空间，发布的阶段是什么（`pending-install`），以及修订号。由于这是一个安装过程，修订号是`1`。在升级时，它将是`2`或更高。

最后，如果此图表声明了任何挂钩，它们将在此处枚举。有关挂钩的更多信息，请参阅第六章和第七章。

乍一看，这个元数据条目似乎包含了许多不必要的数据。毕竟，如果我们实际上没有进行安装，`LAST DEPLOYED`有什么作用呢？事实上，这一信息块是 Helm 中使用的标准设置之一。它是*发布记录*的一部分：关于发布的一组信息。像`helm get`这样的命令使用这些相同的字段。

紧接着信息块之后，所有渲染的模板都会被转储到标准输出：

```
# Source: drupal/charts/mariadb/templates/test-runner.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "mysite-mariadb-test-afv3u"
  annotations:
    "helm.sh/hook": test-success
spec:
  initContainers:
    - name: "test-framework"
      image: docker.io/dduportal/bats:0.4.0
...
```

渲染后的 Drupal 图表有数千行，因此前面只显示了输出的前几行。

最后，在干运行输出的底部，Helm 打印了面向用户的释放说明：

```
NOTES:
*******************************************************************
*** PLEASE BE PATIENT: Drupal may take a few minutes to install ***
*******************************************************************

1\. Get the Drupal URL:

  You should be able to access your new Drupal installation through

  http://drupal.local/
...
```

示例为了简洁起见被截断。

这个干运行特性为 Helm 用户提供了一种在发送到 Kubernetes 之前调试图表输出的方式。通过渲染所有模板，您可以检查到底会提交到集群的内容。并且通过发布数据，您可以验证释放是否按您期望的方式创建。

`--dry-run` 标志的主要目的是让人们有机会在发送到 Kubernetes 之前检查和调试输出。但不久之后，Helm 的维护者注意到用户中存在一种趋势。人们想要使用 `--dry-run` 将 Helm 作为模板引擎使用，然后使用其他工具（如 `kubectl`）将渲染后的输出发送到 Kubernetes。

但 `--dry-run` 并非为此用例而编写，这导致了一些问题：

1.  `--dry-run` 将非 YAML 信息与渲染模板混合在一起。这意味着在发送到 `kubectl` 等工具之前，数据必须被清理。

1.  升级时的 `--dry-run` 可能会生成与安装时不同的 YAML 输出，这可能会令人困惑。

1.  它联系 Kubernetes API 服务器进行验证，这意味着即使只用于 `--dry-run` 释放，Helm 也必须具有 Kubernetes 凭据。

1.  它还将信息插入到模板引擎中，这些信息是特定于集群的。因此，某些渲染过程的输出可能是集群特定的。

为了解决这些问题，Helm 的维护者引入了一个完全独立的命令：`helm template`。

## `helm template` 命令

虽然 `--dry-run` 设计用于调试，`helm template` 则旨在将 Helm 的模板渲染过程与安装或升级逻辑隔离开来。

早些时候，我们看过 Helm 安装或升级的五个阶段。`template` 命令执行前四个阶段（加载图表、确定值、渲染模板、格式化为 YAML）。但这需要考虑一些额外的注意事项：

+   在 `helm template` 过程中，Helm *从不* 联系远程 Kubernetes 服务器。

+   `template` 命令始终像一个安装操作一样。

+   通常需要与 Kubernetes 服务器联系的模板函数和指令现在将仅返回默认数据。

+   图表仅有默认的 Kubernetes 种类访问权限。

关于最后一项，`helm template` 做了一个显著的简化假设。Kubernetes 服务器支持内置种类（`Pod`、`Service`、`ConfigMap` 等）以及由自定义资源定义（CRD）生成的自定义种类。在运行安装或升级时，Helm 在处理图表之前从 Kubernetes 服务器获取这些种类。

然而，`helm template` 在此步骤上的操作方式有所不同。当 Helm 编译时，它是针对特定版本的 Kubernetes 进行编译的。Kubernetes 库包含了该版本发布的内置种类列表。Helm 使用该内置列表，而不是从 API 服务器获取列表。因此，在 `helm template` 运行期间，Helm 没有访问任何 CRD，因为 CRD 安装在集群上，而不包括在 Kubernetes 库中。

###### 注意

在使用旧版本的 Helm 运行针对使用新种类或版本的图表时，可能会在 `helm template` 过程中出现错误，因为 Helm 没有将最新的种类或版本编译进去。

由于这些决策，`helm template` 在每次运行后产生一致的输出。更重要的是，它可以在没有访问 Kubernetes 集群的环境中运行，比如持续集成（CI）流水线。

输出也不同于 `--dry-run`。以下是一个示例命令：

```
$ helm template mysite bitnami/drupal --values values.yaml --set \
drupalEmail=foo@example.com
 ---
# Source: drupal/charts/mariadb/templates/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysite-mariadb
  labels:
    app: "mariadb"
    chart: "mariadb-7.5.1"
    release: "mysite"
    heritage: "Helm"
type: Opaque
# ... LOTS removed from here
  volumes:
    - name: tests
      configMap:
        name: mysite-mariadb-tests
    - name: tools
      emptyDir: {}
  restartPolicy: Never
```

上述内容是输出的大大简化版本，仅显示了命令以及数据的开头和结尾的示例。需要注意的是，默认情况下仅打印 YAML 格式的 Kubernetes 清单。

由于 Helm 在 `helm template` 运行期间不联系 Kubernetes 集群，它不会对输出进行完整验证。在这种情况下，如果您希望有这种行为，可以选择使用 `--validate` 标志，但在这种情况下，Helm 需要一个带有集群凭据的有效 *kubeconfig* 文件。

`helm template` 命令有许多标志，与 `helm install` 中的标志相对应。因此，在许多情况下，您可以像执行 `helm install` 命令一样执行 `helm template` 命令，然后捕获 YAML 并将其与其他工具一起使用。

# 使用后渲染而不是 Helm Template

有时您希望拦截 YAML，使用自己的工具修改它，然后将其加载到 Kubernetes 中。Helm 提供了一种执行此外部工具的方式，而无需使用 `helm template`。在 `install`、`upgrade`、`rollback` 和 `template` 上使用 `--post-renderer` 标志会导致 Helm 将 YAML 数据发送到命令，然后再将结果读回到 Helm。这是与 Kustomize 等工具配合使用的一个很好的方式。

总结一下，`helm template` 是将 Helm charts 渲染为 YAML 的工具，而 `--dry-run` 标志是在不将数据加载到 Kubernetes 的情况下调试安装和升级命令的工具。

# 了解发布情况

在前一章中，我们简要介绍了 `helm get` 命令。现在，我们将深入了解该命令以及其他提供有关 Helm 发布信息的命令。

首先，让我们回顾一下前一节中 Helm 安装的五个阶段。它们是：

1.  加载图表。

1.  解析值。

1.  执行模板。

1.  渲染 YAML。

1.  发送到 Kubernetes。

前四个阶段主要涉及数据的本地表示。也就是说，Helm 在运行 `helm` 命令的同一台计算机上执行所有处理。

然而，在最后一个阶段，Helm 将该数据发送到 Kubernetes。然后两者之间来回通信，直到发布被接受或拒绝。

在第五阶段期间，Helm 必须监控发布的状态。此外，由于许多人可能在同一个应用程序安装的副本上工作，因此 Helm 需要以多用户可以看到信息的方式监控状态。

Helm 提供了 *发布记录* 功能。

## 发布记录

当我们安装 Helm 图表（使用 `helm install`）时，新的安装会创建在您指定的命名空间中，或者默认命名空间中。我们在前一章已经看过了这个过程。

在那一章的结尾，我们还看到了 `helm install` 如何创建一种特殊类型的 Kubernetes `Secret` 来保存发布信息。我们还看到了如何使用 `kubectl` 检查这些 `Secret`：

```
$ kubectl get secret
NAME                           TYPE                                  DATA   AGE
default-token-g777k            kubernetes.io/service-account-token   3      6m
mysite-drupal                  Opaque                                1      2m20s
mysite-mariadb                 Opaque                                2      2m20s
sh.helm.release.v1.mysite.v1   helm.sh/release.v1                    1      2m20s
```

特别要注意的是最后一个 `Secret`，`sh.helm.release.v1.mysite.v1`。请注意，它使用了特殊类型（`helm.sh/release.v1`）来指示这是一个 Helm 密钥。Helm 自动生成此密钥来跟踪我们的 `mysite` 安装的版本 1（这是一个 Drupal 站点）。

每次我们升级 `mysite` 安装时，都会创建一个新的 `Secret` 来跟踪每个发布。换句话说，发布记录跟踪每个安装的 *修订版本*：

```
$ helm upgrade mysite bitnami/drupal
# Output omitted
$ helm upgrade mysite bitnami/drupal
# Output omitted
$ kubectl get secrets
NAME                           TYPE                                  DATA   AGE
default-token-g777k            kubernetes.io/service-account-token   3      8m43s
mysite-drupal                  Opaque                                1      5m3s
mysite-mariadb                 Opaque                                2      5m3s
sh.helm.release.v1.mysite.v1   helm.sh/release.v1                    1      5m3s
sh.helm.release.v1.mysite.v2   helm.sh/release.v1                    1      20s
sh.helm.release.v1.mysite.v3   helm.sh/release.v1                    1      8s
```

在上述例子中，我们已经升级了几次，现在我们在 `mysite` 的 `v3` 版本。默认情况下，Helm 跟踪每个安装的十个修订版本。一旦一个安装超过十个发布，Helm 将删除最旧的发布记录，直到最多保留指定数量。

每个发布记录包含足够的信息来重新创建该修订版本的 Kubernetes 对象（这对于 `helm rollback` 非常重要）。它还包含关于发布的元数据。

例如，如果我们使用 `kubectl` 查看发布，我们会看到类似于这样的内容：

```
apiVersion: v1
data:
  release: SDRzSUFBQU... # Lots of Base64-encoded data removed
kind: Secret
metadata:
  creationTimestamp: "2020-08-11T18:37:26Z"
  labels: ![1](img/1.png)
    modifiedAt: "1597171046"
    name: mysite
    owner: helm
    status: deployed
    version: "3"
  name: sh.helm.release.v1.mysite.v3
  namespace: default
  resourceVersion: "1991"
  selfLink: /api/v1/namespaces/default/secrets/sh.helm.release.v1.mysite.v3
  uid: cbb8b457-e331-467b-aa78-1e20360b5be6
type: helm.sh/release.v1
```

![1](img/#co_beyond_the_basics_with_helm_CO1-1)

标签包含 Helm 元数据。

在这个例子中，已经删除了巨大的 Base64 编码数据以及一些其他不必要的字段。该数据块包含了图表和发布的 gzip 压缩表示。但是重要的是，Kubernetes 元数据中的 `labels` 部分包含了关于此发布的信息。

例如，我们可以看到，这些数据描述了名为 `mysite` 的发布，其当前修订版本号为 `3`，并且发布被标记为 `deployed`。如果我们查看版本 `2`，我们会看到发布的 `status` 是 `superseded`，这意味着它已被后续版本替换。

简而言之，这个密钥是存储在 Kubernetes 内部的，以便同一集群的不同用户可以访问相同的发布信息。

在发布的生命周期中，它可以通过几种不同的状态。以下是您可能会看到的顺序大致排列如下：

`pending-install`

在将清单发送到 Kubernetes 之前，Helm 通过创建一个状态设置为`pending-install`的发布（标记为版本 1）来声明安装。

`deployed`

一旦 Kubernetes 接受来自 Helm 的清单，Helm 会更新发布记录，将其标记为已部署。

`pending-upgrade`

当开始 Helm 升级时，将为安装创建一个新的发布（例如，`v2`），并且其状态设置为`pending-upgrade`。

`superseded`

当运行升级时，将更新最后部署的发布，标记为`superseded`，并且新升级的发布从`pending-upgrade`更改为`deployed`。

`pending-rollback`

如果创建了回滚操作，则会创建一个新的发布（例如，`v3`），并且其状态设置为`pending-rollback`，直到 Kubernetes 接受发布清单。然后它被标记为`deployed`，而上一个发布被标记为`superseded`。

`uninstalling`

当执行`helm uninstall`时，会读取最近的发布，然后其状态更改为`uninstalling`。

`uninstalled`

如果在删除过程中保留历史记录，则在`helm uninstall`完成后，上一个发布的状态将更改为`uninstalled`。

`failed`

最后，如果在*任何*操作期间，Kubernetes 拒绝了 Helm 提交的清单，Helm 将标记该发布为`failed`。

## 列出发布版本

状态消息出现在多个 Helm 命令中。我们已经看到`pending-install`如何在`--dry-run`中出现。在本节和下一节中，我们将看到这些消息出现的几个更多地方。

在前一章中，我们使用`helm list`查看我们已安装的图表。考虑到我们的状态覆盖，值得重新访问`helm list`。`list`命令是快速检查您发布状态的最佳工具。

例如，假设我们有一个同时安装`drupal`和`wordpress`图表的集群。这是`helm list`的输出：

```
NAME     	NAMESPACE REVISION  UPDATED       STATUS  	CHART           	APP V...
mysite   	default  	3         2020-08-11... deployed  drupal-7.0.0      9.0.0
wordpress default  	2         2020-08-12... deployed  wordpress-9.3.11  5.4.2
```

要显示失败的结果，我们可以运行一个我们知道会失败的升级命令：

```
$ helm upgrade wordpress bitnami/wordpress --set image.pullPolicy=NoSuchPolicy
Error: UPGRADE FAILED: cannot patch "wordpress" with kind Deployment:
Deployment.apps "wordpress" is invalid:
spec.template.spec.containers[0].imagePullPolicy: Unsupported value:
"NoSuchPolicy": supported values: "Always", "IfNotPresent", "Never"
```

正如错误消息所示，无法将拉取策略设置为`NoSuchPolicy`。此错误来自 Kubernetes API 服务器，这意味着 Helm 提交了清单，而 Kubernetes 拒绝了它。因此，我们的发布应该处于失败状态。

我们可以通过再次运行`helm ls`来验证这一点：

```
$ helm ls
NAME      NAMESPACE REVISION  UPDATED     STATUS    CHART             APP VER...
mysite   	default  	3         2020-08-11  deployed  drupal-7.0.0      9.0.0
wordpress default  	3         2020-08-12  failed    wordpress-9.3.11  5.4.2
```

值得再次注意的是，我们新失败的`wordpress`安装的`REVISION`字段已从`2`增加到`3`。即使失败的发布也附有修订版本。我们将看到为什么这一点在“历史和回滚”中很重要。

## 使用`helm get`查找发布的详细信息

虽然`helm list`提供了安装的摘要视图，但`helm get`命令集提供了有关特定发布的更详细信息。

有五个`helm get`子命令（`hooks`、`manifests`、`notes`、`values`和`all`）。每个子命令检索 Helm 为发布跟踪的一部分信息。

### 使用 helm get notes

`helm get notes`子命令打印发布说明：

```
$ helm get notes mysite
NOTES:

*******************************************************************
*** PLEASE BE PATIENT: Drupal may take a few minutes to install ***
*******************************************************************

1\. Get the Drupal URL:

  You should be able to access your new Drupal installation through

  http://drupal.local/
...
```

这个输出应该看起来很熟悉，因为`helm install`和`helm upgrade`都会在成功操作结束时打印发布说明。但是`helm get notes`提供了一个方便的方式来随需获取这些说明。在您忘记 Drupal 站点的 URL 是什么的情况下，这是很有用的。

### 使用 helm get values

一个有用的子命令是`values`。您可以使用它来查看上次发布时提供的值。在前一节中，我们升级了一个 WordPress 安装并导致其失败。我们可以使用`helm get values`查看导致其失败的值：

```
$ helm get values wordpress
USER-SUPPLIED VALUES:
image:
  pullPolicy: NoSuchPolicy
```

我们知道修订版本 2 成功了，但修订版本 3 失败了。因此，我们可以查看早期的值以查看发生了什么变化：

```
$ helm get values wordpress --revision 2
USER-SUPPLIED VALUES:
image:
  tag: latest
```

通过这个，我们可以看到一个值被移除，一个值被添加。这样的功能旨在帮助 Helm 用户更容易地识别错误的来源。

这个命令也对了解发布配置的总体状态很有用。我们可以使用`helm get values`查看该发布当前设置的*所有*值。为此，我们使用`--all`标志：

```
$ helm get values wordpress --all
COMPUTED VALUES:
affinity: {}
allowEmptyPassword: true
allowOverrideNone: false
customHTAccessCM: null
customLivenessProbe: {}
customReadinessProbe: {}
externalDatabase:
  database: bitnami_wordpress
  host: localhost
  password: ""
  port: 3306
...
```

当指定`--all`标志时，Helm 将获取完整的计算值集，按字母顺序排序。这是一个查看发布配置的确切状态的好工具。

# 查看默认值

尽管`helm get values`没有显示默认值的方法，但您可以使用`helm inspect values CHARTNAME`查看这些值。这会检查图表本身（而不是发布）并打印出文档化的默认*values.yaml*文件。因此，我们可以使用`helm inspect values bitnami/wordpress`来查看 WordPress 图表的默认配置。

### 使用 helm get manifest

我们将要介绍的最后一个`helm get`子命令是`helm get manifest`。这个子命令检索 Helm 使用图表模板生成的确切 YAML 清单：

```
$ helm get manifest wordpress
# Source: wordpress/charts/mariadb/templates/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: wordpress-mariadb
  labels:
    app: "mariadb"
    chart: "mariadb-7.5.1"
    release: "wordpress"
    heritage: "Helm"
type: Opaque
...
```

有关此命令的一个重要细节是，它不会返回所有资源的*当前状态*。它返回*从模板生成的清单*。在前面的示例中，我们看到一个名为`wordpress-mariadb`的`Secret`。如果我们使用`kubectl`查询该`Secret`，`metadata`部分如下所示：

```
$ kubectl get secret wordpress-mariadb -o yaml
apiVersion: v1
kind: Secret
metadata:
  annotations:
    meta.helm.sh/release-name: wordpress
    meta.helm.sh/release-namespace: default
  creationTimestamp: "2020-08-12T16:45:00Z"
  labels:
    app: mariadb
    app.kubernetes.io/managed-by: Helm
    chart: mariadb-7.5.1
    heritage: Helm
    release: wordpress
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
...
```

`kubectl`的输出包含当前在 Kubernetes 中存在的记录。自模板输出以来已添加了几个字段。一些字段（如注释）由 Helm 本身管理，其他字段（如`managedFields`和`creationTimestamp`）由 Kubernetes 管理。

再次强调，Helm 提供了设计用于简化调试的工具。在 `helm get manifest` 和 `kubectl get` 之间，您可以使用工具比较 Kubernetes 认为是当前对象与图表生成的对象之间的差异。当由 Helm 管理的资源在 Helm 外部手动编辑（例如使用 `kubectl edit`）时，这尤其有帮助。

使用 `helm get`，我们可以仔细检查单个发布。但接下来我们将介绍的工具将为我们提供发布版本的进展视图。在接下来的部分中，我们将查看 `helm history` 和 `helm rollback`。

# 历史和回滚

在本书中，我们区分了安装和修订版本。在本章中，我们使用了一个名为 `mysite` 的安装和另一个名为 `wordpress` 的安装。当我们之前运行 `helm list` 时，我们看到每个安装都有三个发布版本。此外，我们看到 `wordpress` 处于失败状态：

```
$ helm list
NAME     	NAMESPACE REVISION  UPDATED     STATUS    CHART             APP VER...
mysite    default  	3         2020-08-11  deployed  drupal-7.0.0      9.0.0
wordpress default  	3         2020-08-12  failed    wordpress-9.3.11  5.4.2
```

我们可以调查 WordPress 的发布历史记录以查看发生了什么。为此，我们将使用 `helm history`：

```
$ helm history wordpress
REVISION UPDATED       STATUS     CHART             APP VER  DESCRIPTION
1        Wed Aug 12... superseded wordpress-9.3.11  5.4.2    Install complete
2        Wed Aug 12... deployed  	wordpress-9.3.11  5.4.2    Upgrade complete
3        Wed Aug 12... failed    	wordpress-9.3.11  5.4.2    Upgrade \
  "wordpress" failed: cannot patch "wordpress" with kind Deployment: \
  Deployment.apps "wordpress" is invalid: \
  spec.template.spec.containers[0].imagePullPolicy: Unsupported value: \
  "NoSuchPolicy": supported values: "Always", "IfNotPresent", "Never"
```

此命令的输出为我们提供了 `wordpress` 发布的良好历史记录。首先安装了它，然后升级并标记为已部署（这意味着升级成功）。但当再次升级时，该升级失败了。`helm history` 命令甚至给出了 Kubernetes 在标记发布为“失败”时返回的错误消息。

根据错误信息，我们知道发布失败是因为我们提供了无效的镜像拉取策略。所以当然，我们可以通过运行另一个 `helm upgrade` 来纠正这个问题。但想象一种情况，即错误的原因不容易获取。在诊断问题时，而不是将应用程序留在失败状态，最好是简单地回退到之前工作正常的发布版本。

`helm rollback` 的作用是这样的：

```
$ helm rollback wordpress 2
Rollback was a success! Happy Helming!
```

此命令告诉 Helm 检索 `wordpress` 版本 `2` 的发布，并将该清单重新提交给 Kubernetes。回滚不会恢复到集群的先前快照。Helm 并未跟踪足够的信息来执行此操作。它所做的是重新提交先前的配置，然后 Kubernetes 尝试重置资源以匹配该配置。

现在我们可以再次使用 `helm history` 来查看发生了什么：

```
REVISION  UPDATED       STATUS      CHART             APP VER  DESCRIPTION
1         Wed Aug 12... superseded  wordpress-9.3.11  5.4.2    Install complete
2         Wed Aug 12... superseded  wordpress-9.3.11  5.4.2    Upgrade complete
3         Wed Aug 12... failed      wordpress-9.3.11  5.4.2    Upgrade \
  "wordpress" failed: cannot patch "wordpress" with kind Deployment: \
  Deployment.apps "wordpress" is invalid: \
  spec.template.spec.containers[0].imagePullPolicy: Unsupported value: \
  "NoSuchPolicy": supported values: "Always", "IfNotPresent", "Never"
4         Wed Aug 12... deployed    wordpress-9.3.11  5.4.2    Rollback to 2
```

回滚操作创建了一个新的修订版本（`4`）。由于回滚成功（并且 Kubernetes 接受了更改），该发布被标记为“已部署”。请注意，修订版本 `2` 现在被标记为“已取代”，而失败的发布 `3` 仍被标记为“失败”。

因为 Helm 保留了历史记录，所以即使回滚到已知良好的配置后，您仍然可以检查失败的发布。

在大多数情况下，`helm rollback` 是从灾难中恢复的好方法。但是，如果手动编辑了由 Helm 管理的资源，则可能会出现有趣的问题。回滚有时可能会导致一些意外行为，特别是如果 Kubernetes 资源已被用户手动编辑。如果手动编辑不与回滚冲突，Helm 和 Kubernetes 将尝试保留这些手动编辑。本质上，回滚将生成当前资源状态、失败的 Helm 发布和回滚到的 Helm 发布之间的三向差异。在某些情况下，生成的差异可能会导致回滚手工编辑的内容，而在其他情况下，这些差异将被合并。在最坏的情况下，一些手工编辑可能会被覆盖，而其他相关编辑会被合并，导致配置的不一致。这是 Helm 核心维护者建议不要手动编辑资源的许多原因之一。如果所有编辑都通过 Helm 进行，那么您可以有效地使用 Helm 工具，而无需猜测。

## 保留历史记录和回滚

在前一章中，我们看到`helm uninstall`命令有一个名为`--keep-history`的标志。通常，删除事件将销毁与该安装相关联的所有发布记录。但是，如果指定了`--keep-history`，即使已删除，您也可以查看安装的历史记录：

```
$ helm uninstall wordpress --keep-history
release "wordpress" uninstalled
```

```
$ helm history wordpress
REVISION UPDATED       STATUS     	CHART             APP V  DESCRIPTION
1        Wed Aug 12... superseded 	wordpress-9.3.11  5.4.2  Install complete
2        Wed Aug 12... superseded 	wordpress-9.3.11  5.4.2  Upgrade complete
3        Wed Aug 12... failed     	wordpress-9.3.11  5.4.2  Upgrade \
  "wordpress" failed: cannot patch "wordpress" with kind Deployment: \
  Deployment.apps "wordpress" is invalid: \
  spec.template.spec.containers[0].imagePullPolicy: Unsupported value: \
  "NoSuchPolicy": supported values: "Always", "IfNotPresent", "Never"
4        Wed Aug 12... uninstalled  wordpress-9.3.11  5.4.2  Uninstall complete
```

注意，最后一个发布现在标记为`uninstalled`。当保留历史记录时，您可以回滚已删除的安装：

```
$ helm rollback wordpress 4
Rollback was a success! Happy Helming!
```

现在我们可以看到一个新部署的发布`5`：

```
$ helm history wordpress
REVISION UPDATED    STATUS      CHART             APP VER  DESCRIPTION
1        Wed Aug... superseded 	wordpress-9.3.11	5.4.2    Install complete
2        Wed Aug... superseded 	wordpress-9.3.11	5.4.2    Upgrade complete
3        Wed Aug... failed     	wordpress-9.3.11	5.4.2    Upgrade \
  "wordpress" failed: cannot patch "wordpress" with kind Deployment: \
  Deployment.apps "wordpress" is invalid: \
  spec.template.spec.containers[0].imagePullPolicy: Unsupported value: \
  "NoSuchPolicy": supported values: "Always", "IfNotPresent", "Never"
4        Wed Aug... uninstalled wordpress-9.3.11  5.4.2    Uninstall complete
5        Wed Aug... deployed   	wordpress-9.3.11  5.4.2    Rollback to 4
```

但是如果没有`--keep-history`标志，这将无法工作：

```
$ helm uninstall wordpress
release "wordpress" uninstalled
$ helm history wordpress
Error: release: not found
$ helm rollback wordpress 4
Error: release: not found
```

# 深入探讨安装和升级

在第二章中，我们首次介绍了安装和升级 Helm 包，并在本章中探讨了帮助我们处理 Helm 安装的工具。为了结束这一章，我们将回到安装和升级，并看看一些高级特性。

## `--generate-name`和`--name-template`标志

Kubernetes 工作方式的一个微妙危险与命名有关。Kubernetes 假设名称具有某些唯一性属性。例如，一个`Deployment`对象必须在其命名空间内具有唯一名称。也就是说，在命名空间`mynamespace`中，我不能有两个名为`myapp`的`Deployment`。但是我可以有一个名为`myapp`的`Deployment`和一个名为`myapp`的`Pod`。

这使得某些任务变得更加复杂。例如，自动部署事物的 CI 系统必须能够确保它给这些事物的名称在命名空间内是唯一的。解决此问题的一种方法是让 Helm 提供一个生成唯一名称的工具。（另一种方法是如果名称已经存在则始终覆盖。请参阅下一节了解该方法。）

Helm 为`helm install`提供了`--generate-name`标志：

```
$ helm install bitnami/wordpress --generate-name
NAME: wordpress-1597689085
LAST DEPLOYED: Mon Aug 17 11:31:27 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
```

使用 `--generate-name` 标志，我们不再需要在 `helm install` 的第一个参数中提供名称。Helm 根据图表名称和时间戳的组合生成一个名称。在前面的输出中，我们可以看到为我们生成的名称：`wordpress-1597689085`。

在 Helm 2 中，“友好名称”是使用形容词和动物名称生成的。由于有人抱怨发布名称不专业，因此在 Helm 3 中删除了该功能。目前没有重新启用此功能的方法。

然而，还有一个额外的标志允许您指定一个命名模板。`--name-template` 标志允许您像这样做：

```
$ helm install bitnami/wordpress --generate-name \
  --name-template "foo-{{ randAlpha 9 | lower }}"
NAME: foo-yejpiyjmp
LAST DEPLOYED: Mon Aug 17 11:46:04 2020
NAMESPACE: default
```

在本例中，我们使用了名称模板 `foo-{{ randAlpha 9 | lower }}`。这使用 Helm 模板引擎为您生成一个名称。我们将在接下来的几章中介绍 Helm 模板引擎。但这里的名称模板的作用是：`{{` 和 `}}` 标志着模板的开始和结束。在模板内部，我们调用 `randAlpha` 函数，请求从 `a-z, A-Z` 字符范围中获取一个 `9` 字符的随机字符串。然后，我们通过第二个函数 (`lower`) 将结果转换为小写。

查看前面示例的输出，`{{ randAlpha 9 | lower }}` 的结果是 `yejpiyjmp`。因此整个名称模板的结果是 `foo-yejpiyjmp`。

## `--create-namespace` 标志

Kubernetes 中关于命名的另一个考虑与命名空间有关。前面，我们看到在*同一命名空间内*，同一种类的两个对象不能具有相同的名称。但 Kubernetes 也有全局名称的概念。CRD 和命名空间各自都有全局名称。

因此，命名空间在整个集群中必须是唯一的。

每当 Helm 遇到全局唯一名称时，它会采取一种防御性姿态。在后面的章节中，我们将看到图表如何处理全局唯一名称。但在这里，值得指出的是，Helm 3 默认假定如果您尝试将图表部署到一个命名空间中，那么该命名空间应已经存在。

例如，在一个新的集群上，这将失败：

```
$ helm install drupal bitnami/drupal --namespace mynamespace
Error: create: failed to create: namespaces "mynamespace" not found
```

它失败的原因是 `mynamespace` 尚未创建，并且*Helm 不会自动创建命名空间*。它不会创建命名空间，因为命名空间是全局的，安全的假设是，在创建命名空间时，可能需要访问控制（如 RBAC）和其他分配给它的事物，然后才能安全地在生产中使用。简而言之，它认为静默地创建命名空间是意外创建安全漏洞的机会。

然而，Helm 允许您通过明确声明要创建一个命名空间来覆盖此考虑：

```
$ helm install drupal bitnami/drupal --namespace mynamespace --create-namespace
NAME: drupal
LAST DEPLOYED: Mon Aug 17 11:59:29 2020
NAMESPACE: mynamespace
STATUS: deployed
```

通过添加 `--create-namespace`，我们已告知 Helm 我们*知晓*可能没有具有该名称的命名空间，并且我们只想要创建一个。当然，如果您在生产实例上使用此标志，请确保您有其他机制来确保对该新命名空间的安全性。

在 `helm uninstall` 上并没有类似的 `--delete-namespace`。这背后的原因在于 Helm 对全局对象的保护性措施。一旦创建了命名空间，可以向其中放入任意数量的对象，其中并非全部由 Helm 管理。当删除命名空间时，该命名空间中的所有对象也会被删除。因此，Helm 不会自动删除通过 `--create-namespace` 创建的命名空间。要删除命名空间，请使用 `kubectl delete namespace`（当然，在确保该命名空间中不存在重要对象之后）。

## 使用 `helm upgrade --install`

一些系统，如 CI 流水线，被用来在每次发生重要事件时自动安装或升级图表。例如，许多组织都有流水线，每当新代码上传到版本控制系统（如 Git）时触发。GitHub，一个流行的 Git 托管服务，甚至提供了工具，可以在代码更改合并时自动部署。

此类系统通常在无法查询 Kubernetes 的无状态平台上运行基本脚本。这类系统的用户请求 Helm 功能，允许在单个命令中支持 "安装或升级"。

为了促进这种行为，Helm 的维护者们在 `helm upgrade` 命令中添加了 `--install` 标志。`helm upgrade --install` 命令将在不存在该名称的发布时安装一个发布，或者在找到该名称的发布时升级一个发布。在幕后，它通过查询 Kubernetes 是否存在具有给定名称的发布来工作。如果该发布不存在，则切换出升级逻辑并进入安装逻辑。

例如，我们可以使用完全相同的命令顺序运行安装和升级：

```
$ helm upgrade --install wordpress bitnami/wordpress
Release "wordpress" does not exist. Installing it now.
NAME: wordpress
LAST DEPLOYED: Mon Aug 17 13:18:14 2020
NAMESPACE: default
STATUS: deployed
...
$ helm upgrade --install wordpress bitnami/wordpress
Release "wordpress" has been upgraded. Happy Helming!
NAME: wordpress
LAST DEPLOYED: Mon Aug 17 13:18:43 2020
NAMESPACE: default
STATUS: deployed
```

正如我们在输出的第一行中所见，第一次运行命令时进行了安装，而第二次运行命令时进行了升级。

然而，这个命令确实存在一些危险。Helm 无法确定您在 `helm upgrade --install` 中提供的安装名称是否属于您打算升级的发布版本，或者只是碰巧与您想要安装的内容同名。对这个命令的粗心使用可能导致用一个安装覆盖另一个安装。这就是为什么这不是 Helm 的默认行为。

## `--wait` 和 `--atomic` 标志

`helm install` 和 `helm upgrade` 的另一对重要标志修改了 Helm 操作的成功标准。这些是 `--wait` 和 `--atomic` 标志。

`--wait`标志在几个方面修改了 Helm 客户端的行为。首先，在 Helm 运行安装时，它会保持活动状态一段固定的时间窗口（可以使用`--timeout`标志修改），在此期间它会监视 Kubernetes。它会轮询 Kubernetes API 服务器以获取有关由图表创建的所有 Pod 运行对象的信息。例如，`DaemonSet`、`Deployment`和`StatefulSet`都会创建 Pod。因此，使用`--wait`的 Helm 将跟踪这些对象，直到它们创建的 Pod 被 Kubernetes 标记为`Running`。

在正常安装或升级中，Helm 会在 Kubernetes API 服务器接受清单后立即将发布标记为成功。这类似于将包成功安装视为将包内容写入正确存储位置的软件包管理器。

但是使用`--wait`时，安装的成功标准会发生变化。除非（1）Kubernetes API 服务器接受清单，并且（2）图表创建的所有 Pod 在 Helm 超时到期之前达到`Running`状态，否则不会认为图表安装成功。

因此，使用`--wait`进行安装可能会因多种原因而失败，包括网络延迟、慢调度器、繁忙节点、慢镜像拉取以及容器无法启动的明确失败。

这种行为被视为一种理想的结果，操作员使用`helm install --wait`确保图表不仅成功安装，而且生成的应用程序正确启动。然而，这在故障排除时引入了一些复杂因素。瞬态故障可能导致 Helm 失败，但稍后由 Kubernetes 解决。例如，延迟的镜像拉取可能导致 Helm 发布被标记为失败，尽管几分钟后镜像拉取完成并且应用程序可以启动。

考虑到这一点，`helm install --wait`是确保发布完全运行的好工具。但在自动化系统（如 CI）中使用时，可能会导致偶发失败。建议在 CI 中使用`--wait`的一个方法是设置较长的`--timeout`（五到十分钟），以确保 Kubernetes 有足够时间解决任何瞬态故障。

第二种策略是使用`--atomic`标志，而不是`--wait`标志。该标志引起的行为与`--wait`相同，除非发布失败。然后，它不会将发布标记为`failed`并退出，而是自动回滚到上次成功的发布。在自动化系统中，`--atomic`标志对于故障更具抵抗力，因为其最终结果不太可能是失败。（请注意，回滚成功也无法保证。）

就像`--wait`可以因 Kubernetes 自身解决的传递原因将发布标记为失败一样，`--atomic`可能因同样的原因触发不必要的回滚。因此，建议在与 CI 系统一起使用`--atomic`时使用更长的`--timeout`持续时间。

## 使用`--force`和`--cleanup-on-fail`进行升级

我们将讨论的最后两个标志修改了 Helm 处理升级细节的方式。

当 Helm 升级管理 pods 的资源（如`Pod`、`Deployment`和`StatefulSet`）时，`--force`标志会修改 Helm 的行为。通常情况下，当 Kubernetes 收到修改此类对象的请求时，它会确定是否需要重新启动此资源管理的 pods。例如，一个`Deployment`可能运行五个 pod 的副本。但如果 Kubernetes 收到`Deployment`对象的更新，它只会在修改了特定字段时重新启动那些 pods。

有时候，Helm 用户想要确保 pod 被重新启动。这时就需要用到`--force`标志。与修改`Deployment`（或类似对象）不同，它会删除并重新创建。这会强制 Kubernetes 删除旧的 pods 并创建新的。设计上，使用`--force`会导致停机时间。尽管通常只有几秒钟的停机时间，但仍然会有停机。建议只在情况明确需要时使用`--force`，而不是默认选项。例如，核心维护者不建议在部署到生产环境的 CI 流水线中使用`--force`。

修改升级行为的另一种方法是使用`--cleanup-on-fail`标志。与`--force`类似，此标志指示 Helm 执行额外的工作。

考虑一种情况，您安装了一个创建一个 Kubernetes `Secret` 的图表。创建了图表的新版本，并创建了第二个`Secret`。但在安装过程中的某个时候，Helm 遇到错误并标记了发布为失败。第二个`Secret`有可能被悬而未决。如果使用了`--wait`或`--atomic`，这种情况更有可能发生，因为这些操作可能在 Kubernetes 接受清单并创建资源后失败。

在失败时，`--cleanup-on-fail`标志将尝试修复此情况。它会请求删除在升级期间*新创建*的每个对象。使用它可能会稍微增加调试的难度（特别是如果失败是由于新创建的对象导致的），但如果您不想冒在失败后留下未使用的对象的风险，这是非常有用的。

# 结论

Helm 命令行工具提供了许多有用的命令。虽然基本命令已在前一章介绍过，本章重点介绍了 Helm 中的其他一些有用命令。最后，我们还重新审视了安装和升级命令，了解了与这些命令一起使用的一些更复杂的特性。

然而，并非所有命令都在这里讨论过。在接下来的章节中，我们将查看用于创建和打包图表的命令，用于签名和验证软件包的命令，以及更多处理仓库的命令。
