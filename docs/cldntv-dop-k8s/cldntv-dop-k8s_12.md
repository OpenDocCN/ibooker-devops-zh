# 第十章：配置和 Secrets

> 如果你想保守一个秘密，你必须也要从自己那里隐藏它。
> 
> George Orwell，*1984*

能够将 Kubernetes 应用程序的逻辑与其配置分离开来非常有用：即可能在应用程序生命周期内发生变化的任何值或设置。配置值通常包括诸如特定于环境的设置、第三方服务的 DNS 地址和身份验证凭据等内容。

虽然您可以直接将这些值放入您的代码中，但这并不是一个非常灵活的方法。首先，更改配置值将需要完全重建和重新部署应用程序。最好是将这些值从代码中分离出来，并从文件或环境变量中读取它们。

Kubernetes 提供了几种不同的方法来帮助您管理配置。一种方法是通过 Pod 规范中的环境变量向应用程序传递值（参见 “环境变量”）。另一种方法是直接将配置数据存储在 Kubernetes 中，使用 ConfigMap 和 Secret 对象。

在本章中，我们将详细探讨 ConfigMaps 和 Secrets，并查看一些管理应用程序中配置和密钥的实用技术，使用演示应用程序作为示例。

# ConfigMaps

ConfigMap 是 Kubernetes 中存储配置数据的主要对象。您可以将其视为一个命名的键值对集合，用于存储配置数据。一旦创建了 ConfigMap，您可以通过在 Pod 中创建文件或将其注入到 Pod 的环境中来为应用程序提供数据。

在本节中，我们将探讨将数据输入到 ConfigMap 中的几种不同方法，然后探索将这些数据提取并馈送到您的 Kubernetes 应用程序中的各种方法。

## 创建 ConfigMaps

假设您想在 Pod 的文件系统中创建一个名为 *config.yaml* 的 YAML 配置文件，并具有以下内容：

```
autoSaveInterval: 60
batchSize: 128
protocols:
  - http
  - https
```

有了这些值集合，您如何将它们转换为可应用于 Kubernetes 的 ConfigMap 资源？

一种方式是将数据指定为 ConfigMap 清单中的 YAML 字面值。这是 ConfigMap 对象清单的样子：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
data:
  config.yaml: |
    autoSaveInterval: 60
    batchSize: 128
    protocols:
      - http
      - https
```

您可以通过从头开始编写清单来创建 ConfigMap，并将 *config.yaml* 中的值添加到 `data` 部分，就像我们在这个例子中所做的那样。

不过，更简单的方法是让 `kubectl` 为您做一些工作。您可以直接从 YAML 文件创建 ConfigMap 如下所示：

```
`kubectl create configmap demo-config --from-file=config.yaml`
configmap "demo-config" created
```

要导出与此 ConfigMap 对应的清单文件，请运行：

```
`kubectl get configmap/demo-config -o yaml ` `>demo-config.yaml`
```

这将写入集群的 ConfigMap 资源的 YAML 清单表示到文件 *demo-config.yaml*，但它将包含额外的信息，如 `status` 部分，您可能希望在再次应用之前将其删除（参见 “导出资源”）。

## 从 ConfigMaps 设置环境变量

现在我们已经在 ConfigMap 对象中拥有了所需的配置数据，那么如何将这些数据传输到容器中呢？让我们看一个使用我们演示应用程序的完整示例。您将在演示存储库的*hello-config-env*目录中找到代码。

这是我们在之前章节中使用的相同演示应用程序，它监听 HTTP 请求并以问候语回复（参见“查看源代码”）。

不过，这次我们不会将字符串`Hello`硬编码到应用程序中，而是希望将问候语设置为可配置。因此，稍微修改了`handler`函数以从环境变量`GREETING`中读取该值：

```
func handler(w http.ResponseWriter, r *http.Request) {
  `greeting` `:=` `os``.``Getenv``(``"GREETING"``)`
  fmt.Fprintf(w, "%s, 世界\n", greeting)
}
```

不用担心 Go 代码的具体细节；这只是一个演示。可以肯定的是，如果在程序运行时存在`GREETING`环境变量，它将在响应请求时使用该值。无论您使用何种语言编写应用程序，都可以使用它来读取环境变量。

现在，让我们创建 ConfigMap 对象以保存问候值。您将在演示存储库的*hello-config-env*目录中找到 ConfigMap 清单文件，以及修改后的 Go 应用程序。

看起来像这样：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
data:
  greeting: Hola
```

为了使这些数据在容器的环境中可见，我们需要稍微修改部署。以下是演示部署的相关部分：

```
spec:
  containers:
    - name: demo
      `image``:` `cloudnatived/demo:hello-config-env`
      ports:
        - containerPort: 8888
      `env``:`
        `-` `name``:` `GREETING`
          `valueFrom``:`
            `configMapKeyRef``:`
              `name``:` `demo-config`
              `key``:` `greeting`
```

注意我们使用了不同的容器镜像标签，与之前的示例不同（参见“镜像标识符”）。`:hello-config-env`标签让我们获取修改后的演示应用程序版本，该版本读取`GREETING`变量：`cloudnatived/demo:hello-config-env`。

其次感兴趣的是`env`部分。从“环境变量”中记得，您可以通过添加`name`/`value`对来创建具有字面值的环境变量。

这里仍然有`name`，但是我们使用了`valueFrom`而不是`value`。这告诉 Kubernetes，它不应该使用变量的字面值，而是应该去其他地方找到该值。

`configMapKeyRef`告诉它引用特定 ConfigMap 中的特定键。要查看的 ConfigMap 的名称是`demo-config`，我们要查找的键是`greeting`。我们使用 ConfigMap 清单创建了这些数据，所以现在应该可以将其读取到容器的环境中。

如果 ConfigMap 不存在，部署将无法运行（其 Pod 将显示`CreateContainerConfigError`状态）。

这就是使更新后的应用程序工作所需的一切，因此继续将清单部署到您的 Kubernetes 集群。从演示存储库目录中运行以下命令：

```
`kubectl apply -f hello-config-env/k8s/`
configmap/demo-config created deployment.apps/demo created
```

与之前一样，要在 Web 浏览器中查看应用程序，您需要将本地端口转发到 Pod 的端口 8888：

```
`kubectl port-forward deploy/demo 9999:8888`
Forwarding from 127.0.0.1:9999 -> 8888 Forwarding from [::1]:9999 -> 8888
```

（这次我们没有费心创建一个 Service；尽管在实际的生产应用程序中您会使用 Service，但在这个示例中，我们只是直接使用 `kubectl` 将本地端口转发到 `demo` Deployment。）

如果您将您的网络浏览器指向 http://localhost:9999/，如果一切正常，您应该能看到：

`Hola, 世界`

## 从 ConfigMap 设置整个环境

虽然您可以从单个 ConfigMap 键设置一两个环境变量，就像我们在前面的示例中看到的那样，但对于大量变量来说，这可能会变得乏味。

幸运的是，有一种简单的方法可以从 ConfigMap 中获取所有键，并将它们转换为环境变量，使用 `envFrom`：

```
spec:
  containers:
    - name: demo
      image: cloudnatived/demo:hello-config-env
      ports:
        - containerPort: 8888
      `envFrom``:`
      `-` `configMapRef``:`
            `name``:` `demo-config`
```

现在，`demo-config` ConfigMap 中的每个设置都将成为容器环境中的变量。因为在我们的示例 ConfigMap 中键称为 `greeting`，所以环境变量也将被命名为 `greeting`（小写）。如果您在使用 `envFrom` 时希望使环境变量名大写，请在 ConfigMap 中进行更改。

您也可以像在我们之前的示例中那样，在清单文件中直接放置文字值或使用 `ConfigMapKeyRef`，正常方式设置容器的其他环境变量。Kubernetes 允许您同时使用 `env`、`envFrom` 或两者来设置环境变量。

如果在 `env` 中设置的变量与在 `envFrom` 中引用的 ConfigMap 中设置的变量名称相同，则将优先使用 `env` 中指定的值。例如，如果您在 `env` 和 `envFrom` 中引用的 ConfigMap 中同时设置了变量 `GREETING`，则 `env` 中指定的值将覆盖 ConfigMap 中的值。

## 在命令参数中使用环境变量

虽然将配置数据放入容器环境非常有用，但有时您需要将其作为容器入口点的命令行参数来提供。

您可以通过从 ConfigMap 中获取环境变量，如前面的示例那样，但使用特殊的 Kubernetes 语法 `$(VARIABLE)` 在命令行参数中引用它们。

在演示库的 *hello-config-args* 目录中，您可以在 *deployment.yaml* 文件中找到这个示例：

```
spec:
  containers:
    - name: demo
      image: cloudnatived/demo:hello-config-args
      `args``:`
        `-` `"``-greeting``"`
        `-` `"``$(GREETING)``"`
      ports:
        - containerPort: 8888
      env:
        - name: GREETING
          valueFrom:
            configMapKeyRef:
              name: demo-config
              key: greeting
```

在这里，我们为容器规范添加了一个 `args` 字段，这将把我们的自定义参数传递给容器的默认入口点（`/bin/demo`）。

Kubernetes 会将清单中形式为 `$(VARIABLE)` 的任何内容替换为环境变量 `VARIABLE` 的值。由于我们已创建了 `GREETING` 变量并从 ConfigMap 设置了其值，因此可以在容器的命令行中使用它。

当您应用这些清单时，`GREETING` 的值将以这种方式传递给演示应用程序：

```
`kubectl apply -f hello-config-args/k8s/`
configmap/demo-config created deployment.apps/demo created
```

您应该在您的网络浏览器中看到效果：

`Salut, 世界`

## 从 ConfigMap 创建配置文件

我们已经看到了几种不同的方法将 Kubernetes ConfigMap 中的数据传递到应用程序中：通过环境和通过容器命令行。然而，更复杂的应用程序通常希望从磁盘上的文件中读取它们的配置。

幸运的是，Kubernetes 提供了一种直接从 ConfigMap 创建这些文件的方法。首先，让我们更改我们的 ConfigMap，使其不再是单个键，而是存储一个完整的 YAML 文件（这个文件恰好只包含一个键，但如果您愿意，它可以是一百个）：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
data:
  config: |
    greeting: Buongiorno
```

与上一个示例中设置 `greeting` 键不同，我们正在创建一个名为 `config` 的新键，并将其分配给一个数据 *块*（YAML 中的竖线符号 `|` 表示接下来是一个原始数据块）。这就是数据的内容：

```
greeting: Buongiorno
```

它碰巧是有效的 YAML，但不要因此而困惑；它可以是 JSON、TOML、纯文本或任何其他格式。无论是什么，Kubernetes 最终都会将整个数据块按原样写入容器中的文件。

现在我们已经存储了必要的数据，让我们将其部署到 Kubernetes。在 demo 仓库的 *hello-config-file* 目录中，您会找到包含 Deployment 模板的内容：

```
spec:
  containers:
    - name: demo
      image: cloudnatived/demo:hello-config-file
      ports:
        - containerPort: 8888
      `volumeMounts``:`
      `-` `mountPath``:` `/config/`
        `name``:` `demo-config-volume`
        `readOnly``:` `true`
  `volumes``:`
  `-` `name``:` `demo-config-volume`
    `configMap``:`
      `name``:` `demo-config`
      `items``:`
      `-` `key``:` `config`
        `path``:` `demo.yaml`
```

查看 `volumes` 部分，您可以看到我们从现有的 `demo-config` ConfigMap 创建了一个名为 `demo-config-volume` 的 Volume。

在容器的 `volumeMounts` 部分，我们将此 Volume 挂载在 `mountPath: /config/` 上，选择 `config` 键，并将其写入路径 *demo.yaml*。其结果将是，Kubernetes 将在容器中创建一个文件 */config/demo.yaml*，其中包含 YAML 格式中的 `demo-config` 数据：

```
greeting: Buongiorno
```

演示应用程序将在启动时从此文件读取其配置。像以前一样，使用以下命令应用清单：

```
`kubectl apply -f hello-config-file/k8s/`
configmap/demo-config created deployment.apps/demo created
```

您应该在您的 Web 浏览器中看到结果：

`Buongiorno, 世界`

如果您想查看集群中 ConfigMap 数据的内容，请运行以下命令：

```
`kubectl describe configmap/demo-config`
Name:         demo-config Namespace:    default Labels:       <none> 
Data ==== config: greeting: Buongiorno 
Events:  <none>
```

如果您更新 ConfigMap 并更改其值，则相应的文件（在我们的示例中为 */config/demo.yaml*）将自动更新。某些应用程序可能会自动检测到其配置文件已更改并重新读取它；其他则可能不会。

一种选择是重新部署应用程序以获取更改（参见“更新 Pods 的配置更改”），但如果应用程序有触发实时重新加载的方式，例如 Unix 信号（例如 `SIGHUP`）或在容器中运行命令，这可能并不必要。

## 更新 Pods 的配置更改

假设您在集群中运行一个 Deployment，并且您想更改其 ConfigMap 中的一些值。如果您正在使用 Helm chart（参见“Helm：Kubernetes 包管理器”），有一个巧妙的技巧可以使其自动检测配置更改并重新加载您的 Pods。在您的 Deployment 规范中添加此注释：

```
`checksum/config``:` `{``{` `include` `(print` `$.Template.BasePath` `"/configmap.yaml")` `.`
    `|` `sha256sum` `}``}`
```

因为 Deployment 模板现在包含配置设置的哈希值，如果这些设置更改，哈希也会更改。当您运行 `helm upgrade` 时，Helm 将检测到 Deployment 规范已更改，并重新启动所有 Pods。

# Kubernetes Secrets

我们已经看到 Kubernetes ConfigMap 对象提供了一种灵活的方式来存储和访问集群中的配置数据。然而，大多数应用程序都有一些涉及密码或 API 密钥等机密和敏感的配置数据。虽然我们可以使用 ConfigMaps 来存储这些数据，但这并不是一个理想的解决方案。

相反，Kubernetes 提供了一种特殊的对象类型来存储秘密数据：Secret。让我们看一个如何在演示应用程序中使用它的示例。

首先，这是 Secret 的 Kubernetes 清单（参见 *hello-secret-env/k8s/secret.yaml*）：

```
apiVersion: v1
kind: Secret
metadata:
  name: demo-secret
stringData:
  magicWord: xyzzy
```

在本例中，秘密键是 `magicWord`，秘密值是单词 [`xyzzy`](https://oreil.ly/Ww0ME)（在计算中非常有用）。与 ConfigMap 一样，你可以在 Secret 中放入多个键和值。这里为了简单起见，我们只使用一个键值对。

## 使用 Secrets 作为环境变量

就像 ConfigMaps 一样，Secrets 可以通过将它们放入环境变量或者将它们作为文件挂载到容器的文件系统中，从而对容器可见。在这个例子中，我们将设置一个环境变量，其值为 Secret 的值：

```
spec:
  containers:
    - name: demo
      image: cloudnatived/demo:hello-secret-env
      ports:
        - containerPort: 8888
      env:
        - name: MAGIC_WORD
          valueFrom:
            `secretKeyRef``:`
              `name``:` `demo-secret`
              `key``:` `magicWord`
```

我们设置了环境变量 `MAGIC_WORD`，方式与使用 ConfigMap 时完全相同，只是现在是一个 `secretKeyRef` 而不是 `configMapKeyRef`（参见 “从 ConfigMaps 设置环境变量”）。

在演示库目录中运行以下命令来应用这些清单：

```
`kubectl apply -f hello-secret-env/k8s/`
deployment.apps/demo created secret/demo-secret created
```

与之前一样，将本地端口转发到 Deployment，以便在 Web 浏览器中查看结果：

```
`kubectl port-forward deploy/demo 9999:8888`
Forwarding from 127.0.0.1:9999 -> 8888 Forwarding from [::1]:9999 -> 8888
```

浏览到 *http://localhost:9999/*，你应该能看到：

`魔法词是 "xyzzy"`

## 写入 Secrets 到文件

在本例中，我们将把 Secret 作为文件挂载到容器中。你可以在演示库的 *hello-secret-file* 文件夹中找到这个示例的代码。

为了在容器中将 Secret 挂载为文件，我们使用了如下的 Deployment：

```
spec:
  containers:
    - name: demo
      image: cloudnatived/demo:hello-secret-file
      ports:
        - containerPort: 8888
      `volumeMounts``:`
        `-` `name``:` `demo-secret-volume`
          `mountPath``:` `"``/secrets/``"`
          `readOnly``:` `true`
  `volumes``:`
    `-` `name``:` `demo-secret-volume`
      `secret``:`
        `secretName``:` `demo-secret`
```

就像我们在 “从 ConfigMaps 创建配置文件” 中所做的一样，我们创建了一个 Volume（本例中为 `demo-secret-volume`），并在 spec 的 `volumeMounts` 部分将其挂载到容器上。`mountPath` 是 `/secrets`，Kubernetes 将在该目录下为 Secret 中定义的每个键值对创建一个文件。

在示例 Secret 中，我们只定义了一个名为 `magicWord` 的键值对，因此此清单将在容器中创建只读文件 */secrets/magicWord*，文件的内容将是秘密数据。

如果你像前面的示例一样应用这个清单，你应该能看到相同的结果：

`魔法词是 "xyzzy"`

## 读取 Secrets

在前面的部分中，我们能够使用 `kubectl describe` 查看 ConfigMap 中的数据。我们能否对 Secret 也做同样的操作呢？

```
`kubectl describe secret/demo-secret`
Name:         demo-secret Namespace:    default Labels:       <none> Annotations: Type:         Opaque 
Data ==== magicWord:  5 bytes
```

请注意，这次不显示实际数据。Kubernetes 的 Secrets 是 `Opaque` 类型，这意味着它们不会显示在 `kubectl describe` 的输出中，也不会出现在日志消息或终端中。这可以防止秘密数据意外暴露。

您可以通过使用 `kubectl get` 命令以 YAML 输出格式查看秘密数据的模糊版本：

```
`kubectl get secret/demo-secret -o yaml`
apiVersion: v1 data:
  `magicWord: eHl6enk=`
kind: Secret metadata: ... type: Opaque
```

### base64

那个 `eHl6enk=` 是什么？它看起来与我们原始的秘密数据不太相似。事实上，这是 Secret 的 *base64* 表示。Base64 是一种将任意二进制数据编码为字符字符串的方案。

因为秘密数据可能是不可打印的二进制数据（例如，传输层安全 [TLS] 加密密钥），所以 Kubernetes Secrets 总是以 base64 格式存储。

文本 `eHl6enk=` 是我们秘密词 `xyzzy` 的 base64 编码版本。您可以在终端使用 `base64 --decode` 命令验证这一点：

```
`echo "eHl6enk=" | base64 --decode`
xyzzy
```

尽管 Kubernetes 可以防止您意外地将秘密数据打印到终端或日志文件中，但如果您有权限读取特定命名空间中的 Secrets，则可以获取以 base64 格式编码的数据，然后解码它。

如果您需要对一些文本进行 base64 编码（例如，将其添加到 Secret 中），请使用带有 `-n` 标志的 `base64` 工具以避免包含换行符：

```
`echo -n xyzzy | base64`
eHl6enk=
```

## 访问 Secrets

谁可以读取或编辑 Secrets？这由 Kubernetes 访问控制机制 RBAC 控制，我们将在 “介绍基于角色的访问控制（RBAC）” 中详细讨论。

## 静态加密

那么，如果有人能够访问存储所有 Kubernetes 信息的 `etcd` 数据库呢？即使没有 API 权限读取 Secret 对象，他们也能访问秘密数据吗？

从 Kubernetes 版本 1.7 开始，支持 *静态加密*。这意味着存储在 `etcd` 数据库中的秘密数据实际上以加密形式存储在磁盘上，即使可以直接访问数据库的人也无法阅读。只有 Kubernetes API 服务器有解密此数据的密钥。在正确配置的集群中，应启用静态加密。

## 保持 Secrets 和 ConfigMaps

从版本 1.21 开始，Kubernetes 支持 [不可变的 Secrets](https://oreil.ly/5797O) 和 [不可变的 ConfigMaps](https://oreil.ly/baz5W)。在 Secret 或 ConfigMap 的清单中添加 `immutable: true` 将阻止其被修改。更改不可变的 Secret 或 ConfigMap 的唯一方法是删除并重新创建一个新的。

有时您会有一些 Kubernetes 资源，您永远不希望它们从集群中删除。如果您使用 Helm，则可以使用 Helm 特定的注释来防止资源被删除：

```
kind: Secret
metadata:
  annotations:
    "helm.sh/resource-policy": keep
```

# **保密管理策略**

在上一节的示例中，一旦我们将秘密数据存储在集群中，就能保护它免受未授权访问。但秘密数据在我们的清单文件中以明文形式表示。

您永远不应该在提交到源代码控制的文件中公开此类机密数据。那么，在应用于 Kubernetes 集群之前，如何安全地管理和存储机密数据呢？

无论您选择何种工具或策略来管理应用程序中的机密信息，您都需要至少回答以下问题：

1.  您应该将机密信息存储在何处，以确保其高可用性？

1.  如何使机密信息对运行中的应用程序可用？

1.  当您更换或更改机密信息时，运行中的应用程序需要执行哪些操作？

在本节中，我们将介绍一些流行的机密管理策略，并分析它们如何处理这些问题。

## 在版本控制中加密机密信息

处理机密信息的第一种，也许是最简单的选项是将机密信息直接存储在版本控制存储库中，与源代码一起，但是以加密形式存储。存储在源代码存储库中的机密信息绝不能以明文保存。相反，它们以一种只能在部署时或启动时使用某个受信任密钥解密的形式加密。然后，应用程序可以像处理任何其他配置数据一样读取和使用解密后的机密信息。

在版本控制中加密机密信息，您可以像处理应用程序代码更改一样审查和跟踪机密信息的更改。只要您的版本控制存储库具有高可用性，您的机密信息也将具有高可用性。

要更改或轮换机密信息，只需在源代码的本地副本中解密它们，更新它们，重新加密并将更改提交到版本控制。

尽管这种策略实施简单，除了密钥和加密/解密工具外没有依赖性，但也存在一些潜在的缺点。如果多个应用程序使用相同的机密信息，则所有应用程序都需要其源代码中的副本。这意味着更换机密信息会更费功夫，因为您必须确保找到并更改了所有实例。

在意外提交明文机密信息到版本控制也存在严重风险。错误是难免的，即使是私有版本控制存储库，任何这样提交的机密信息都应视为已泄露，应尽快进行更换。在源代码控制中处理带有加密机密信息的合并冲突也可能会有些棘手。

尽管如此，对于小团队或非关键机密信息而言，该策略可能是一个很好的起点。它相对不需要太多干预和易于设置，同时仍足够灵活，可以处理多个应用程序和不同类型的机密数据。在本章的最后一节中，我们将概述一些您可以使用的加密/解密工具选项，但首先，让我们简要描述其他机密管理策略。

## 使用专用机密管理工具

虽然在源代码中加密机密信息是一个相对简单的入门方法，但您可能希望评估使用专用的机密管理工具，例如 [HashiCorp 的 Vault](https://www.vaultproject.io) 或 [Square 的 Keywhiz](https://square.github.io/keywhiz)。您还可以考虑使用托管的云服务，例如 [AWS Secrets Manager](https://oreil.ly/IVAhS)，[Azure 的 Key Vault](https://oreil.ly/4WaXg)，或者 [Google 的 Secret Manager](https://oreil.ly/257Ue)。这些工具以高可用的方式安全地存储所有应用程序机密，并且还可以控制哪些用户和服务账户具有添加、删除、更改或查看机密的权限。

在机密管理系统中，所有操作都经过审计和可审查，这样更容易分析安全漏洞并证明合规性。一些工具还提供定期自动轮换机密的能力，这不仅在任何情况下都是个好主意，而且在许多企业安全策略中也是必需的。开发人员可以拥有自己的个人凭据，只有对他们负责的应用程序的读取或写入机密的权限。

应用程序如何从机密管理工具获取数据？一种常见的方法是使用一个具有对机密保险库只读访问权限的服务账户，以便每个应用程序只能读取其所需的机密。通常会使用一个初始化容器（参见“初始化容器”）首先拉取并解密机密，然后通过卷挂载到 Pod 中。

虽然集中式机密管理系统是目前最强大和灵活的选择，但它也会给您的基础架构增加相当多的复杂性，特别是如果您决定自行托管工具。使用托管解决方案将使您摆脱运行此基础架构的负担，但会增加您的云账单成本。您还需要为应用程序实施一些中间件或流程来安全地使用机密。虽然应用程序可以直接访问特定的机密保险库，但与直接在其前面添加一个获取机密并在应用程序启动时将其放入环境或配置文件中的层相比，这可能更昂贵和耗时。

# 使用 Sops 加密机密信息

现在让我们来看看一个流行的加密工具，您可以使用它在源代码中安全存储您的机密信息。Sops（*secrets operations* 的缩写），来自 Mozilla 项目，是一个与 YAML、JSON 或二进制文件兼容的加密/解密工具，支持多种加密后端，包括 [`age`](https://oreil.ly/d6jtA)，[Azure Key Vault](https://oreil.ly/4WaXg)，[AWS 的密钥管理服务 (KMS)](https://oreil.ly/pyDJT)，以及 [Google 的 Cloud KMS](https://oreil.ly/N7Bo4)。访问 [Sops 项目主页](https://oreil.ly/7oGbC) 获取安装和使用说明。

Sops 不是加密整个文件，而是仅加密键-值对中的个别密钥值。例如，如果你的明文文件包含：

```
password: foo
```

当你用 Sops 加密时，生成的文件将如下所示：

```
password: `ENC[AES256_GCM,data:p673w==,iv:YY=,aad:UQ=,tag:A=]`
```

这样可以轻松地查看代码，而无需解密值就能理解使用的哪个密钥。

## 使用 Sops 加密文件

让我们试试 Sops 来加密一个文件。正如我们提到的，Sops 实际上不处理加密本身；它将其委托给一个不同的后端工具。在本例中，我们将使用一个名为`age`的工具与 Sops 一起加密一个包含秘密的文件。最终结果将是一个可以安全提交到版本控制的文件。

我们不会详细讨论`age`加密的工作原理，但请知道，它像 SSH 和 TLS 一样，是一种*公钥*加密系统。它不是用单一密钥加密数据，而是使用一对密钥：一个公钥，一个私钥。你可以安全地与他人分享你的公钥，但绝不能泄露你的私钥。

现在让我们生成你的密钥对。首先，[安装 age](https://oreil.ly/CPGhH)，如果你还没有安装。

一旦安装完成，请运行以下命令生成一个新的密钥对：

```
`age-keygen -o key.txt`
Public key: age1fea...
```

一旦你的密钥成功生成，请记下`Public key`。它将是唯一的，并且标识刚刚创建的密钥。*key.txt*文件还包含你的私钥，因此应该安全地存储，永远不要提交到源代码控制中。

现在让我们使用 Sops 和`age`来加密一个文件。如果你还没有在你的机器上安装 Sops，请先安装它。

首先让我们创建一个测试秘密的 YAML 文件来加密：

```
`echo "password: secret123" > test.yaml`
`cat test.yaml`
password: secret123
```

现在，使用 Sops 进行加密。将你的密钥指纹传递给`--age`开关和上面的`Public key`，就像这样：

```
`sops --age age1fea... --encrypt --in-place test.yaml`
`cat test.yaml`
password: ENC[AES256_GCM,data:6U6tZpn/TCTG,iv:yfO6... ... sops:
 ... age: - recipient: age1fea...
```

成功了！*test.yaml*文件已经安全加密，`password`的值被加密并且只能用你的私钥解密。你还会注意到，Sops 在文件底部添加了一些元数据，以便将来如何解密它。

Sops 的一个很好的特性是，因为只有`password`的*值*被加密，文件的 YAML 格式保持不变，你仍然可以查看密钥的名称。

为了确保我们能够恢复加密数据，并检查它是否与我们输入的匹配，请运行：

```
`SOPS_AGE_KEY_FILE=$(pwd)/key.txt sops --decrypt test.yaml`
password: secret123
```

命令中的`SOPS_AGE_KEY_FILE`部分指向你最初与`age`生成的密钥文件的位置。你可以考虑将该文件存储在 Sops 期望的默认位置，即你的*$HOME/sops/*目录。

在部署应用程序时，可以使用 Sops 解密模式来生成应用程序使用的明文密钥——但记住在之后删除明文文件，并且永远不要提交它们到版本控制！

当作为集中式 CI/CD 流水线的一部分使用 Sops 时，您的部署服务器基础架构还需要一个`age`密钥，并且必须是一个[受信任的接收者](https://oreil.ly/zjPJs)，以便解密文件。

现在您知道如何使用 Sops，在您的源代码中可以加密任何敏感数据，无论是应用程序配置文件、Kubernetes YAML 资源还是其他任何内容。接下来，我们将向您展示如何在 Helm chart 中以这种方式使用 Sops。您不仅可以在使用 Helm 部署应用程序时解密秘密，还可以根据部署环境使用不同的秘密集：例如，`staging`与`production`（参见“使用 Sops 管理 Helm Chart 中的秘密”）。

还值得一提的是，如果您需要在 Helm chart 中管理加密的秘密，可以使用名为`helm-secrets`的插件来完成。当您运行`helm upgrade...`或`helm install...`时，`helm-secrets`将解密部署所需的秘密。有关`helm-secrets`的更多信息，包括安装和使用说明，请参阅[GitHub 仓库](https://oreil.ly/p3KBj)。

## 使用 KMS 后端

如果您在云中使用 Amazon KMS 或 Google Cloud KMS 进行密钥管理，也可以将它们与 Sops 一起使用。在我们的`age`示例中，使用 KMS 密钥的方式完全相同，但文件中的元数据将不同。文件底部的`sops:`部分可能看起来像这样：

```
sops:
  kms:
  - created_at: 1441570389.775376
    enc: CiC....Pm1Hm
    arn: arn:aws:kms:us-east-1:656532957310:key/920aff2e...
```

就像使用`age`一样，文件中嵌入了密钥 ID（`arn:aws:kms...`），以便 Sops 知道如何稍后解密它。

# Sealed Secrets

另一个在源代码控制中存储加密密钥的好选择是 Bitnami 团队维护的开源工具[Sealed Secrets](https://oreil.ly/Lq6Zo)。与 Sops 不同的是，在这种情况下，加密密钥实际上是在您的 Kubernetes 集群内生成、安装和存储的，使部署和解密过程变得简单直接。

安装了 Sealed Secrets 后，您可以使用`kubeseal`客户端工具来加密 Kubernetes Secret。这将生成一个新的`SealedSecret`，然后可以安全地提交到您的源代码仓库中，就像使用 Sops 加密的 YAML 文件一样。在应用到集群时，Sealed Secret 工具将从 Kubernetes 内部解密`SealedSecret`对象，并安全传递给您的应用程序 Pod。

# 摘要

有关 Kubernetes 相关的配置和密钥是人们最常询问我们的主题之一。我们很高兴能够为此奉献一章，并概述一些将您的应用程序连接到所需设置和数据的方法。

我们学到的最重要的事情：

+   将配置数据与应用程序代码分离，并使用 Kubernetes ConfigMaps 和 Secrets 进行部署。这样，每次更改密码时，您无需重新部署应用程序。

+   你可以通过直接在你的 Kubernetes 清单文件中编写数据，或使用`kubectl`将现有的 YAML 文件转换为 ConfigMap 规范，将数据输入到 ConfigMaps 中。

+   一旦数据在 ConfigMap 中，你可以将它插入到容器的环境中，或者插入到其入口点的命令行参数中。或者，你可以将数据写入挂载在容器上的文件中。

+   秘密的工作方式与 ConfigMaps 类似，不同之处在于数据在静态时加密，并在`kubectl`输出中混淆。

+   管理秘密的一个简单方法是将它们直接存储在源代码仓库中，但使用 Sops 或其他基于文本的加密工具对其进行加密。

+   不要忽视秘密管理，特别是在初始阶段。从团队理解的东西开始，提供一个安全的管理团队秘密的流程。

+   像 Vault 这样的专用秘密管理工具，或者托管的云 KMS 工具，增加了堆栈的成本和复杂性，但提供了更好的审计和灵活性，用于安全地保护你的秘密。

+   Sops 是一种加密工具，适用于像 YAML 和 JSON 这样的键值文件。它可以从本地密钥环或云密钥管理服务（如 Amazon KMS 和 Google Cloud KMS）获取其加密密钥。

+   Sealed Secrets 使得在源代码控制中存储加密的秘密并从 Kubernetes 集群内安全传递它们到你的应用程序变得容易。
