# 第七章：Argo CD

在前一章中，您学习了有关 Tekton 和其他引擎（如 GitHub Actions）如何实现项目的持续集成（CI）部分。

虽然 CI 很重要，因为这是构建应用程序并检查是否存在故障（运行单元测试、组件测试等）的地方，但仍有一个缺失的部分：如何使用 GitOps 方法论将此应用程序部署到环境（一个 Kubernetes 集群），而不是创建一个运行 `kubectl`/`helm` 命令的脚本。

正如 Daniel Bryant 所说：“如果您以前没有使用 SSH 在生产环境中部署应用程序，请不要使用 `kubectl` 在 Kubernetes 中执行此操作。”

在本章中，我们将介绍 Argo CD，这是一个用于 Kubernetes 的声明式 GitOps 连续交付（CD）工具。在本节的第一部分中，我们将看到如何使用 Argo CD 部署应用程序（Recipes 7.1 和 7.2）。

Argo CD 不仅支持普通 Kubernetes 清单的部署，还支持 Kustomize 项目（Recipe 7.3）和 Helm 项目（Recipe 7.4）的部署。

Kubernetes 中典型的操作是对容器的新版本进行滚动更新，Argo CD 与另一个工具集成，使此过程更加顺畅（Recipe 7.5）。

交付复杂的应用程序可能需要在何时以及如何部署和发布应用程序时进行一些编排（Recipes 7.7 和 7.8）。

我们将看到如何：

+   安装并部署第一个应用程序。

+   使用自动部署和自愈应用程序。

+   当发布新容器时执行滚动更新。

+   给出执行顺序。

在本章中，我们使用 `https://github.com/gitops-cookbook/gitops-cookbook-sc.git` GitHub 仓库作为源目录。要在本章中成功运行它，您应该 fork 它并在提供的示例 YAML 文件中使用它。

# 7.1 使用 Argo CD 部署应用程序

## 问题

您希望 Argo CD 部署在 Git 仓库中定义的应用程序。

## 解决方案

创建一个 `Application` 资源来设置 Argo CD 以部署该应用程序。

要安装 Argo CD，请创建 `argocd` 命名空间并应用 Argo CD 安装清单：

```
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/v2.3.4/manifests/install.yaml
```

让我们让 Argo CD 部署一个简单的 Web 应用程序，显示一个配置了颜色的方框。该应用由三个 Kubernetes 清单文件组成，包括一个 `Namespace`、一个 `Deployment` 和一个 `Service` 定义。

这些文件位于书的存储库中的 [ch07/bgd](https://oreil.ly/DAH50) 文件夹中。

所有这些文件在 Argo CD 中被称为一个 `Application`。因此，您必须将其定义为这样的文件，以将这些清单应用到您的集群中。

让我们检查用于部署应用程序的 Argo CD `Application` 资源文件：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgd-app
  namespace: argocd ![1](img/1.png)
spec:
  destination: ![2](img/2.png)
    namespace: bgd
    server: https://kubernetes.default.svc 
  project: default ![3](img/3.png)
  source:
    repoURL: https://github.com/gitops-cookbook/gitops-cookbook-sc.git ![4](img/4.png)
    path: ch07/bgd ![5](img/5.png)
    targetRevision: main ![6](img/6.png)
```

![1](img/#co_argo_cd_CO1-1)

安装 Argo CD 的命名空间

![2](img/#co_argo_cd_CO1-2)

目标集群和命名空间

![3](img/#co_argo_cd_CO1-3)

在 Argo CD 的默认项目中安装应用程序

![4](img/#co_argo_cd_CO1-4)

YAML 所在的清单仓库

![5](img/#co_argo_cd_CO1-5)

查找清单的路径

![6](img/#co_argo_cd_CO1-6)

要检出的分支

在终端窗口中，运行以下命令注册 Argo CD 应用程序：

```
kubectl apply -f manual-bgd-app.yaml
```

此时，应用程序已注册为 Argo CD 应用程序。

您可以使用`argocd`或 UI 来检查状态；也可以使用 CLI 运行以下命令来列出应用程序：

```
argocd app list
```

输出如下：

```
NAME     CLUSTER                         NAMESPACE  PROJECT  STATUS
bgd-app  https://kubernetes.default.svc  bgd        default  OutOfSync
```

这里重要的字段是`STATUS`。它是`OutOfSync`，这意味着应用程序已注册，当前状态（在本例中为未部署任何应用程序）与 Git 存储库中的内容（应用程序部署文件）之间存在偏差。

如果从`bgd`命名空间获取所有 Pod，您会注意到没有正在运行的 Pod：

```
kubectl get pods -n bgd

No resources found in bgd namespace.
```

Argo CD 默认不会自动同步应用程序。它只显示差异，用户可以通过触发同步操作来修复。

使用 CLI，在终端中运行以下命令来同步应用程序：

```
argocd app sync bgd-app
```

命令的输出显示了关于部署的所有重要信息：

```
Name:               bgd-app
Project:            default
Server:             https://kubernetes.default.svc
Namespace:          bgd
URL:                https://openshift-gitops-server-openshift-gitops.apps.openshift.sotogcp.com/applications/bgd-app
Repo:               https://github.com/lordofthejars/gitops-cookbook-sc.git
Target:             main
Path:               ch07/bgd
SyncWindow:         Sync Allowed
Sync Policy:        <none>
Sync Status:        Synced to main (384cd3d)
Health Status:      Progressing

Operation:          Sync
Sync Revision:      384cd3d21c534e75cb6b1a6921a6768925b81244
Phase:              Succeeded
Start:              2022-06-16 14:45:12 +0200 CEST
Finished:           2022-06-16 14:45:13 +0200 CEST
Duration:           1s
Message:            successfully synced (all tasks run)

GROUP  KIND        NAMESPACE  NAME  STATUS   HEALTH       HOOK  MESSAGE
       Namespace   bgd        bgd   Running  Synced             namespace/bgd created
       Service     bgd        bgd   Synced   Healthy            service/bgd created
apps   Deployment  bgd        bgd   Synced   Progressing        deployment.apps/bgd created
       Namespace              bgd   Synced
```

您也可以通过单击 UI 中显示的`SYNC`按钮来同步应用程序，如图 7-1 所示。

![同步按钮用于同步应用程序](img/gocb_0701.png)

###### 图 7-1\. Argo CD Web 控制台

如果从`bgd`命名空间获取所有 Pod，您会注意到有一个 Pod 正在运行：

```
kubectl get pods -n bgd

NAME                   READY   STATUS    RESTARTS   AGE
bgd-788cb756f7-jll9n   1/1     Running   0          60s
```

同样适用于服务：

```
kubectl get services -n bgd

NAME   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
bgd    ClusterIP   172.30.35.199   <none>        8080:32761/TCP ![1](img/1.png)
```

![1](img/#co_argo_cd_CO2-1)

暴露的端口为 32761

在接下来的章节中，您需要访问部署的服务以验证其部署情况。有几种方法可以访问部署到 Minikube 的服务；在接下来的章节中，我们使用 Minikube IP 和服务的暴露端口。

在终端窗口中运行以下命令以获取 Minikube IP：

```
minikube ip -p gitops
192.168.59.100
```

打开浏览器窗口，设置前面的 IP，后面跟上暴露的端口（例如`192.168.59.100:32761`），并访问服务以验证盒子中圆圈的颜色是否为蓝色，如图 7-2 所示。

![蓝色框应用](img/gocb_0704.png)

###### 图 7-2\. 已部署的应用程序

## 讨论

现在是更新应用程序部署文件的时候了。这次我们将更改*bgd-deployment.yaml*文件中定义的环境变量的值。

在文件编辑器中打开*ch07/bgd/bgd-deployment.yaml*，将`COLOR`环境变量的值从`blue`改为`green`：

```
spec:
  containers:
  - image: quay.io/redhatworkshops/bgd:latest
    name: bgd
    env:
    - name: COLOR
      value: "green"
```

在终端中运行以下命令来提交并推送文件，以便 Argo CD 可以使用更改：

```
git add .
git commit -m "Updates color"

git push origin main
```

推送更改后，再次检查应用程序的状态：

```
argocd app list

NAME     CLUSTER                         NAMESPACE  PROJECT  STATUS
bgd-app  https://kubernetes.default.svc  bgd        default  Sync
```

我们看到应用程序状态为`Sync`。这是因为 Argo CD 使用轮询方法检测部署和 Git 定义之间的差异。一段时间后（默认为 3 分钟），应用程序状态将为`OutOfSync`：

```
argocd app list
NAME     CLUSTER                         NAMESPACE  PROJECT  STATUS     HEALTH
bgd-app  https://kubernetes.default.svc  bgd        default  OutOfSync  Healthy
```

要同步更改，请运行`sync`子命令：

```
argocd app sync bgd-app
```

几秒钟后，访问服务，并验证圆圈是否为绿色，如图 7-3 所示。

![绿色方框应用](img/gocb_0705.png)

###### 图 7-3\. 已部署的应用程序

要删除应用程序，请使用 CLI 工具或 UI：

```
argocd app delete bgd-app
```

同时，撤消 Git 仓库中所做的更改，以获取应用程序的初始版本并推送它们：

```
git revert HEAD

git push origin main
```

# 7.2 自动同步

## 问题

您希望 Argo CD 在有更改时自动更新资源。

## 解决方案

使用具有`automated`策略的`syncPolicy`部分。

当 Argo CD 检测到 Git 中的清单与 Kubernetes 集群中定义的差异时，它可以自动同步应用程序。

自动同步的好处是无需登录到 Argo CD API，避免了涉及的安全问题（管理秘钥、网络等）和使用`argocd`工具。相反，当清单发生变化并推送到 Git 仓库时，跟踪 Git 仓库的清单会自动应用。

让我们修改先前的 Argo CD 清单文件（`Application`），添加`sync​Po⁠licy`部分，以便自动部署更改：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgd-app
  namespace: argocd
spec:
  destination:
    namespace: bgd
    server: https://kubernetes.default.svc 
  project: default 
  source: 
    path: ch07/bgd
    repoURL: https://github.com/gitops-cookbook/gitops-cookbook-sc.git
    targetRevision: main
  syncPolicy: ![1](img/1.png)
    automated: {} ![2](img/2.png)
```

![1](img/#co_argo_cd_CO3-1)

启动同步策略配置部分

![2](img/#co_argo_cd_CO3-2)

Argo CD 会自动同步仓库

在这一点上，我们可以通过运行以下命令将`Application`文件应用到运行中的集群中：

```
kubectl apply -f bgd/bgd-app.yaml
```

现在，Argo CD 部署应用程序，而无需执行任何其他命令。

运行`kubectl`命令或在 Argo CD UI 中检查，验证部署正在进行：

```
kubectl get pods -n bgd

NAME                   READY   STATUS    RESTARTS   AGE
bgd-788cb756f7-jll9n   1/1     Running   0          60s
```

访问服务，并验证圆圈是否为蓝色，如图 7-4 所示。

![蓝色方框应用](img/gocb_0704.png)

###### 图 7-4\. 已部署的应用程序

要删除应用程序，请使用 CLI 工具或 UI：

```
argocd app delete bgd-app
```

## 讨论

尽管 Argo CD 会自动部署应用程序，但出于安全考虑，它使用一些默认的保守策略。

其中两个是删除已删除资源的修剪和应用程序的自愈，以防在 Kubernetes 集群中直接进行更改而不是通过 Git。

默认情况下，当 Argo CD 检测到 Git 中不再可用时，它不会删除（修剪）任何资源，并且资源状态将是`OutOfSync`。如果希望 Argo CD 删除这些资源，可以通过两种方式实现。

第一种方法是手动使用`-prune`选项调用同步：

```
argocd app sync --prune
```

第二种方法是通过在`automated`部分中将`prune`属性设置为`true`，让 Argo CD 自动删除修剪资源：

```
syncPolicy:
  automated:
    prune: true ![1](img/1.png)
```

![1](img/#co_argo_cd_CO4-1)

启用自动修剪

另一个影响应用程序自动更新方式的重要概念是自我修复。

Argo CD 配置为不纠正手动在集群中进行的任何漂移。例如，Argo CD 允许在集群中直接执行 `kubectl patch` 改变应用程序的任何配置参数。

让我们看看它的效果。

圆圈的颜色设定为环境变量（`COLOR`）。

现在，让我们使用 `kubectl patch` 命令将 `COLOR` 环境变量改为 `green`。

在终端中运行以下命令：

```
kubectl -n bgd patch deploy/bgd \
--type='json' -p='[{"op": "replace", "path": "/
spec/template/spec/containers/0/env/0/value", "value":"green"}]'
```

等待部署完成：

```
kubectl rollout status deploy/bgd -n bgd
```

如果刷新浏览器，您应该看到现在是绿色圆圈，如 图 7-5 所示。

![绿色方框应用程序](img/gocb_0705.png)

###### 图 7-5\. 已部署的应用程序

查看 Argo CD 的同步状态，您会看到它处于 `OutOfSync` 状态，因为应用程序与 Git 仓库中定义的定义（`COLOR: blue`）不一致。

由于 `selfHeal` 属性的默认设置为 `false`，Argo CD 不会回滚以纠正此漂移。

让我们移除该应用程序并部署一个新的，但在这种情况下将 `selfHeal` 设置为 `true`：

```
argocd app delete bgd-app
```

让我们启用 `selfHealing` 属性，并重复实验：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgd-app
  namespace: argocd
spec:
  destination:
    namespace: bgd
    server: https://kubernetes.default.svc 
  project: default 
  source: 
    path: ch07/bgd
    repoURL: https://github.com/gitops-cookbook/gitops-cookbook-sc.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true ![1](img/1.png)
```

![1](img/#co_argo_cd_CO5-1)

将 `selfHeal` 设置为 `true` 以纠正任何漂移

然后在终端应用该资源：

```
kubectl apply -f bgd/heal-bgd-app.yaml
```

重复之前的步骤：

1.  打开浏览器检查圆圈是否为蓝色。

1.  重新执行 `kubectl -n bgd patch deploy/bgd ...` 命令。

1.  刷新浏览器并检查圆圈是否仍然是蓝色。

Argo CD 通过 `patch` 命令来纠正引入的漂移，将应用程序同步到 Git 仓库中定义的正确状态。

要移除该应用程序，请使用 CLI 工具或 UI：

```
argocd app delete bgd-app
```

## 参见

+   [Argo CD 自动同步策略](https://oreil.ly/mw4b2)

+   [Argo CD 同步选项](https://oreil.ly/wIleG)

# 7.3 Kustomize 集成

## 问题

您希望使用 Argo CD 部署 Kustomize 清单。

## 解决方案

Argo CD 支持多种定义 Kubernetes 清单的方式：

+   Kustomize

+   Helm

+   Ksonnet

+   Jsonnet

您也可以将支持的方式扩展到自定义的方式，但这超出了本书的范围。

如果存在以下任一文件，则 Argo CD 将检测到 Kustomize 项目：*kustomization.yaml*、*kustomization.yml* 或 *Kustomization*。

让我们部署相同的 BGD 应用程序，但这次使用 Kustomize 清单进行部署。

此外，我们将设置 `kustomize` 以覆盖 `COLOR` 环境变量为黄色。

仓库中定义的 Kustomize 文件如下所示：

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: bgdk
resources:
- ../base ![1](img/1.png)
- bgdk-ns.yaml ![2](img/2.png)
patchesJson6902: ![3](img/3.png)
  - target: ![4](img/4.png)
      version: v1
      group: apps
      kind: Deployment
      name: bgd
      namespace: bgdk
    patch: |- ![5](img/5.png)
      - op: replace
        path: /spec/template/spec/containers/0/env/0/value
        value: yellow
```

![1](img/#co_argo_cd_CO6-1)

包含标准部署文件的目录（蓝色圆圈）

![2](img/#co_argo_cd_CO6-2)

用于创建命名空间的特定文件

![3](img/#co_argo_cd_CO6-3)

补丁标准部署文件

![4](img/#co_argo_cd_CO6-4)

对部署文件进行补丁

![5](img/#co_argo_cd_CO6-5)

将环境变量值覆盖为 `yellow`

###### 注意

您无需创建此文件，因为它已存储在 Git 存储库中。

创建以下*Application*文件以部署应用程序：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgdk-app
  namespace: argocd
spec:
  destination:
    namespace: bgdk
    server: https://kubernetes.default.svc 
  project: default 
  source: 
    path: ch07/bgdk/bgdk
    repoURL: https://github.com/gitops-cookbook/gitops-cookbook-sc.git
    targetRevision: main
  syncPolicy:
    automated: {}
```

此时，我们可以通过运行以下命令将*Application*文件应用于正在运行的集群：

```
kubectl apply -f bgdk/bgdk-app.yaml
```

访问服务，您会注意到圆圈是黄色而不是蓝色。

要删除应用程序，请使用 CLI 工具或 UI：

```
argocd app delete bgdk-app
```

## 讨论

我们可以明确指定使用哪个工具，覆盖 Argo CD 在*Application*文件中使用的默认算法。例如，我们可以根据*kustomization.yaml*文件的存在使用简单的目录策略：

```
source:
  directory: ![1](img/1.png)
    recurse: true
```

![1](img/#co_argo_cd_CO7-1)

覆盖始终使用简单的目录策略

可能的策略包括：`directory`、`chart`、`helm`、`kustomize`、`path`和`plugin`。

###### 提示

当使用 Argo CD 时，Kustomize 的所有已见内容都是有效的。

## 参见

+   第四章

+   [argo-cd/application-crd.yaml on GitHub](https://oreil.ly/EIxY1)

+   [Argo CD 工具检测](https://oreil.ly/DJbiU)

# 7.4 Helm 集成

## 问题

您想使用 Argo CD 来部署 Helm 清单。

## 解决方案

当检测到部署目录中存在 Helm 项目（即*Chart.yaml*文件存在）时，Argo CD 支持将 Helm Charts 安装到集群中。

让我们部署相同的 BGD 应用程序，但在这种情况下，部署为 Helm 清单。

项目的布局是在您先前克隆的 GitHub 存储库中已经创建的一个简单的 Helm 布局：

```
├── Chart.yaml
├── charts
├── templates
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ns.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml
```

创建*bgdh/bgdh-app.yaml*文件以定义 Argo CD 应用程序：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgdh-app
  namespace: argocd
spec:
  destination:
    namespace: bgdh
    server: https://kubernetes.default.svc 
  project: default 
  source: 
    path: ch07/bgdh
    repoURL: https://github.com/gitops-cookbook/gitops-cookbook-sc.git
    targetRevision: main
  syncPolicy:
    automated: {}
```

此时，我们可以通过运行以下命令将*Application*文件应用到正在运行的集群中：

```
kubectl apply -f bgdh/bgdh-app.yaml
```

验证 Pod 是否在*bgdh*命名空间中运行：

```
kubectl get pods -n bgdh

NAME                        READY   STATUS    RESTARTS   AGE
bgdh-app-556c46fcd6-ctfkf   1/1     Running   0          5m43s
```

要删除应用程序，请使用 CLI 工具或 UI：

```
argocd app delete bgdh-app
```

## 讨论

Argo CD 将构建环境变量填充到 Helm 清单中（实际上还支持 Kustomize、Jsonnet 和自定义工具）。

设置以下变量：

+   `ARGOCD_APP_NAME`

+   `ARGOCD_APP_NAMESPACE`

+   `ARGOCD_APP_REVISION`

+   `ARGOCD_APP_SOURCE_PATH`

+   `ARGOCD_APP_SOURCE_REPO_URL`

+   `ARGOCD_APP_SOURCE_TARGET_REVISION`

+   `KUBE_VERSION`

+   `KUBE_API_VERSIONS`

在以下代码片段中，您可以查看应用程序名称的使用：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgdh-app
  namespace: openshift-gitops
spec:
  destination:
  ...
  source:
    path: ch07/bgd
    ...
    helm: ![1](img/1.png)
      parameters: ![2](img/2.png)
      - name: app ![3](img/3.png)
        value: $ARGOCD_APP_NAME ![4](img/4.png)
```

![1](img/#co_argo_cd_CO8-1)

特定的 Helm 部分

![2](img/#co_argo_cd_CO8-2)

设置额外的参数，与在*values.yaml*中设置它们一样，但更高优先级

![3](img/#co_argo_cd_CO8-3)

参数的名称

![4](img/#co_argo_cd_CO8-4)

参数的值，在本例中来自 Build Env var

Argo CD 可以使用不同的*values.yaml*文件或设置参数值来覆盖*values.yaml*中定义的值：

```
argocd app set bgdh-app --values new-values.yaml

argocd app set bgdh-app -p service.type=LoadBalancer
```

请注意，值文件必须与 Helm Chart 存储在同一个 Git 存储库中。

###### 注意

Argo CD 还支持 Helm 钩子。

## 参见

+   第五章

+   [*argo-cd/application-crd.yaml* on GitHub](https://oreil.ly/EIxY1)

# 7.5 镜像更新器

## 问题

您希望 Argo CD 在发布时自动部署容器镜像。

## 解决方案

使用 [Argo CD Image Updater](https://oreil.ly/kztMq) 检测容器注册表上的更改，并更新部署文件。

在开发过程中最重复的任务之一是部署容器镜像的新版本。

使用纯 Argo CD 解决方案后，将容器镜像发布到容器注册表后，我们需要更新指向新容器镜像的 Kubernetes/Kustomize/Helm 清单文件，并将结果推送到 Git 仓库。

此过程意味着：

1.  克隆仓库

1.  解析 YAML 文件并相应更新它们

1.  提交并推送更改

在持续集成阶段为每个仓库定义这些样板任务。虽然这种方法有效，但可以自动化，以便集群能够检测到推送到容器注册表的新镜像，并更新指向更新版本的当前部署文件。

这正是 Argo CD Image Updater（*ArgoCD IU*）所做的。它是一个 Kubernetes 控制器，监控新的容器版本，并更新在 Argo CD `Application` 文件中定义的清单。

Argo CD IU 的生命周期及其与 Argo CD 的关系显示在 图 7-6 中。

![Argo CD 和 Argo CD Image Updater 的生命周期](img/gocb_0706.png)

###### 图 7-6\. Argo CD Image Updater 生命周期

目前，Argo CD IU 仅更新 Kustomize 或 Helm 的清单。在 Helm 的情况下，需要支持使用参数（`image.tag`）指定镜像的标签。

让我们在与 Argo CD 相同的命名空间中安装控制器：

```
kubectl apply -f \
https://raw.githubusercontent.com/argoproj-labs/argocd-imageupdater/v0.12.0/manifests/install.yaml -n argocd
```

验证安装过程，检查控制器的 Pod 状态是否为 `Running`：

```
kubectl get pods -n argocd

NAME                                                         READY   STATUS    RESTARTS   AGE
argocd-image-updater-59c45cbc5c-kjjtp                        1/1     Running   0          40h
```

在使用 Argo CD IU 之前，我们创建一个代表 Git 凭据的 Kubernetes Secret，以便更新的清单可以推送到仓库。该密钥必须在 Argo CD 命名空间中，并且在本例中，我们将其命名为 `git-creds`。

```
kubectl -n argocd create secret generic git-creds \ --from-literal=username=<git_user> \
--from-literal=password=<git_password_or_token>
```

最后，让我们使用一些特殊的注解来为 `Application` 清单添加注解，以便控制器可以开始监视注册表：

image-list

指定一个或多个（逗号分隔的）被考虑更新的镜像。

write-back-method

传播新版本的方法。实现了 `git` 和 `argocd` 方法以更新到更高版本的镜像。Git 方法将更改提交到 Git 仓库。Argo CD 使用 Kubernetes/ArgoCD API 更新资源。

还有更多配置选项，但前面的是开始的最重要部分。

让我们创建一个带有 Argo CD `Application` 注解的清单：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgdk-app
  namespace: argocd
  annotations: ![1](img/1.png)
    argocd-image-updater.argoproj.io/image-list: myalias=quay.io/rhdevelopers/bgd ![2](img/2.png)
    argocd-image-updater.argoproj.io/write-back-method: git:secret:openshift-gitops/git-creds ![3](img/3.png)
    argocd-image-updater.argoproj.io/git-branch: main ![4](img/4.png)
spec:
  destination:
    namespace: bgdk
    server: https://kubernetes.default.svc 
  project: default 
  source: 
    path: ch07/bgdui/bgdk
    repoURL: https://github.com/gitops-cookbook/gitops-cookbook-sc.git
    targetRevision: main
  syncPolicy:
    automated: {}
```

![1](img/#co_argo_cd_CO9-1)

添加注解部分

![2](img/#co_argo_cd_CO9-2)

设置监视的镜像名称

![3](img/#co_argo_cd_CO9-3)

配置使用 Git 作为 `write-back-method`，设置凭据的位置（`<namespace>/<secretname>`）

![4](img/#co_argo_cd_CO9-4)

设置推送更改的分支

现在应用清单以部署应用程序的第一个版本，并启用 Argo CD IU 在推送新镜像到容器注册表时更新仓库：

```
kubectl apply -f bgdui/bgdui-app.yaml
```

此时，版本 `1.0.0` 在 `bgdk` 命名空间中正在运行，您可以像以前一样访问它。让我们生成一个新的容器版本，以验证新图像是否在仓库中。

为简化流程，我们将标记容器版本 `1.1.0` 为新版本。

转到本章开头创建的 Quay 仓库，转到标签部分，点击齿轮图标，选择 `Add New Tag` 来创建一个新的容器，如图 图 7-7 所示。

![在 Quay 中创建新标签](img/gocb_0707.png)

###### 图 7-7\. 标记容器

将标签设置为`1.1.0`，如图 图 7-8 所示。

![设置新标签](img/gocb_0708.png)

###### 图 7-8\. 标记容器

在此步骤之后，您应该已经创建了一个新容器，如图 图 7-9 所示。

等待大约两分钟，直到检测到更改并触发仓库更新。

![在 Quay 仓库中放置一个新容器](img/gocb_0709.png)

###### 图 7-9\. 最终结果

要验证触发过程，请检查控制器的日志：

```
kubectl logs argocd-image-updater-59c45cbc5c-kjjtp -f -n argocd

...
time="2022-06-20T21:19:05Z" level=info msg="Setting new image to quay.io/rhdevelopers/bgd:1.1.0" alias=myalias application=bgdk-app image_name=rhdevelopers/bgd image_tag=1.0.0 registry=quay.io
time="2022-06-20T21:19:05Z" level=info msg="Successfully updated image 'quay.io/rhdevelopers/bgd:1.0.0' to 'quay.io/rhdevelopers/bgd:1.1.0', but pending spec update (dry run=false)" alias=myalias application=bgdk-app image_name=rhdevelopers/bgd image_tag=1.0.0 registry=quay.io ![1](img/1.png)
time="2022-06-20T21:19:05Z" level=info msg="Committing 1 parameter update(s) for application bgdk-app" application=bgdk-app
...
```

![1](img/#co_argo_cd_CO10-1)

检测到更改并更新图像

之后，如果检查仓库，您将看到一个名为 `.argocd-source-bgdk-app.yaml` 的新 Kustomize 文件，将图像值更新为新容器，如图 图 7-10 所示。

![更新至新容器的新 Kustomize 文件](img/gocb_0710.png)

###### 图 7-10\. 新 Kustomize 文件更新至新容器

现在 Argo CD 可以检测到更改并正确地更新集群以适应新的镜像。

要删除应用程序，请使用 CLI 工具或 UI：

```
argocd app delete bgdk-app
```

## 讨论

更新策略定义了 Argo CD IU 如何查找新版本。如果没有更改，Argo CD IU 使用语义版本来检测最新版本。

可以添加一个可选的版本约束字段来限制允许自动更新的版本。要仅更新补丁版本，可以按以下片段更改 `image-list` 注释：

```
argocd-image-updater.argoproj.io/image-list: myalias=quay.io/rhdevelopers/bgd:1.2.x
```

Argo CD Image Updater 可以更新到具有最新构建日期的镜像：

```
argocd-image-updater.argoproj.io/myalias.update-strategy: latest
argocd-image-updater.argoproj.io/myimage.allow-tags: regexp:^[0-9a-f]{7}$ ![1](img/1.png)
```

![1](img/#co_argo_cd_CO11-1)

限制用于更新的标签

摘要更新策略将使用镜像摘要更新应用程序的镜像标签：

```
argocd-image-updater.argoproj.io/myalias.update-strategy: digest
```

到目前为止，容器存储在公共注册表中。如果仓库是私有的，Argo CD Image Updater 需要读取仓库以检测任何更改。

首先，创建一个新的代表容器注册凭据的秘密：

```
kubectl create -n argocd secret docker-registry quayio --docker-server=quay.io --docker-username=$QUAY_USERNAME --docker-password=$QUAY_PASSWORD
```

Argo CD Image Updater 使用 `ConfigMap` 作为配置源，这是注册私有容器注册表的位置。创建一个新的 `ConfigMap` 清单设置支持的注册表：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config ![1](img/1.png)
data:
  registries.conf: |
    registries: ![2](img/2.png)
    - name: RedHat Quay ![3](img/3.png)
      api_url: https://quay.io ![4](img/4.png)
      prefix: quay.io ![5](img/5.png)
      insecure: yes
      credentials: pullsecret:argocd/quayio ![6](img/6.png)
```

![1](img/#co_argo_cd_CO12-1)

Argo CD IU 的 `ConfigMap` 名称

![2](img/#co_argo_cd_CO12-2)

注册所有注册表的位置

![3](img/#co_argo_cd_CO12-3)

用于标识的名称

![4](img/#co_argo_cd_CO12-4)

服务的网址

![5](img/#co_argo_cd_CO12-5)

容器镜像中使用的前缀

![6](img/#co_argo_cd_CO12-6)

从存储在 `argocd` 命名空间中的 `quayio` 密钥获取凭据

Argo CD Image Updater 提交更新，附带默认消息：

```
commit 3caf0af8b7a26de70a641c696446bbe1cd04cea8 (HEAD -> main, origin/main)
Author: argocd-image-updater <noreply@argoproj.io>
Date:   Thu Jun 23 09:41:00 2022 +0000

    build: automatic update of bgdk-app

    updates image rhdevelopers/bgd tag '1.0.0' to '1.1.0'
```

我们可以将默认提交消息更新为符合您要求的消息。在 ArgoCD IU `argocd-image-updater-config` `ConfigMap` 中配置 `git.commit-message-template` 键与消息：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config ![1](img/1.png)
data:
  git.user: alex ![2](img/2.png)
  git.email: alex@example.com ![3](img/3.png)
  git.commit-message-template: | ![4](img/4.png)
    build: automatic update of {{ .AppName }} ![5](img/5.png)

    {{ range .AppChanges -}} ![6](img/6.png)
    updates image {{ .Image }} tag '{{ .OldTag }}' to '{{ .NewTag }}' ![7](img/7.png) ![8](img/8.png) ![9](img/9.png)
    {{ end -}}
```

![1](img/#co_argo_cd_CO13-1)

Argo CD IU `ConfigMap`

![2](img/#co_argo_cd_CO13-2)

提交用户

![3](img/#co_argo_cd_CO13-3)

提交的电子邮件

![4](img/#co_argo_cd_CO13-4)

Golang `text/template` 内容

![5](img/#co_argo_cd_CO13-5)

应用的名称

![6](img/#co_argo_cd_CO13-6)

更新执行的更改列表

![7](img/#co_argo_cd_CO13-7)

镜像名称

![8](img/#co_argo_cd_CO13-8)

先前的容器标签

![9](img/#co_argo_cd_CO13-9)

新的容器标签

###### 注意

在 `ConfigMap` 更改时，请记得重新启动 Argo CD UI 控制器：

```
kubectl rollout restart deployment argocd-image-updater -n argocd
```

## 参见

+   [Argo CD Image Updater](https://oreil.ly/kztMq)

# 7.6 从私有 Git 仓库部署

## 问题

您希望 Argo CD 部署清单。

## 解决方案

使用 Argo CD CLI/UI 或 YAML 文件注册仓库的凭据信息（用户名/密码/令牌/密钥）。

在 Argo CD 中，您有两种方法可以注册带有其凭据的 Git 仓库。一种方法是使用 Argo CD CLI/Argo CD UI 工具。要在 Argo CD 中注册私有仓库，请通过以下命令设置用户名和密码：

```
argocd repo add https://github.com/argoproj/argocd-example-apps \
--username <username> --password <password>
```

或者，我们也可以使用 Argo CD UI 进行注册。在浏览器中打开 Argo CD UI，点击“设置/仓库”按钮（带有齿轮图标），如图 7-11 所示。

![选择仓库部分](img/gocb_0711.png)

###### 图 7-11\. 设置菜单

然后点击“使用 HTTPS 连接 Repo”按钮，并填写所需数据，如图 7-12 所示。

![使用 HTTPS 凭据配置仓库](img/gocb_0712.png)

###### 图 7-12\. 仓库的配置

最后，单击“连接”按钮以测试是否可以建立连接并将仓库添加到 Argo CD 中。

另一种方法是创建一个带有仓库和凭据信息的 Kubernetes Secret 清单文件：

```
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd ![1](img/1.png)
  labels:
    argocd.argoproj.io/secret-type: repository ![2](img/2.png)
stringData:
  type: git
  url: https://github.com/argoproj/private-repo ![3](img/3.png)
  password: my-password ![4](img/4.png)
  username: my-username ![5](img/5.png)
```

![1](img/#co_argo_cd_CO14-1)

在 Argo CD 命名空间中创建一个密钥

![2](img/#co_argo_cd_CO14-2)

将密钥类型设置为 `repository`

![3](img/#co_argo_cd_CO14-3)

要注册的仓库的网址

![4](img/#co_argo_cd_CO14-4)

访问密码

![5](img/#co_argo_cd_CO14-5)

访问用户名

如果您应用此文件，则效果与手动操作相同。

在这一点上，每当我们在 `Application` 资源中定义一个 `repoURL` 值，并且使用了为身份验证注册的存储库 URL，Argo CD 将使用注册的凭据登录。

## 讨论

除了为访问私有 Git 存储库设置用户名和密码之外，Argo CD 还支持其他方法，如令牌、TLS 客户端证书、SSH 私钥或 GitHub App 凭据。

让我们看一些使用 Argo CD CLI 或 Kubernetes Secrets 的示例。

要配置 TLS 客户端证书：

```
argocd repo add https://repo.example.com/repo.git \
--tls-client-cert-path ~/mycert.crt \
--tls-client-cert-key-path ~/mycert.key
```

对于 SSH，您只需设置 SSH 私钥的位置：

```
argocd repo add git@github.com:argoproj/argocd-example-apps.git \
--ssh-privatekey-path ~/.ssh/id_rsa
```

或者使用 Kubernetes Secret：

```
apiVersion: v1
kind: Secret
metadata:
  name: private-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:argoproj/my-private-repository
  sshPrivateKey: | ![1](img/1.png)
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

![1](img/#co_argo_cd_CO15-1)

设置 SSH 私钥的内容

如果使用 GitHub App 方法，您需要设置 App ID、App Installation ID 和私钥：

```
argocd repo add https://github.com/argoproj/argocd-example-apps.git --github-app-id 1 --github-app-installation-id 2 --github-app-private-key-path test.private-key.pem
```

或者使用声明性方法：

```
apiVersion: v1
kind: Secret
metadata:
  name: github-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  repo: https://ghe.example.com/argoproj/my-private-repository
  githubAppID: 1 ![1](img/1.png)
  githubAppInstallationID: 2
  githubAppEnterpriseBaseUrl: https://ghe.example.com/api/v3 ![2](img/2.png)
  githubAppPrivateKeySecret: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

![1](img/#co_argo_cd_CO16-1)

设置 GitHub App 参数

![2](img/#co_argo_cd_CO16-2)

仅当使用 GitHub App Enterprise 时有效

对于访问令牌，请使用账户名作为用户名，并在密码字段中使用令牌。

选择使用哪种策略将取决于您管理 Kubernetes Secrets 的经验。请记住，Kubernetes 中的 Secret 并不是加密的，而是以 Base64 编码，因此默认情况下不安全。

我们建议只在有了保护机密的良好策略时才使用声明性方法。

###### 注意

我们还没有讨论过 Sealed Secrets 项目（我们将在下一章中讨论），但是在使用 Sealed Secrets 时，标签将被移除，以避免 `SealedSecret` 对象具有一个 `template` 部分，该部分编码了您希望控制器放入未密封 Secret 中的所有字段：

```
spec:
  ...
  template:
    metadata:
      labels:
        "argocd.argoproj.io/secret-type": repository
```

# 7.7 订单 Kubernetes 清单

## 问题

您想要使用 Argo CD 进行部署。

## 解决方案

使用 *同步波* 和 *资源钩子* 修改应用清单的默认顺序。

Argo CD 使用以下逻辑按特定顺序应用 Kubernetes 清单（普通、Helm、Kustomize）：

1.  By kind

    1.  Namespaces

    1.  NetworkPolicy

    1.  Limit Range

    1.  ServiceAccount

    1.  Secret

    1.  ConfigMap

    1.  StorageClass

    1.  PersistentVolumes

    1.  ClusterRole

    1.  Role

    1.  Service

    1.  DaemonSet

    1.  Pod

    1.  ReplicaSet

    1.  Deployment

    1.  StatefulSet

    1.  Job

    1.  Ingress

1.  以相同类型，然后按名称（按字母顺序）

当应用资源时，Argo CD 有三个阶段：第一阶段在应用清单（`PreSync`）之前执行，第二阶段是应用清单（`Sync`），第三阶段在所有清单应用并同步后执行（`PostSync`）。

图 7-13 总结了这些阶段。

![阶段和同步波之间的关系](img/gocb_0713.png)

###### 图 7-13\. 钩子和同步波

资源钩子是在给定阶段执行的脚本，或者如果同步阶段失败，则可以运行一些回滚操作。

表 7-1 列出了可用的资源钩子。

表 7-1\. 资源钩子

| 钩子 | 描述 | 使用案例 |
| --- | --- | --- |

|

`PreSync`

| 在清单应用之前执行 | 数据库迁移 |
| --- | --- |

|

`Sync`

| 与清单同时执行 | 复杂的滚动更新策略，如金丝雀发布或暗部署 |
| --- | --- |

|

`PostSync`

| 在所有 `Sync` 钩子完成并成功（健康）后执行 | 运行测试以验证部署是否正确完成 |
| --- | --- |

|

`SyncFail`

| 在同步操作失败时执行 | 在失败情况下执行回滚操作 |
| --- | --- |

|

`Skip`

| 跳过清单的应用 | 当需要手动步骤来部署应用（例如，释放公共流量到新版本）时 |
| --- | --- |

钩子被定义为一个名为 `argocd.argoproj.io/hook` 的注释，应用于 Kubernetes 资源。在以下片段中，定义了一个 `PostSync` 清单：

```
apiVersion: batch/v1
kind: Job
metadata:
  name: todo-insert ![1](img/1.png)
  annotations:
    argocd.argoproj.io/hook: PostSync ![2](img/2.png)
```

![1](img/#co_argo_cd_CO17-1)

作业名称

![2](img/#co_argo_cd_CO17-2)

设置清单应用的时间点

*同步波*是 Argo CD 应用存储在 Git 中的清单的一种排序方式。

默认情况下，所有清单的波次都为零，较低的值先执行。使用 `argocd.argoproj.io/sync-wave` 注释将波次号设置为资源。

例如，您可能希望先部署数据库，然后再创建数据库架构；对于这种情况，您应该在数据库部署文件中将 `sync-wave` 设置为比创建数据库架构作业中更低，如下片段所示：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgresql ![1](img/1.png)
  namespace: todo
  annotations:
    argocd.argoproj.io/sync-wave: "0" ![2](img/2.png)
...
apiVersion: batch/v1
kind: Job
metadata:
  name: todo-table ![3](img/3.png)
  namespace: todo
  annotations:
    argocd.argoproj.io/sync-wave: "1" ![4](img/4.png)
```

![1](img/#co_argo_cd_CO18-1)

PostgreSQL 部署

![2](img/#co_argo_cd_CO18-2)

PostgreSQL 部署的同步波是 0

![3](img/#co_argo_cd_CO18-3)

作业名称

![4](img/#co_argo_cd_CO18-4)

当 PostgreSQL 健康时执行的作业

## 讨论

当 Argo CD 开始应用清单时，它按以下方式排序资源：

1.  阶段

1.  波次（优先级低的先）

1.  种类

1.  名称

让我们部署一个更大的应用，其中包括部署文件、同步波和钩子。

部署的示例应用是一个 TODO 应用程序，连接到一个数据库（PostgreSQL）来存储 TODO。为了部署该应用程序，需要按特定顺序进行操作；例如，必须先运行数据库服务器，然后再创建数据库架构。此外，当整个应用程序部署完成后，我们会向数据库中插入一些默认的 TODO 来运行后同步清单。

整个过程显示在 图 7-14 中。

![带有执行顺序清单的 TODO 应用](img/gocb_0714.png)

###### 图 7-14\. Todo 应用

创建一个指向应用程序的 `Application` 资源：

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-app
  namespace: argocd
spec:
  destination:
    namespace: todo
    server: https://kubernetes.default.svc
  project: default
  source:
    path: ch07/todo
    repoURL: https://github.com/gitops-cookbook/gitops-cookbook-sc.git
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: false
    syncOptions:
    - CreateNamespace=true
```

在终端中应用资源，Argo CD 将按指定顺序部署所有应用程序。

## 参见

+   [*GitHub 上的 gitops-engine/sync_tasks.go*](https://oreil.ly/NDWru)

# 7.8 定义同步窗口

## 问题

您希望 Argo CD 根据时间阻塞或允许应用程序同步。

## 解决方案

Argo CD 提供了 *同步窗口* 的概念，用于配置应用程序同步的时间窗口（将推送到存储库的新资源应用）是否被阻止或允许。

要定义同步窗口，创建一个 `AppProject` 清单，设置 `kind`（允许或拒绝），以 cron 格式设置初始时间 `schedule`，窗口的 `duration`，以及应用同步窗口的资源（`Application`，命名空间或集群）。

`AppProject` 资源负责定义这些窗口，其中允许/阻止同步。

创建一个新文件，仅允许在 22:00 到 23:00 进行同步（仅一个小时），并且用于以 `-prod` 结尾的 Argo CD `应用程序`：

```
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
spec:
  syncWindows: ![1](img/1.png)
  - kind: allow  ![2](img/2.png)
    schedule: '0 22 * * *' ![3](img/3.png)
    duration: 1h ![4](img/4.png)
    applications: ![5](img/5.png)
    - '*-prod' ![6](img/6.png)
```

![1](img/#co_argo_cd_CO19-1)

窗口列表

![2](img/#co_argo_cd_CO19-2)

允许同步

![3](img/#co_argo_cd_CO19-3)

仅在 22:00

![4](img/#co_argo_cd_CO19-4)

1 小时（23:00）

![5](img/#co_argo_cd_CO19-5)

设置影响此窗口的应用程序

![6](img/#co_argo_cd_CO19-6)

正则表达式匹配以 `-prod` 结尾的任何应用程序

## 讨论

当未在 `AppProject` 清单中定义的时间窗口内配置的时间时，我们无法执行应用程序的同步（无论是自动还是手动）。但是，我们可以配置一个窗口来允许手动同步。

使用 CLI 工具：

```
argocd proj windows enable-manual-sync <PROJECT ID>
```

同时，在 YAML 文件中可以设置手动同步。在下面的示例中，我们设置了 `default` 命名空间的手动同步，拒绝在 22:00 进行一个小时的同步，并允许在 `prod-cluster` 中在 23:00 进行一个小时的同步：

```
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: default
  namespace: argocd
spec:
  syncWindows:
  - kind: deny ![1](img/1.png)
    schedule: '0 22 * * *'
    duration: 1h
    manualSync: true ![2](img/2.png)
    namespaces: ![3](img/3.png)
    - bgd
  - kind: allow
    schedule: '0 23 * * *'
    duration: 1h
    clusters: ![4](img/4.png)
    - prod-cluster
```

![1](img/#co_argo_cd_CO20-1)

阻止同步

![2](img/#co_argo_cd_CO20-2)

启用到 `default` 命名空间的手动同步

![3](img/#co_argo_cd_CO20-3)

配置要阻止的命名空间

![4](img/#co_argo_cd_CO20-4)

配置集群，允许在 23:00 进行同步

我们可以通过转到 设置 → 项目 → 默认 → 窗口选项卡 或者使用 `argocd` CLI 工具来检查当前窗口的 UI：

```
argocd proj windows list default

ID  STATUS    KIND   SCHEDULE    DURATION  APPLICATIONS  NAMESPACES  CLUSTERS      MANUALSYNC
0   Inactive  deny   0 22 * * *  1h        -             bgd         -             Enabled
1   Inactive  allow  0 23 * * *  1h        -             -           prod-cluster  Disabled
```
