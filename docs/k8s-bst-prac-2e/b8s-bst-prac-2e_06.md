# 第六章：版本控制、发布和部署

传统的单体应用程序的主要问题之一是随着时间推移，它们开始变得过于庞大和笨拙，无法按照业务需要的速度进行正确升级、版本控制或修改。许多人认为，这是导致更敏捷开发实践和微服务架构出现的关键因素之一。能够快速迭代新代码、解决新问题或在它们成为主要问题之前修复隐藏问题，以及实现零停机升级的承诺，都是开发团队努力实现的目标。实际上，通过适当的流程和程序可以解决这些问题，无论是什么类型的系统，但这通常会以更高的技术和人力成本来维护。

在设计系统时，隔离性和组合性是重要的变量。采用容器作为应用代码的运行时可以实现这一点，但仍然需要高水平的人工自动化或系统管理来保持大型系统的可靠水平。随着时间的推移，系统变得更加脆弱，引入了更多脆弱性，并且系统工程师开始构建复杂的自动化流程，以实现复杂的发布、升级和故障检测机制。诸如 Apache Mesos、HashiCorp Nomad 以及甚至专门的基于容器的编排器，如 Kubernetes 和 Docker Swarm，已经将这些流程进化为更原始的组件直接集成到它们的运行时中。现在，系统工程师可以解决更复杂的系统问题，因为基础已经提升到包括将应用程序版本控制、发布和部署到系统中。

# 版本控制

本节并非关于软件版本控制及其历史的入门介绍；关于这个主题有无数的文章和计算机科学课程书籍可供参考。最重要的是选择一种模式并坚持下去。大多数软件公司和开发者已经达成共识，某种形式的*语义化版本控制*是最有用的，特别是在微服务架构中，写作某个微服务的团队会依赖于构成系统的其他微服务的 API 兼容性。

对于那些不熟悉语义化版本控制的人来说，基本的概念是，它遵循一个由*主版本*、*次版本*和*修订版本*组成的三部分版本号，通常以点号表示，比如 1(主).2(次).3(修订)。修订版本表示包含了修复 bug 或者没有 API 变更的非常小的改动。次版本表示有可能有新的 API 变更，但与之前的版本兼容。对于与其他微服务进行交互但未参与开发的开发者来说，这是一个关键属性。知道我将我的服务编写成与另一个微服务的 1.4.7 版本进行通信，而后者最近升级到了 1.5.7，就意味着除非我想利用任何新的 API 特性，否则我可能不需要更改我的代码。主版本是代码的重大变更增量。在大多数情况下，同一代码的主版本之间的 API 不再兼容。此过程有许多细微的修改，包括使用“4”版本来指示软件在其开发生命周期中的阶段，例如 alpha 代码的 1.4.7.0 版本和发布的 1.4.7.3 版本。最重要的是，系统内部保持一致性。

# 发布版本

事实上，Kubernetes 实际上没有发布控制器，因此没有发布的本地概念。这通常添加到部署的`metadata.labels`规范中和/或`pod.spec.template.metadata.label`规范中。何时包含其中一个非常重要，并且基于 CD 用于更新部署变更的方式，它可能会产生不同的影响。当引入用于 Kubernetes 的 Helm 时，其主要概念之一是引入了发布的概念，以区分集群中相同 Helm 图表的运行实例。这个概念可以很容易地在没有 Helm 的情况下复制，但是 Helm 本身可以跟踪发布及其历史记录，因此许多 CD 工具将 Helm 集成到其流水线中，作为实际的发布服务。再次强调的关键是在集群系统状态中使用版本控制的一致性。

如果对某些名称的定义达成机构一致意见，发布名称可能非常有用。通常会使用诸如`stable`或`canary`之类的标签，这有助于在添加诸如服务网格等工具以进行细粒度路由决策时进行一些操作控制。驱动多个面向不同受众的变更的大型组织也会采用环形架构，可以表示为`ring-0`、`ring-1`等。

这个主题需要稍微深入了解 Kubernetes 声明性模型中标签的具体细节。标签本身非常自由形式，可以是任何遵循 API 语法规则的键/值对。关键不是内容本身，而是每个控制器如何处理标签，标签的更改以及标签的选择器匹配。Jobs、Deployments、ReplicaSets 和 DaemonSets 支持通过直接映射或基于集合的表达式来基于标签对 pod 进行选择器匹配。重要的是要理解，在创建后标签选择器是不可变的，这意味着如果添加了新的选择器，并且 pod 的标签有相应的匹配，会创建一个新的 ReplicaSet，而不是对现有 ReplicaSet 进行升级。在处理讨论的扩展时，理解这一点非常重要。

# Rollouts

在 Kubernetes 引入 Deployment 控制器之前，控制应用程序如何由 Kubernetes 控制器进程进行滚动更新的唯一机制是在特定要更新的 `replicaController` 上使用命令行界面（CLI）命令 `kubectl rolling-update`。这对于声明式 CD 模型来说非常困难，因为这不是原始清单状态的一部分。必须确保清单正确更新，适当版本化，以防意外回滚系统，并在不再需要时进行存档。Deployment 控制器增加了使用特定策略自动化此更新过程的能力，然后允许系统根据 Deployment 的 `spec.template` 的更改来读取声明性的新状态。这一点经常被 Kubernetes 的新用户误解，当他们更改 Deployment 元数据字段中的标签，重新应用清单时，并未触发任何更新，从而导致 frustration。Deployment 控制器能够确定规范的更改，并将根据规范定义的策略采取行动来更新 Deployment。Kubernetes Deployments 支持两种策略，`rollingUpdate` 和 `recreate`，前者是默认值。

如果指定了滚动更新，Deployment 将创建一个新的 ReplicaSet 来扩展到所需的副本数量，并且旧的 ReplicaSet 将根据 `maxUnavailble` 和 `maxSurge` 的特定值缩减到零。实质上，这两个值将防止 Kubernetes 删除旧的 pod，直到足够数量的新 pod 上线，并且 Kubernetes 不会创建新的 pod，直到一定数量的旧 pod 被删除。好处在于，Deployment 控制器将保留更新的历史记录，并且通过 CLI，可以将 Deployment 回滚到先前的版本。

`recreate` 策略是对某些工作负载有效的策略，能够处理 ReplicaSet 中的 Pod 完全宕机而无需降级服务。在这种策略中，部署控制器将使用新的配置创建一个新的 ReplicaSet，并在将新的 Pod 上线之前删除先前的 ReplicaSet。在排队系统后面的服务就是可以处理此类中断的示例，因为消息将在等待新的 Pod 上线时排队，一旦新的 Pod 上线，消息处理将恢复。

# 将所有内容整合在一起

在单个服务的部署中，版本控制、发布和推出管理影响几个关键领域。让我们来看一个部署的例子，然后分析与最佳实践相关的特定关注领域：

```
# Web Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gb-web-deploy
  labels:
    app: guest-book
    appver: 1.6.9
    environment: production
    release: guest-book-stable
    release number: 34e57f01
spec:
  strategy:
    type: rollingUpdate
    rollingUpdate:
      maxUnavailbale: 3
      maxSurge: 2
  selector:
    matchLabels:
      app: gb-web
      ver: 1.5.8
    matchExpressions:
      - {key: environment, operator: In, values: [production]}
  template:
    metadata:
      labels:
        app: gb-web
        ver: 1.5.8
        environment: production
    spec:
      containers:
      - name: gb-web-cont
        image: evillgenius/gb-web:v1.5.5
        env:
        - name: GB_DB_HOST
          value: gb-mysql
        - name: GB_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
---
# DB Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gb-mysql
  labels:
    app: guest-book
    appver: 1.6.9
    environment: production
    release: guest-book-stable
    release number: 34e57f01
spec:
  selector:
    matchLabels:
      app: gb-db
      tier: backend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: gb-db
        tier: backend
        ver: 1.5.9
        environment: production
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim
---
# DB Backup Job
apiVersion: batch/v1
kind: Job
metadata:
  name: db-backup
  labels:
    app: guest-book
    appver: 1.6.9
    environment: production
    release: guest-book-stable
    release number: 34e57f01
  annotations:
    "helm.sh/hook": pre-upgrade
    "helm.sh/hook": pre-delete
    "helm.sh/hook": pre-rollback
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    metadata:
      labels:
        app: gb-db-backup
        tier: backend
        ver: 1.6.1
        environment: production
    spec:
      containers:
      - name: mysqldump
        image: evillgenius/mysqldump:v1
        env:
        - name: DB_NAME
          value: gbdb1
        - name: GB_DB_HOST
          value: gb-mysql
        - name: GB_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        volumeMounts:
          - mountPath: /mysqldump
            name: mysqldump
      volumes:
        - name: mysqldump
          hostPath:
            path: /home/bck/mysqldump
      restartPolicy: Never
  backoffLimit: 3
```

初次检查时，可能会感觉有些奇怪。一个部署如何具有版本标签，而部署使用的容器镜像却具有不同的版本标签？如果一个更改而另一个不更改会发生什么？在这个例子中发布意味着什么，如果更改会对系统有什么影响？如果更改了某个标签，何时会触发对我的部署的更新？我们可以通过查看版本控制、发布和推出的一些最佳实践来找到这些问题的答案。

# 版本控制、发布和推出的最佳实践

有效的 CI/CD 和提供减少或零停机时间的部署能力取决于使用一致的版本控制和发布管理实践。以下最佳实践可以帮助定义一致的参数，以协助 DevOps 团队提供平稳的软件部署：

+   对应用程序使用语义化版本控制，其版本与组成整个应用程序的容器和 Pod 的版本不同。这允许容器具有独立的生命周期，而组成应用程序的容器和整个应用程序可以有不同的版本。起初可能会有些混乱，但如果采用原则性的分层方法来处理一个变化另一个的情况，您可以轻松跟踪它。在前面的例子中，容器本身目前使用的是 `v1.5.5`，然而 Pod 的规范是 `1.5.8`，这可能意味着对 Pod 规范进行了更改，比如新的 ConfigMaps、额外的 Secrets 或更新的副本值，但所使用的具体容器版本并没有更改。整个应用程序，包括其所有服务的整个应用程序客户端，目前处于 `1.6.9`，这可能意味着运维团队在这一过程中进行了超出这个具体服务之外的更改，比如组成整个应用程序的其他服务。

+   在部署元数据中使用一个发布和发布版本/编号标签来跟踪从 CI/CD 管道中的发布。发布名称和发布编号应与 CI/CD 工具记录中的实际发布协调。这样可以通过 CI/CD 过程追踪到集群，并更容易识别回滚标识。在前面的示例中，发布编号直接来自创建清单的 CD 管道的发布 ID。

+   如果正在使用 Helm 打包服务以部署到 Kubernetes，特别注意将需要一起回滚或升级的服务捆绑到同一个 Helm 图中。Helm 允许轻松回滚应用程序的所有组件，将状态恢复到升级之前的状态。因为 Helm 实际上在传递扁平化的 YAML 配置之前会处理模板和所有 Helm 指令，使用生命周期钩子可以确保特定模板的正确应用顺序。操作员可以使用适当的 Helm 生命周期钩子来确保升级和回滚的正确执行。前面关于 `Job` 规范的示例使用 Helm 生命周期钩子来确保在回滚、升级或删除 Helm 发布之前运行数据库备份。它还确保在作业成功运行后删除 `Job`，在 Kubernetes 的 TTL 控制器退出 alpha 版本之前，这需要手动清理。

+   对于组织的运营节奏，达成一个有意义的发布命名约定是很重要的。对于大多数情况来说，简单的 `stable`、`canary` 和 `alpha` 状态是相当足够的。

# 摘要

Kubernetes 已经允许公司（无论大小）采用更复杂的敏捷开发流程。能够自动化许多通常需要大量人力和技术资本的复杂流程，现在已经被普及，即使是初创公司也能相对容易地利用这种云模式。Kubernetes 的真正声明性质在于在规划正确使用标签和使用本地 Kubernetes 控制器功能时表现得非常出色。通过在部署到 Kubernetes 的应用程序的声明属性中正确识别操作和开发状态，组织可以将工具和自动化联系起来，更轻松地管理升级、发布和回滚能力的复杂流程。
