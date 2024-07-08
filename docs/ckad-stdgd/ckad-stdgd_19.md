# 第十九章：配置映射和机密

Kubernetes 专门为定义配置数据而设计了两个基本概念：ConfigMap 和 Secret。这两个基本概念与 Pod 的生命周期完全解耦，这使得您可以更改它们的配置数据值，而无需重新部署 Pod。

本质上，配置映射和机密存储一组键值对。这些键值对可以作为环境变量注入到容器中，也可以作为卷挂载。图 19-1 说明了选项。

![ckd2 1901](img/ckd2_1901.png)

###### 图 19-1。消耗配置数据

表面上看，配置映射和机密在目的和结构上几乎相同；然而，它们之间存在一个轻微但重要的区别。配置映射存储纯文本数据，例如连接 URL、运行时标志，甚至结构化数据，如 JSON 或 YAML 内容。机密更适合表示像密码、API 密钥或 SSL 证书等敏感数据，并以 base64 编码形式存储数据。

# 配置映射和机密数据的加密

存储配置映射和机密对象数据的集群组件是 etcd。Etcd 默认以未加密形式管理此数据。您可以按照[Kubernetes 文档](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)中的描述配置 etcd 中的数据加密。Etcd 加密不在考试范围内。

本章大量涉及卷的概念。请参考第七章，以重新了解在 Pod 中使用卷的机制。

# 使用配置映射

应用程序通常实现使用配置数据来控制运行时行为的逻辑。配置数据的示例包括连接 URL 和网络通信选项（如第三方服务之间的重试次数或超时）在不同目标部署环境中的差异。

同一配置数据需要在多个 Pod 中提供并不罕见。您可以选择将信息集中在一个配置映射对象中，而不是在多个 Pod 定义中复制粘贴相同的键值对。配置映射对象保存配置数据，并可以由尽可能多的 Pod 消耗。因此，如果需要更改数据，则只需在一个位置进行修改即可。

## 创建配置映射

您可以通过使用命令行`create configmap`来创建配置映射。此命令要求您以选项的形式提供数据的来源。Kubernetes 区分如表 19-1 所示的四种不同选项。

表 19-1。配置映射解析的数据源选项

| 选项 | 示例 | 描述 |
| --- | --- | --- |
| `--from-literal` | `--from-literal=locale=en_US` | 字面值，即键-值对的纯文本形式 |
| `--from-env-file` | `--from-env-file=config.env` | 包含键值对并期望它们作为环境变量的文件 |
| `--from-file` | `--from-file=app-config.json` | 一个包含任意内容的文件 |
| `--from-file` | `--from-file=config-dir` | 一个包含一个或多个文件的目录 |

很容易混淆选项`--from-env-file`和`--from-file`。选项`--from-env-file`期望一个文件，该文件包含以新行分隔的`KEY=value`格式的环境变量。键值对遵循环境变量的典型命名约定（例如，键是大写的，并且单词之间用下划线分隔）。从历史上看，此选项已用于处理[Docker Compose `.env`文件](https://docs.docker.com/compose/environment-variables/env-file/)，尽管您可以将其用于包含环境变量的任何其他文件。

`--from-env-file`选项不强制或规范环境变量的典型命名约定。`--from-file`选项指向一个包含*任何*任意内容的文件或目录。这是用于包含应用程序读取的结构化配置数据文件的合适选项（例如，属性文件、JSON 文件或 XML 文件）。

以下命令展示了创建 ConfigMap 的过程。我们只需提供键值对作为字面量：

```
$ kubectl create configmap db-config --from-literal=DB_HOST=mysql-service \
  --from-literal=DB_USER=backend
configmap/db-config created
```

生成的 YAML 对象看起来像示例 19-1 中显示的对象。正如您所见，该对象在名为`data`的部分中定义了键值对。ConfigMap 没有`spec`部分。

##### 示例 19-1。ConfigMap YAML 清单

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  DB_HOST: mysql-service
  DB_USER: backend
```

您可能已经注意到，分配给 ConfigMap 数据的键遵循环境变量使用的典型命名约定。这样做的目的是在容器中以这种方式使用它们。

## 将 ConfigMap 作为环境变量消耗

创建 ConfigMap 后，您现在可以将其键值对作为环境变量注入到容器中。示例 19-2 展示了使用`spec.containers[].envFrom[].configMapRef`来按名称引用 ConfigMap。

##### 示例 19-2。将 ConfigMap 键值对注入到容器中

```
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - image: bmuschko/web-app:1.0.1
    name: backend
    envFrom:
    - configMapRef:
        name: db-config
```

从 YAML 清单创建 Pod 后，您可以通过运行`env`Unix 命令来检查容器中可用的环境变量。

```
$ kubectl exec backend -- env
...
DB_HOST=mysql-service
DB_USER=backend
...
```

注入的配置数据将列在容器可用的环境变量中。

## 作为卷挂载 ConfigMap

另一种在运行时配置应用程序的方法是处理机器可读的配置文件。假设我们决定将数据库配置存储在名为*db.json*的 JSON 文件中，其结构如示例 19-3 所示。

##### 示例 19-3。用于配置数据库信息的 JSON 文件

```
{
    "db": {
      "host": "mysql-service",
      "user": "backend"
    }
}
```

给定我们不处理字面上的键值对，创建 ConfigMap 对象时需要提供选项`--from-file`：

```
$ kubectl create configmap db-config --from-file=db.json
configmap/db-config created
```

示例 19-4 显示了 ConfigMap 的相应 YAML 清单。可以看到文件名成为键；文件内容使用了多行值。

##### 示例 19-4。定义结构化数据的 ConfigMap YAML 清单

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: db-config
data:
  db.json: |-                        ![1](img/1.png)
    {
       "db": {
          "host": "mysql-service",
          "user": "backend"
       }
    }
```

![1](img/#co_configmaps_and_secrets_CO1-1)

此 YAML 结构中使用的多行字符串语法(`|-`)删除了换行符并去除了尾随空白行。有关更多信息，请参见[YAML 多行字符串的语法](https://yaml-multiline.info/)。

Pod 将 ConfigMap 作为卷挂载到容器内的特定路径，并具有只读权限。假设应用程序将在启动时读取配置文件。示例 19-5 演示了 YAML 定义。

##### 示例 19-5。将 ConfigMap 挂载为卷

```
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - image: bmuschko/web-app:1.0.1
    name: backend
    volumeMounts:
    - name: db-config-volume
      mountPath: /etc/config
  volumes:
  - name: db-config-volume
    configMap:                      ![1](img/1.png)
      name: db-config
```

![1](img/#co_configmaps_and_secrets_CO2-1)

分配卷类型以通过名称引用 ConfigMap 对象。

要验证正确的行为，请打开一个交互式 shell 到容器。如下命令所示，目录*/etc/config*包含一个文件，其中包含我们在 ConfigMap 中使用的键。内容表示 JSON 配置：

```
$ kubectl exec -it backend -- /bin/sh
# ls -1 /etc/config
db.json
# cat /etc/config/db.json
{
    "db": {
      "host": "mysql-service",
      "user": "backend"
    }
}
```

应用程序代码现在可以从挂载路径读取文件，并根据需要配置运行时行为。

# 处理 Secrets

存储在 ConfigMaps 中的数据表示任意的明文键值对。与 ConfigMap 相比，Secret 原语旨在表示敏感的配置数据。Secret 数据的典型示例是用于认证的密码或 API 密钥。

# 存储在 Secret 中的值仅进行编码，而不是加密。

Secrets 期望每个条目的值都是 Base64 编码的。Base64 仅对值进行编码，但不进行加密。因此，任何具有其值访问权限的人都可以毫无问题地解码它。因此，应避免将 Secret 清单存储在源代码仓库中，与其他资源文件一起。

Kubernetes 项目决定选择“Secret”这个术语来表示敏感数据，有些令人遗憾。命名暗示数据实际上是秘密的，因此应进行加密。您可以从多种选项中选择，以确保在现实世界的项目中保持敏感数据的安全性。

[Bitnami 密封的 Secrets](https://github.com/bitnami-labs/sealed-secrets) 是一个成熟且经过验证的 Kubernetes 操作器，使用非对称加密技术进行数据加密。数据的清单表示，即 CRD 密封的 Secret，可以安全地存储在公共源代码库中。您无法自行解密此数据。安装了操作器的控制器是唯一可以解密数据的实体。另一个选项是将敏感数据存储在外部 Secrets 管理器中，例如 HashiCorp Vault 或 AWS Secrets Manager，并将其与 Kubernetes 集成。[External Secrets Operator](https://external-secrets.io/) 将外部 API 中的 Secrets 同步到 Kubernetes 中。考试只要求您理解内置的 Secret 原语，在下面的章节中进行了详细介绍。

## 创建一个 Secret

您可以使用命令`create secret`来创建一个 Secret。此外，需要提供一个必需的子命令，以确定 Secret 的类型。Table 19-2 列出了不同类型。Kubernetes 将 Internal Type 列中的值分配给 live 对象中的 `type` 属性。“专用 Secret 类型” 讨论了其他 Secret 类型及其用途。

Table 19-2\. 创建 Secret 的选项

| CLI 选项 | 描述 | 内部类型 |
| --- | --- | --- |
| `generic` | 从文件、目录或明文值创建 Secret | `Opaque` |
| `docker-registry` | 创建用于 Docker 注册表的 Secret，例如，在 Pod 请求时从私有注册表拉取镜像 | `kubernetes.io/dockercfg` |
| `tls` | 创建一个 TLS Secret | `kubernetes.io/tls` |

最常用的 Secret 类型是 `generic`。泛型 Secret 的选项与 ConfigMap 完全相同，如 Table 19-3 所示。

Table 19-3\. 解析为 Secret 的数据源选项

| 选项 | 示例 | 描述 |
| --- | --- | --- |
| `--from-literal` | `--from-literal=password=secret` | 明文键值对的文字值 |
| `--from-env-file` | `--from-env-file=config.env` | 包含键值对并期望它们成为环境变量的文件 |
| `--from-file` | `--from-file=id_rsa=~/.ssh/id_rsa` | 包含任意内容的文件 |
| `--from-file` | `--from-file=config-dir` | 包含一个或多个文件的目录 |

要演示功能，让我们创建一个类型为`generic`的秘密。该命令从作为命令行选项提供的文字中获取键值对：

```
$ kubectl create secret generic db-creds --from-literal=pwd=s3cre!
secret/db-creds created
```

使用命令创建时，Secret 将自动对提供的值进行 Base64 编码。这可以通过查看生成的 YAML 清单来观察到。您可以在 Example 19-6 中看到值 `s3cre!` 已被转换为 `czNjcmUh`，即其 Base64 编码的等效值。

##### Example 19-6\. 含有 Base64 编码值的 Secret

```
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque       ![1](img/1.png)
data:
  pwd: czNjcmUh    ![2](img/2.png)
```

![1](img/#co_configmaps_and_secrets_CO3-1)

类型为`Opaque`的值被分配用于表示通用敏感数据。

![2](img/#co_configmaps_and_secrets_CO3-2)

如果对象是通过命令式方式创建的，那么纯文本值会自动进行 Base64 编码。

如果您使用 YAML 清单开始创建秘密对象，则需要创建 Base64 编码的值以便将其分配给`data`属性。Unix 工具`base64`可以完成此工作。以下命令正好实现了这一点：

```
$ echo -n 's3cre!' | base64
czNjcmUh

```

请记住，如果您可以访问秘密对象或其 YAML 清单，那么您可以随时使用 Unix 工具`base64`解码 Base64 编码的值。因此，在定义清单时，您也可以指定纯文本值，我们将在下一节讨论。

### 使用纯文本值定义秘密数据

必须生成并分配 Base64 编码的值到秘密清单可能变得繁琐。秘密原语提供了`stringData`属性来替代`data`属性。使用`stringData`，您可以在清单文件中分配纯文本值，正如在示例 19-7 中展示的那样。

##### 示例 19-7\. 一个包含纯文本值的秘密

```
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
stringData:      ![1](img/1.png)
  pwd: s3cre!    ![2](img/2.png)
```

![1](img/#co_configmaps_and_secrets_CO4-1)

`stringData`属性允许分配纯文本键值对。

![2](img/#co_configmaps_and_secrets_CO4-2)

`pwd`键引用的值以纯文本格式提供。

Kubernetes 在从清单创建对象时会自动对`s3cre!`值进行 Base64 编码。结果是在示例 19-8 中展示的实时对象表示，您可以使用`kubectl get secret db-creds -o yaml`命令检索它。

##### 示例 19-8\. 一个实时秘密对象

```
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
data:              ![1](img/1.png)
  pwd: czNjcmUh!   ![2](img/2.png)
```

![1](img/#co_configmaps_and_secrets_CO5-1)

一个秘密的实时对象总是使用`data`属性，即使在清单中可能使用了`stringData`。

![2](img/#co_configmaps_and_secrets_CO5-2)

在创建时值已经被 Base64 编码。

您可以使用`Opaque`类型表示任意秘密数据。Kubernetes 提供了专门的秘密类型供您选择，以适应特定的用例。我们将在下一节讨论这些专门的秘密类型。

### 专门的秘密类型

您还可以使用其中一个[专门类型](https://kubernetes.io/docs/concepts/configuration/secret/#secret-types)来表示特定用例的配置数据，而不是使用`Opaque`秘密类型。类型`kubernetes.io/basic-auth`适用于基本认证，并期望键`username`和`password`。在撰写时，Kubernetes 不验证分配的键的正确性。

从这个定义创建的对象会自动为两个键的值进行 Base64 编码。示例 19-9 展示了一个具有类型`kubernetes.io/basic-auth`的秘密的 YAML 清单。

##### 示例 19-9\. 使用 kubernetes.io/basic-auth 秘密类型

```
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:                      ![1](img/1.png)
  username: bmuschko             ![2](img/2.png)
  password: secret               ![2](img/2.png)
```

![1](img/#co_configmaps_and_secrets_CO6-1)

使用`stringData`属性允许分配明文值。

![2](img/#co_configmaps_and_secrets_CO6-2)

指定了`kubernetes.io/basic-auth`秘密类型所需的强制键。

## 作为环境变量消费秘密

与 ConfigMaps 的环境变量类似，将秘密作为环境变量消费的工作方式。在这里，您将使用 YAML 表达式`spec.containers[].env⁠From[].secretRef`来引用秘密的名称。示例 19-10 将名为`secret-basic-auth`的秘密作为环境变量注入到名为`backend`的容器中。

##### 示例 19-10\. 将秘密键值对注入容器

```
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - image: bmuschko/web-app:1.0.1
    name: backend
    envFrom:
    - secretRef:
        name: secret-basic-auth
```

检查容器中的环境变量会显示，秘密值无需解码。这是 Kubernetes 自动完成的操作。因此，运行的应用程序无需实现自定义逻辑来解码该值。请注意，Kubernetes 不验证或规范化环境变量的典型命名约定，正如您在以下输出中看到的：

```
$ kubectl exec backend -- env
...
username=bmuschko
password=secret
...
```

### 重映射环境变量键

有时，存储在秘密中的键值对不符合环境变量的典型命名约定，或者在不影响正在运行的服务的情况下无法更改。您可以使用`spec.containers[].env[].valueFrom`属性重新定义用于将环境变量注入 Pod 中的键。示例 19-11 将键`username`转换为`USER`，将键`password`转换为`PWD`。

##### 示例 19-11\. 重映射秘密条目的环境变量键

```
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - image: bmuschko/web-app:1.0.1
    name: backend
    env:
    - name: USER
      valueFrom:
        secretKeyRef:
          name: secret-basic-auth
          key: username
    - name: PWD
      valueFrom:
        secretKeyRef:
          name: secret-basic-auth
          key: password
```

现在，容器可用的环境变量遵循环境变量的典型约定，并且我们改变了它们在应用程序代码中的消费方式：

```
$ kubectl exec backend -- env
...
USER=bmuschko
PWD=secret
...

```

重新分配环境变量键的相同机制适用于 ConfigMaps。您将使用属性`spec.containers[].env[].valueFrom.configMapRef`。

## 将秘密作为卷挂载

要演示将秘密作为卷挂载，我们将创建一个新的类型为`kubernetes.io/ssh-auth`的秘密。这种秘密类型捕获了 SSH 私钥的值，您可以使用命令`cat ~/.ssh/id_rsa`查看。要使用`create secret`命令处理 SSH 私钥文件，需要将其作为名为`ssh-privatekey`的文件可用：

```
$ cp ~/.ssh/id_rsa ssh-privatekey
$ kubectl create secret generic secret-ssh-auth --from-file=ssh-privatekey \
  --type=kubernetes.io/ssh-auth
secret/secret-ssh-auth created
```

将秘密作为卷挂载遵循两步骤的方法：首先定义卷，然后将其引用为一个或多个容器的挂载路径。卷类型称为`secret`，如示例 19-12 中使用的那样。

##### 示例 19-12\. 将秘密作为卷挂载

```
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
  - image: bmuschko/web-app:1.0.1
    name: backend
    volumeMounts:
    - name: ssh-volume
      mountPath: /var/app
      readOnly: true                ![1](img/1.png)
  volumes:
  - name: ssh-volume
    secret:
      secretName: secret-ssh-auth   ![2](img/2.png)
```

![1](img/#co_configmaps_and_secrets_CO7-1)

由作为卷挂载的秘密提供的文件无法修改。

![2](img/#co_configmaps_and_secrets_CO7-2)

请注意指向**秘密名称**的属性`secretName`与 ConfigMap 的属性`name`不同。

您将在挂载路径*/var/app*中找到名为`ssh-privatekey`的文件。要验证，请打开交互式 shell 并呈现文件内容。文件内容不是 Base64 编码的：

```
$ kubectl exec -it backend -- /bin/sh
# ls -1 /var/app
ssh-privatekey
# cat /var/app/ssh-privatekey
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,8734C9153079F2E8497C8075289EBBF1
...
-----END RSA PRIVATE KEY-----
```

# 总结

应用程序运行时行为可以通过将配置数据作为环境变量注入或通过挂载到路径来控制。在 Kubernetes 中，此配置数据以 API 资源 ConfigMap 和 Secret 的形式表示为键值对。ConfigMap 适用于明文数据，而 Secret 对值进行 Base64 编码以混淆值。对于像凭据和 SSH 私钥这样的敏感信息，Secret 更合适。

# 考试要点

练习使用命令式和声明式方法创建 ConfigMap 对象

最快创建这些对象的方法是使用命令式`kubectl create configmap`命令。了解如何使用不同的命令行标志提供数据的方法。ConfigMap 在 YAML 清单的`data`部分指定了明文键值对。

练习使用命令式和声明式方法创建 Secret 对象

使用命令式命令`kubectl create secret`创建 Secret 时无需对提供的值进行 Base64 编码。`kubectl`会自动执行编码操作。声明式方法要求 Secret 的 YAML 清单在`data`部分指定 Base64 编码的值。如果您更喜欢提供明文值，可以使用`stringData`方便属性代替`data`属性。实时对象将使用 Base64 编码值。在运行时，使用`data`和`stringData`之间没有功能上的区别。

了解专门的 Secret 类型的目的

Secrets 提供专门的类型，例如`kubernetes.io/basic-auth`或`kuber⁠netes.io/service-account-token`，以表示特定用例的数据。请阅读 Kubernetes 文档中关于不同类型的内容，并了解它们的用途。

知道如何检查 ConfigMap 和 Secret 数据

考试可能涉及现有的 ConfigMap 和 Secret 对象。您需要了解如何使用`kubectl get`或`kubectl describe`命令检查这些对象的数据。Secret 的实时对象始终以 Base64 编码格式表示值。

在 Pod 中练习使用 ConfigMaps 和 Secrets

ConfigMaps 和 Secrets 的主要用例是从 Pod 消耗数据。Pod 可以将配置数据作为环境变量注入到容器中，或者将配置数据作为卷挂载。在考试中，您需要熟悉这两种消耗方法。

# 示例练习

这些练习的解决方案可在附录 A 中找到。

1.  在这个练习中，你将首先从一个 YAML 配置文件创建一个 ConfigMap。随后，你将创建一个 Pod，将 ConfigMap 作为 Volume 消费，并作为文件检查键-值对。

    导航至已检出的 GitHub 仓库[*bmuschko/ckad-study-guide*](https://github.com/bmuschko/ckad-study-guide)的目录*app-a/ch19/configmap*。检查名为*application.yaml*的 YAML 配置文件。

    从该文件创建名为`app-config`的新 ConfigMap。

    创建名为`backend`的 Pod，将 ConfigMap 作为 Volume 挂载到路径`/etc/config`。容器运行`nginx:1.23.4-alpine`镜像。

    进入 Pod 的 Shell，并检查挂载点处的文件。

1.  在这个练习中，首先从字面值创建一个 Secret。接下来，你将创建一个 Pod，并将 Secret 作为环境变量消费。最后，你将在容器内部打印出其值。

    创建名为`db-credentials`的新 Secret，包含键/值对`db-password=passwd`。

    创建名为`backend`的 Pod，使用 Secret 作为名为`DB_PASSWORD`的环境变量，并以`nginx:1.23.4-alpine`镜像运行容器。

    进入 Pod 的 Shell，并打印出创建的环境变量。你应该能找到`DB_PASSWORD`变量。
