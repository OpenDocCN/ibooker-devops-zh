# 第十九章：环境变量配置

在这种 *环境变量配置* 模式中，我们探讨了配置应用程序的最简单方法。对于一小组配置值，将它们放入广泛支持的环境变量中是外部化配置的最简单方式。我们将看到在 Kubernetes 中声明环境变量的不同方法，但同时也会看到使用环境变量进行复杂配置的限制。

# 问题

每个非平凡的应用程序都需要一些配置来访问数据源、外部服务或生产级调整。我们早在 [十二要素应用宣言](https://12factor.net) 发布之前就知道，在应用程序中硬编码配置是一件坏事。相反，应将配置 *外部化*，以便在构建应用程序之后甚至可以进行更改。这为容器化应用程序提供了更多价值，促进共享不可变应用程序构件。但在容器化世界中，如何才能做到最好呢？

# 解决方案

十二要素应用宣言建议使用环境变量存储应用程序配置。这种方法简单且适用于任何环境和平台。每个操作系统都知道如何定义环境变量并将其传播给应用程序，每种编程语言也允许轻松访问这些环境变量。可以说环境变量具有普遍适用性。在使用环境变量时，典型的使用模式是在构建时定义硬编码的默认值，然后在运行时进行覆盖。让我们看看在 Docker 和 Kubernetes 中如何实现这一点的具体示例。

对于 Docker 镜像，可以直接在 Dockerfile 中使用 `ENV` 指令定义环境变量。可以逐行定义，也可以一行定义多个，就像 示例 19-1 中展示的那样。

##### 示例 19-1\. 带有环境变量的 Dockerfile 示例

```
FROM openjdk:11
ENV PATTERN "EnvVar Configuration"
ENV LOG_FILE "/tmp/random.log"
ENV SEED "1349093094"

# Alternatively:
ENV PATTERN="EnvVar Configuration" LOG_FILE=/tmp/random.log SEED=1349093094
...
```

那么在此类容器中运行的 Java 应用程序可以通过调用 Java 标准库轻松访问这些变量，如 示例 19-2 所示。

##### 示例 19-2\. 在 Java 中读取环境变量

```
public Random initRandom() {
  long seed = Long.parseLong(System.getenv("SEED"));
  return new Random(seed);      ![1](img/1.png)
}
```

![1](img/#co_envvar_configuration_CO1-1)

使用 EnvVar 初始化随机数生成器的种子。

直接运行这样的镜像将使用默认的硬编码值。但在大多数情况下，您希望从镜像外部覆盖这些参数。

在直接使用 Docker 运行这样的镜像时，可以通过调用 Docker 的命令行来设置环境变量，如 示例 19-3 所示。

##### Example 19-3\. 在启动 Docker 容器时设置环境变量

```
docker run -e PATTERN="EnvVarConfiguration" \
           -e LOG_FILE="/tmp/random.log" \
           -e SEED="147110834325" \
           k8spatterns/random-generator:1.0
```

对于 Kubernetes，这些类型的环境变量可以直接在控制器的 Pod 规范中设置，如部署或副本集中所示（如 示例 19-4 所示）。

##### 示例 19-4\. 配置环境变量的部署

```
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    env:
    - name: LOG_FILE
      value: /tmp/random.log             ![1](img/1.png)
    - name: PATTERN
      valueFrom:
        configMapKeyRef:                 ![2](img/2.png)
          name: random-generator-config  ![3](img/3.png)
          key: pattern                   ![4](img/4.png)
    - name: SEED
      valueFrom:
        secretKeyRef:                    ![5](img/5.png)
          name: random-generator-secret
          key: seed
```

![1](img/#co_envvar_configuration_CO2-1)

使用文字值的 EnvVar。

![2](img/#co_envvar_configuration_CO2-2)

来自 ConfigMap 的 EnvVar。

![3](img/#co_envvar_configuration_CO2-3)

ConfigMap 的名称。

![4](img/#co_envvar_configuration_CO2-4)

ConfigMap 中查找 EnvVar 值的键。

![5](img/#co_envvar_configuration_CO2-5)

从 Secret 中获取的 EnvVar（查找语义与 ConfigMap 相同）。

在这样的 Pod 模板中，您不仅可以直接附加值到环境变量（例如 `LOG_FILE`），还可以委托给 Kubernetes Secrets 和 ConfigMaps。ConfigMap 和 Secret 的间接性的优势在于可以独立于 Pod 定义管理环境变量。在 第二十章，“配置资源” 中详细解释了 Secret 和 ConfigMap 及其优缺点。

在前面的示例中，`SEED` 变量来自 Secret 资源。虽然这是 Secret 的一个完全有效的用法，但也很重要指出环境变量不是安全的。将敏感且可读的信息放入环境变量中使得这些信息易于阅读，甚至可能泄露到日志中。

您还可以通过 `envFrom` 导入特定 Secret 或 ConfigMap 的 *所有* 值，而不是单独引用 Secrets 或 ConfigMaps 的配置值。我们在 第二十章，“配置资源” 中详细解释了此字段，在详细讨论 ConfigMaps 和 Secrets 时会谈到。

可与环境变量一起使用的另外两个有价值的功能是 downward API 和 *dependent variables*。您在 第十四章，“自我意识” 中已经学习了有关 downward API 的全部内容，现在让我们看一下 示例 19-5 中的 dependent variables，这使您可以引用先前定义的变量作为其他条目值定义中的值。

##### 示例 19-5\. Dependent environment variables

```
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    env:
    - name: PORT
      value: "8181"
    - name: IP                        ![1](img/1.png)
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: MY_URL
      value: "https://$(IP):$(PORT)"  ![2](img/2.png)
```

![1](img/#co_envvar_configuration_CO3-1)

使用 downward API 获取 Pod 的 IP。详细讨论 downward API，请参阅 第十四章，“自我意识”。

![2](img/#co_envvar_configuration_CO3-2)

包含先前定义的环境变量 `IP` 和 `PORT` 来构建 URL。

使用 `$(...)` 表示法，您可以引用在 `env` 列表中先前定义的环境变量或来自 `envFrom` 导入的变量。Kubernetes 将在容器启动期间解析这些引用。不过要注意顺序：如果引用列表中稍后定义的变量，它将不会被解析，并且 `$(...)` 引用将直接采用字面量。此外，您还可以在 Pod 命令中使用此语法引用环境变量，如 示例 19-6 所示。

##### 示例 19-6\. 在容器命令定义中使用环境变量

```
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
    - name: random-generator
      image: k8spatterns/random-generator:1.0
      command: [ "java", "RandomRunner", "$(OUTPUT_FILE)", "$(COUNT)" ] ![1](img/1.png)
      env:                       ![2](img/2.png)
        - name: OUTPUT_FILE
          value: "/numbers.txt"
        - name: COUNT
          valueFrom:
            configMapKeyRef:
              name: random-config
              key: RANDOM_COUNT
```

![1](img/#co_envvar_configuration_CO4-1)

启动容器的启动命令的参考环境变量。

![2](img/#co_envvar_configuration_CO4-2)

命令中替换的环境变量的定义。

# 讨论

环境变量易于使用，每个人都知道它们。这个概念与容器无缝映射，并且每个运行时平台都支持环境变量。但是环境变量不安全，仅适用于相当数量的配置值。当需要配置大量不同的参数时，管理所有这些环境变量变得笨拙。

在这些情况下，许多人使用额外的间接级别，并将配置放入各种配置文件中，每个环境一个文件。然后使用单个环境变量选择其中一个文件。*Spring Boot* 的*Profiles*就是这种方法的一个示例。由于这些配置文件通常存储在应用程序自身内部，即在容器内部，这将配置与应用程序紧密耦合。这经常导致开发和生产的配置最终并排存放在同一个 Docker 镜像中，每次更改任何一个环境都需要重新构建镜像。我们不推荐这种设置（配置应始终外部化），但这种解决方案表明环境变量仅适用于小到中等配置集。

在更复杂的配置需求出现时，下一章节描述的*配置资源*、*不可变配置*和*配置模板*是良好的替代方案。

环境变量是通用的，因此我们可以在不同层次设置它们。这种选择导致配置定义的碎片化，并且很难追踪给定环境变量的设置位置。当没有一个集中的地方定义所有环境变量时，调试配置问题就变得困难。

环境变量的另一个缺点是它们只能在应用程序启动*之前*设置，不能稍后更改。一方面，不能在运行时“热”更改配置是一个缺点，以调整应用程序。然而，许多人认为这是一个优点，因为它促进了配置的*不可变性*。这里的不可变性意味着你丢弃正在运行的应用程序容器，并启动一个具有修改配置的新副本，很可能使用滚动更新等平滑的部署策略。这样，你始终处于定义明确和良好已知的配置状态。

环境变量使用简单，但主要适用于简单用例，并且对于复杂配置需求有限制。下面的模式展示了如何克服这些限制。

# 更多信息

+   [环境变量配置示例](https://oreil.ly/W25g0)

+   [十二要素应用](https://oreil.ly/DzBTm)

+   [将 Pod 信息通过环境变量暴露给容器](https://oreil.ly/KxFtr)

+   [定义依赖的环境变量](https://oreil.ly/YoUVj)

+   [Spring Boot 配置文件使用配置值集合](https://oreil.ly/3XVe9)
