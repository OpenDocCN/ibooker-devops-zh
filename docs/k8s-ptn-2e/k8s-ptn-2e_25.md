# 第二十章：配置资源

Kubernetes 提供了用于常规和机密数据的原生配置资源，允许您将配置生命周期与应用程序生命周期解耦。*配置资源* 模式解释了 ConfigMap 和 Secret 资源的概念，以及如何使用它们，以及它们的局限性。

# 问题

“EnvVar Configuration” 模式的一个显著缺点，讨论见 第十九章，是它仅适用于少量变量和简单配置。另一个缺点是，由于环境变量可以在各种地方定义，因此很难找到变量的定义。即使找到了，也不能完全确定它不会在其他位置被覆盖。例如，在 Kubernetes 部署资源中，OCI 镜像内定义的环境变量可能在运行时被替换。

通常，将所有配置数据放在一个地方而不是分散在各种资源定义文件中更好。但是，将整个配置文件的内容放入环境变量中是没有意义的。因此，一些额外的间接方式可以提供更多灵活性，这正是 Kubernetes 配置资源所提供的。

# 解决方案

Kubernetes 提供了专用的配置资源，比纯环境变量更灵活。这些是用于一般用途和敏感数据的 ConfigMap 和 Secret 对象。

我们可以以相同的方式使用两者，因为它们都提供键值对的存储和管理。当我们描述 ConfigMaps 时，大多数时间也可以应用于 Secrets。除了实际数据编码（Secrets 的 Base64 编码外），在使用 ConfigMaps 和 Secrets 时没有技术上的区别。

一旦创建了 ConfigMap 并存储了数据，我们可以以两种方式使用 ConfigMap 的键：

+   作为 *环境变量* 的引用，其中键是环境变量的名称。

+   作为映射到 Pod 中挂载卷的 *文件*。键用作文件名。

当通过 Kubernetes API 更新 ConfigMap 时，挂载的 ConfigMap 卷中的文件也会更新。因此，如果应用程序支持配置文件的热重载，它可以立即从此类更新中受益。但是，使用 ConfigMap 条目作为环境变量时，更新不会反映出来，因为环境变量在进程启动后无法更改。

除了 ConfigMap 和 Secret，另一种选择是直接将配置存储在外部卷中，然后挂载。

以下示例集中于 ConfigMap 的使用，但它们也可以用于 Secrets。但是有一个重要区别：Secrets 的值必须进行 Base64 编码。

ConfigMap 资源在其 `data` 部分包含键值对，如 示例 20-1 所示。

##### 示例 20-1\. ConfigMap 资源

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: random-generator-config
data:
  PATTERN: Configuration Resource  ![1](img/1.png)
  application.properties: |
    # Random Generator config
    log.file=/tmp/generator.log
    server.port=7070
  EXTRA_OPTIONS: "high-secure,native"
  SEED: "432576345"
```

![1](img/#co_configuration_resource_CO1-1)

ConfigMaps 可以作为环境变量和挂载文件访问。我们建议在 ConfigMap 中使用大写键来表示 EnvVar 的用法，并在用作挂载文件时使用适当的文件名。

我们在这里看到，ConfigMap 还可以承载完整配置文件的内容，例如本示例中的 Spring Boot `application.properties`。您可以想象，对于非平凡的用例，此部分可能会变得相当大！

不必手动创建完整的资源描述符，我们也可以使用`kubectl`创建 ConfigMaps 或 Secrets。对于前述示例，等效的`kubectl`命令如示例 20-2 所示。

##### 示例 20-2\. 从文件创建 ConfigMap

```
kubectl create cm spring-boot-config \
   --from-literal=PATTERN="Configuration Resource" \
   --from-literal=EXTRA_OPTIONS="high-secure,native" \
   --from-literal=SEED="432576345" \
   --from-file=application.properties
```

然后可以在各处读取此 ConfigMap——在定义环境变量的每个地方，正如示例 20-3 所示。

##### 示例 20-3\. 从 ConfigMap 设置的环境变量

```
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
  - env:
    - name: PATTERN
      valueFrom:
        configMapKeyRef:
          name: random-generator-config
          key: PATTERN
....
```

如果一个 ConfigMap 有许多条目需要作为环境变量使用，使用特定的语法可以节省大量的输入。与单独指定每个条目不同，在`env`部分的前述示例中，`envFrom`允许您公开所有具有也可以用作有效环境变量的键的 ConfigMap 条目。我们可以以前缀的形式添加这个，如示例 20-4 所示。任何不能作为环境变量使用的键都会被忽略（例如，`"illeg.al"`）。当指定了具有重复键的多个 ConfigMaps 时，`envFrom`中的最后一个条目优先。而且，直接使用`env`设置的任何同名环境变量具有更高的优先级。

##### 示例 20-4\. 将 ConfigMap 的所有条目设置为环境变量

```
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
    envFrom:                          ![1](img/1.png)
    - configMapRef:
        name: random-generator-config
      prefix: CONFIG_                 ![2](img/2.png)
```

![1](img/#co_configuration_resource_CO2-1)

检索可以用作环境变量名称的 ConfigMap `random-generator-config` 的所有键。

![2](img/#co_configuration_resource_CO2-2)

使用`CONFIG_`前缀所有适合的 ConfigMap 键。通过在示例 20-1 中定义的 ConfigMap，这将导致三个公开的环境变量：`CONFIG_​PAT⁠TERN_NAME`、`CONFIG_EXTRA_OPTIONS`和`CONFIG_SEED`。

Secrets，与 ConfigMaps 一样，也可以作为环境变量使用，无论是每个条目还是所有条目。要访问 Secret 而不是 ConfigMap，请用`secretKeyRef`替换`configMapKeyRef`。

当将 ConfigMap 用作卷时，它的完整内容将投射到此卷中，其中键用作文件名。参见示例 20-5。

##### 示例 20-5\. 将 ConfigMap 挂载为卷

```
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    volumeMounts:
    - name: config-volume
      mountPath: /config
  volumes:
  - name: config-volume
    configMap:  ![1](img/1.png)
      name: random-generator-config
```

![1](img/#co_configuration_resource_CO3-1)

一个由 ConfigMap 支持的卷将包含与条目数量相同的文件，其中映射的键作为文件名，映射的值作为文件内容。

作为卷挂载的示例例子 20-1 在 */config* 文件夹中生成四个文件：一个 *application.properties* 文件，其中包含在 ConfigMap 中定义的内容，以及 *PATTERN*、*EXTRA_OPTIONS* 和 *SEED* 文件，每个文件都有单行内容。

配置数据的映射可以通过向卷声明添加额外属性来更加精细地调整。与其将所有条目都映射为文件，您还可以单独选择应该暴露的每个键、文件名以及可用的权限。示例 20-6 展示了如何精细选择 ConfigMap 的哪些部分作为卷暴露。

##### 示例 20-6\. 有选择地将 ConfigMap 条目作为卷暴露

```
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    volumeMounts:
    - name: config-volume
      mountPath: /config
  volumes:
  - name: config-volume
    configMap:
      name: random-generator-config
      items:                          ![1](img/1.png)
      - key: application.properties   ![2](img/2.png)
        path: spring/myapp.properties
        mode: 0400
```

![1](img/#co_configuration_resource_CO4-1)

要暴露为卷的 ConfigMap 条目列表。

![2](img/#co_configuration_resource_CO4-2)

仅将 `application.properties` 从 ConfigMap 暴露在路径 `spring/myapp.properties` 下，文件模式为 0400。

如您所见，对 ConfigMap 的更改会直接反映在作为文件包含 ConfigMap 内容的投影卷中。应用程序可以监视这些文件并立即获取任何更改。这种热重载非常有用，可以避免应用程序的重新部署，从而避免服务中断。另一方面，这些实时更改并未被任何地方跟踪，容易在重新启动期间丢失。这些临时更改可能导致配置漂移，难以检测和分析。这就是为什么许多人更喜欢一种*不可变配置*，一旦部署后就保持不变。我们在第二十一章，“不可变配置”中专门介绍了这一范式，但使用 ConfigMap 和 Secrets 也有一种简单的方法可以轻松实现这一点。

自版本 1.21 起，Kubernetes 支持 ConfigMaps 和 Secrets 的 `immutable` 字段，如果设置为 `true`，则阻止资源在创建后更新。除了防止不必要的更新外，使用不可变的 ConfigMaps 和 Secrets 还显著提升了集群的性能，因为 Kubernetes API 服务器不需要监视这些不可变对象的更改。示例 20-7 展示了如何声明一个不可变的 Secret。一旦在集群上存储了这样一个 Secret，唯一更改它的方法是删除并重新创建更新后的 Secret。任何引用此 Secret 的运行中 Pod 也需要重新启动。

##### 示例 20-7\. 不可变的 Secret

```
apiVersion: v1
kind: Secret
metadata:
  name: random-config
data:
  user: cm9sYW5k
immutable: true  ![1](img/1.png)
```

![1](img/#co_configuration_resource_CO5-1)

声明 Secret 的可变性的布尔标志（默认为 `false`）。

# 讨论

ConfigMaps 和 Secrets 允许您将配置信息存储在专用资源对象中，通过 Kubernetes API 很容易管理。使用 ConfigMaps 和 Secrets 的最大优势是，它们将配置数据的*定义*与其*使用*分离。这种解耦允许我们独立管理使用配置的对象，而不依赖于配置的定义。ConfigMaps 和 Secrets 的另一个好处是它们是平台的固有特性。不像需要类似 第二十一章，“不可变配置” 中的自定义构造。

然而，这些配置资源也有它们的限制：Secrets 的大小限制为 1 MB，无法存储任意大的数据，也不适合非配置应用数据。您也可以在 Secrets 中存储二进制数据，但由于必须进行 Base64 编码，只能使用约 700 KB 的数据。现实世界的 Kubernetes 集群还会对每个命名空间或项目可以使用的 ConfigMap 数量设置单独的配额，因此 ConfigMap 不是万能的解决方案。

接下来的两章介绍如何通过使用*不可变配置*和*配置模板*模式来处理大型配置数据。

# 更多信息

+   [配置资源示例](https://oreil.ly/-_jDa)

+   [配置 Pod 使用 ConfigMap](https://oreil.ly/oRN9a)

+   [秘密](https://oreil.ly/mvoXO)

+   [加密静态数据](https://oreil.ly/GrL0_)

+   [安全分发凭据](https://oreil.ly/Im-R9)

+   [不可变 Secrets](https://oreil.ly/9PvQ5)

+   [如何创建不可变的 ConfigMaps 和 Secrets](https://oreil.ly/ndYd0)

+   [ConfigMap 的大小限制](https://oreil.ly/JUDZU)
