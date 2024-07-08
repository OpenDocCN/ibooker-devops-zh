# 第十一章：你的集群的政策与治理

你是否曾想过如何确保集群上运行的所有容器仅来自已批准的容器注册表？或者，也许安全团队要求你强制执行一项政策，即服务永远不暴露在互联网上。这些正是集群政策与治理旨在解决的挑战。随着 Kubernetes 的成熟和越来越多企业的采用，如何将政策与治理应用于 Kubernetes 资源的问题日益频繁。在本章中，我们分享了你可以采取的方法和工具，以确保你的集群符合定义的政策，无论你是在初创企业还是大企业工作。

# 为什么政策与治理至关重要

无论你是在高度受管制的环境中操作（例如医疗或金融服务），还是仅仅想确保你对集群上运行的内容保持控制，你都需要一种实施公司特定政策的方法。一旦定义了你的政策，你就需要确定如何实施它，并维护符合这些政策的集群。这些政策可能需要符合法规合规性，或者仅仅是强制执行最佳实践。无论原因如何，你必须确保在实施这些政策时不牺牲开发者的灵活性和自助服务。

# 这项政策有何不同？

在 Kubernetes 中，政策无处不在。无论是网络策略还是 Pod 安全，我们都明白何时以及如何使用政策。我们相信 Kubernetes 资源规范中声明的任何内容都会按照政策定义进行实施。网络策略和 Pod 安全都是在运行时实施的。然而，政策限制 Kubernetes 资源规范中的字段值。这是政策与治理的工作。与在运行时实施政策不同，当我们在治理的背景下谈论政策时，我们的意思（或者至少我们试图达到的目标）是限制在 Kubernetes 资源中配置字段的方式。只有在通过政策评估时符合的 Kubernetes 资源规范才允许提交到集群状态。

# 云原生政策引擎

为了能够评估哪些资源符合规定，我们需要一个灵活到可以满足各种需求的策略引擎。[开放策略代理 (OPA)](https://oreil.ly/xzN2p) 是一个开源、灵活、轻量级的策略引擎，在云原生生态系统中越来越受欢迎。在生态系统中引入 OPA 后，出现了许多不同的 Kubernetes 管理工具实现。社区正在支持的一个这样的 Kubernetes 策略和治理项目称为 [Gatekeeper](https://oreil.ly/RvKUw)。在本章的其余部分，我们使用 Gatekeeper 作为规范示例，展示如何为您的集群实现策略和治理。尽管生态系统中还有其他策略和治理工具的实现，它们都致力于通过允许只提交符合 Kubernetes 资源规范的资源来提供相同的用户体验（UX）。

# Gatekeeper 简介

Gatekeeper 是一个开源的、可定制的 Kubernetes 准入 webhook，用于集群策略和治理。Gatekeeper 利用 OPA 约束框架来强制执行基于自定义资源定义（CRD）的策略。使用 CRD 允许集成的 Kubernetes 体验，将策略编写与实现分离。策略模板称为 *约束模板*，可以在集群间共享和重用。Gatekeeper 支持资源验证和审计功能。Gatekeeper 的一个很大的优点是它的可移植性，这意味着您可以将其实现在任何 Kubernetes 集群上，如果您已经使用 OPA，可能可以将该策略迁移到 Gatekeeper 上。

###### 注意

Gatekeeper 是一个成熟的开源项目。请访问官方的[上游存储库](https://oreil.ly/Rk8dc)获取最新稳定版本。

## 示例策略

在深入了解如何配置 Gatekeeper 之前，保持解决问题的关注至关重要。虽然每个组织/团队都需要根据其需求优化其策略，但一些普遍适用的策略可作为最佳实践。让我们看一些解决常见合规问题的策略作为背景：

+   服务不得在互联网上公开。

+   仅允许来自受信任的容器注册表的容器。

+   所有容器必须设置资源限制。

+   Ingress 主机名不得重叠。

+   Ingress 必须仅使用 HTTPS。

## Gatekeeper 术语

Gatekeeper 采用了与 OPA 相同的大部分术语。了解这些术语对您理解 Gatekeeper 如何运作至关重要。Gatekeeper 使用 OPA 约束框架，引入了三个新术语：

+   约束

+   Rego

+   约束模板

### 约束

最好的理解约束的方式是将其视为应用于 Kubernetes 资源规范特定字段和值的限制。这实际上只是在说策略的长远方式。当定义约束时，您实际上是在声明您*不希望*允许这样做。这种方法的含义是资源在没有发出拒绝的约束时会被隐式地允许。这是一个重要的细微差别，因为它不是允许您希望的 Kubernetes 资源规范字段和值，而是只拒绝您*不希望*的。这种架构决策非常适合 Kubernetes 资源规范，因为它们是不断变化的。

### Rego

Rego 是一种 OPA 本机查询语言。 Rego 查询是对 OPA 存储的数据的断言。 Gatekeeper 在约束模板中存储 rego。

### 约束模板

将其视为一个策略模板。它是可移植和可重用的。约束模板由类型化参数和用于重用的目标 rego 组成。

## 定义约束模板

约束模板是一个[自定义资源定义](https://oreil.ly/LQSAH)（CRD），提供了模板化策略的手段，以便进行共享或重用。此外，可以验证策略的参数。让我们在上文示例的背景下看一个约束模板，来自上游[Gatekeeper 策略库](https://oreil.ly/HksnE)。在以下示例中，我们分享了一个约束模板，提供策略“仅允许来自可信容器注册表的容器”：

```
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
  annotations:
    metadata.gatekeeper.sh/title: "Allowed Repositories"
    metadata.gatekeeper.sh/version: 1.0.0
    description: >-
      Requires container images to begin with a string from the specified list.
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          type: object
          properties:
            repos:
              description: The list of prefixes a container image is allowed to
                have.
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          satisfied := [good | repo = input.parameters.repos[_] ;
            good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("container <%v> has an invalid image repo <%v>,
            allowed repos are %v",
             [container.name, container.image, input.parameters.repos])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          satisfied := [good | repo = input.parameters.repos[_] ;
            good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("initContainer <%v> has an invalid image repo <%v>,
            allowed repos are %v",
             [container.name, container.image, input.parameters.repos])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.ephemeralContainers[_]
          satisfied := [good | repo = input.parameters.repos[_] ;
            good = startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("ephemeralContainer <%v> has an invalid image repo <%v>,
            allowed repos are %v",
             [container.name, container.image, input.parameters.repos])
        }
```

约束模板由三个主要组成部分组成：

Kubernetes 必需的 CRD 元数据

名称是最重要的部分。最佳做法是使其描述性足够强，以便轻松识别策略的目的。我们稍后引用此名称。

输入参数的模式

根据验证字段指示，此部分定义了输入参数及其关联类型。在本例中，我们有一个名为`repos`的单一参数，它是一个字符串数组。

策略定义

根据`target`字段指示，此部分包含模板化的 rego（在 OPA 中定义策略的语言）。使用约束模板允许重复使用模板化的 rego，这意味着通用策略可以共享。如果规则匹配，则违反了约束。

## 定义约束

要使用先前的约束模板，我们必须创建约束资源。约束资源的目的是为我们之前创建的约束模板提供必要的参数。您可以看到以下示例中定义的资源的`kind`是`K8sAllowedRepos`，它映射到先前部分定义的约束模板：

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: prod-repo-is-openpolicyagent
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "production"
  parameters:
    repos:
      - "openpolicyagent/"
```

约束包含两个主要部分：

Kubernetes 元数据

请注意，此约束属于`kind K8sAllowedRepos`，与约束模板的名称匹配。

规范

`match` 字段定义了策略的意图范围。在此示例中，我们仅匹配生产命名空间中的 Pod。

参数定义策略的意图。请注意，它们与前一节约束模板架构的类型匹配。在本例中，我们只允许以 `openpolicyagent/` 开头的容器镜像。

约束具有以下操作特征：

+   逻辑 AND

    +   当多个策略验证相同字段时，如果一个违反，则整个请求被拒绝

+   允许早期错误检测的模式验证

+   选择标准

    +   可以使用标签选择器

    +   仅约束特定种类

    +   仅在特定命名空间约束

## 数据复制

在某些情况下，您可能希望将当前资源与集群中的其他资源进行比较，例如，“Ingress 主机名不得重叠”的情况。为了评估规则，OPA 需要将所有其他 Ingress 资源缓存到其缓存中。Gatekeeper 使用 `config` 资源来管理在 OPA 缓存中缓存哪些数据，以执行诸如前述的评估。此外，`config` 资源还用于审计功能，稍后我们会详细探讨。

以下示例 `config` 资源缓存 v1 服务、Pod 和命名空间：

```
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
name: config
  namespace: gatekeeper-system
spec:
  sync:
    syncOnly:
    - kind: Service
      version: v1
    - kind: Pod
      version: v1
    - kind: Namespace
      version: v1
```

## UX

Gatekeeper 允许实时反馈给集群用户资源违反了定义的策略。如果我们考虑前面章节的例子，我们只允许从以 `openpolicyagent/` 开头的仓库获取容器。

让我们尝试创建以下资源；根据当前策略，它是不合规的：

```
apiVersion: v1
kind: Pod
metadata:
  name: opa
  namespace: production
spec:
  containers:
    - name: opa
      image: quay.io/opa:0.9.2
```

这会给您定义在约束模板中的违规消息：

```
$ kubectl create -f bad_resources/opa_wrong_repo.yaml
Error from server (Forbidden): error when creating "STDIN": admission webhook
  "validation.gatekeeper.sh" denied the request: [repo-is-openpolicyagent]
    container <opa> has an invalid image repo <quay.io/opa:0.9.2>, allowed
      repos are ["openpolicyagent/"]
```

# 使用执法行动和审计

到目前为止，我们仅讨论了如何定义策略并将其作为请求接受过程的一部分执行。约束包括配置 `enforcementAction` 的能力，默认设置为 `deny`。除了 `deny` 外，`enforcementAction` 还允许接受 `warn` 和 `dryrun` 的值。当我们考虑部署策略时，并不总是情况下你正在应用到一个已有资源的集群或命名空间。因此，了解如何在已部署工作负载的集群上部署策略，并有信心能够识别和修复策略违规，是非常重要的。`enforcementAction` 字段允许您定义其行为。当设置为 `deny` 时，违反策略的资源将不会被创建，并且错误消息将被记录到审计日志并返回给用户。如果设置为 `warn`，资源将会被创建；但是，会记录警告消息到审计日志并返回给用户。最后，如果设置为 `dryrun`，资源将被创建，并且违反策略的资源将在审计日志中可用。

无论您决定使用何种 `enforcementAction`，Gatekeeper 都会定期评估资源，根据任何配置的策略提供审计日志。这有助于检测根据策略配置错误的资源，并允许进行修复。审计结果存储在约束的状态字段中，通过简单使用 `kubectl` 即可找到。要使用审计，必须复制要审计的资源。有关更多详细信息，请参阅 “数据复制”。

让我们看看您在前一节中定义的名为 `prod-repo-is-openpolicyagent` 的约束。在这种情况下，假设我们已经在生产命名空间中运行了一个名为 nginx 的 Pod，并且我们希望使用审计来检查其符合策略：

```
$ kubectl get k8sallowedrepos
NAME                           ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
prod-repo-is-openpolicyagent   deny                 1

$ kubectl get k8sallowedrepos prod-repo-is-openpolicyagent -o yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: ...
  creationTimestamp: "..."
  generation: 1
  name: prod-repo-is-openpolicyagent
  resourceVersion: "..."
  uid: ...
spec:
  match:
    kinds:
    - apiGroups:
      - ""
      kinds:
      - Pod
    namespaces:
    - production
  parameters:
    repos:
    - openpolicyagent/
status:
  auditTimestamp: "2022-11-27T23:37:42Z"
  totalViolations: 1
  violations:
  - enforcementAction: deny
    group: ""
    kind: Pod
    message: container <nginx> has an invalid image repo <nginx>, allowed repos
      are ["openpolicyagent/"]
    name: nginx
    namespace: production
    version: v1
```

检查时，您可以看到审计最后运行的时间在 `auditTimestamp` 字段中。我们还看到所有违反此约束的资源，本例中仅有 nginx Pod，在 `violations` 中，以及 `enforcementAction`。

## 变异

除了资源验证外，Gatekeeper 还允许您配置变异策略。变异策略允许您在准入时修改 Kubernetes 资源。通常情况下，在准入时修改资源并不被认为是最佳实践。Gatekeeper “神奇地”修改资源是云原生反模式，与 Kubernetes 的声明性本质相违背。这里简单提到变异策略，旨在提供指导，以避免除非您的用例绝对需要，并且已经尽力遵循其他最佳实践。有关如何实施 Kubernetes 资源的声明式最佳实践的更多详细信息，请参阅 第十八章。

## 测试策略

随着 GitOps 理念的广泛采纳，将策略和评估作为本地测试或 CI/CD 流水线的一部分已成为必备。Gatekeeper 配备了一个 `gator` CLI，使您能够获取约束模板和约束，并进行本地评估。这是一个很好的工具，用于构建新策略，对资源进行测试，并在部署到生产集群之前解决任何问题。[Gatekeeper 文档](https://oreil.ly/Qj4p8) 提供了使用 `gator` CLI 进行策略测试的实用指南。

## 熟悉 Gatekeeper

如果您希望进一步探索 Gatekeeper，请注意，该存储库附带了出色的演示内容，带您详细了解构建银行合规性政策的示例。我们强烈建议您参与这一演示，以深入了解 Gatekeeper 的操作方式。您可以在 [此 Git 存储库](https://oreil.ly/GcR3i) 中找到演示。Gatekeeper 还维护着一个 [公共库](https://oreil.ly/e8ESD)，其中包含一些政策，您可以通过 [ArtifactHub](https://oreil.ly/uEcfn) 提供的简易安装指南将其应用到您的集群上。

# 政策与治理最佳实践

在实施集群上的策略和治理时，您应考虑以下最佳实践：

+   如果您想强制执行 Pod 中的特定字段，需要确定要检查和强制执行的 Kubernetes 资源规范。例如，让我们考虑 Deployments 的情况。Deployments 管理 ReplicaSets，ReplicaSets 管理 Pods。我们可以在所有三个级别强制执行，但最佳选择是在运行时之前的最低交接点，即 Pod。然而，这个决定有其影响。例如，当我们尝试部署不符合规范的 Pod 时，如在 “UX” 中所见，将不会显示用户友好的错误消息。这是因为用户并未创建不符合规范的资源，而是 ReplicaSet 在创建。这种经历意味着用户需要通过运行 `kubectl describe` 来确定资源是否符合规范，尽管这可能看起来很繁琐，但与其他 Kubernetes 特性（如 Pod 安全性）的行为是一致的。

+   约束可以根据以下标准应用于 Kubernetes 资源：种类、命名空间和标签选择器。我们强烈建议尽可能将约束范围限定在您希望应用的资源上。这样可以确保随着集群资源的增长，政策行为保持一致，并且不需要评估不需要的资源，避免将不需要评估的资源传递给 OPA，这可能导致其他效率低下的情况发生。

+   在已部署资源的集群上，利用 `warn` 和 `dryrun` 结合审核，在将 `enforcementAction` 设置为 `deny` 之前，修复违反策略的资源。

+   不要使用变异策略；而是考虑其他声明性方法，包括 GitOps。

+   不推荐在可能涉及敏感数据的情况下同步和执行，例如 Kubernetes 密钥。考虑到 OPA 可能将其保存在缓存中（如果配置为复制该数据），并且资源将被传递给 Gatekeeper，这会留下潜在的攻击面。

+   如果定义了许多约束，拒绝约束意味着整个请求都将被拒绝。无法使此功能成为逻辑 OR。

# 摘要

在本章中，我们讨论了为什么策略和治理很重要，并介绍了一个基于 OPA 构建的项目，它是一个云原生生态系统策略引擎，提供了一种 Kubernetes 本地方法来处理策略和治理。现在，当安全团队询问“我们的集群是否符合我们定义的策略？”时，您应该已经准备好并自信了。
