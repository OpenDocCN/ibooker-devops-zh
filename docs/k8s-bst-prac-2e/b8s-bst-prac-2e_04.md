# 第四章：配置、秘密和 RBAC

容器的可组合性使我们作为运维人员能够在运行时将配置数据引入容器中。这使得我们能够将应用程序的功能与其运行环境分离。通过容器运行时允许的约定，可以通过环境变量或在运行时将外部卷挂载到容器中，有效地改变应用程序的配置。作为开发人员，重要的是要考虑这种行为的动态性，并允许使用环境变量或从应用程序运行时用户可用路径读取配置数据。

将敏感数据（如秘密）移入原生 Kubernetes API 对象时，理解 Kubernetes 如何安全访问 API 非常重要。Kubernetes 中最常实现的安全方法是基于角色的访问控制（RBAC），以实施针对特定用户或组的可执行操作的精细权限结构。本章介绍了一些关于 RBAC 的最佳实践，并提供了一个小的入门指南。

# 通过 ConfigMaps 和 Secrets 进行配置

Kubernetes 允许您通过 ConfigMaps 或秘密资源原生地为我们的应用程序提供配置信息。两者之间的主要区别在于 pod 存储接收信息的方式以及数据存储在 etcd 数据存储中的方式。

## ConfigMaps

应用程序通常通过一些机制（如命令行参数、环境变量或系统可用的文件）消耗配置信息是非常普遍的。容器允许开发人员将此配置信息与应用程序解耦，从而实现真正的应用程序可移植性。ConfigMap API 允许注入提供的配置信息。ConfigMaps 非常适应应用程序的需求，并可以提供键/值对或复杂的批量数据，如 JSON、XML 或专有配置数据。

ConfigMaps 不仅为 pod 提供配置信息，还可以为更复杂的系统服务（如控制器、CRD、运算符等）提供信息。正如前面提到的，ConfigMap API 更适用于不是真正敏感的字符串数据。如果您的应用程序需要更敏感的数据，那么 Secrets API 更合适。

要使应用程序使用 ConfigMap 数据，可以将其注入为挂载到 pod 中的卷或环境变量的形式。

## Secrets

使用配置映射的原因和属性之多适用于秘密。 主要区别在于秘密的基本性质。 秘密数据应以一种可以轻松隐藏并在环境配置为这样的情况下可能加密的方式存储和处理。 秘密数据表示为 base64 编码的信息，重要的是要理解这不是加密。 一旦秘密注入到 pod 中，pod 本身就可以看到明文的秘密数据。

秘密数据意味着小量数据，默认在 Kubernetes 中限制为 1 MB 的 base64 编码数据，因此由于编码的开销，确保实际数据约为 750 KB。 Kubernetes 中有三种类型的秘密：

`generic`

这通常只是从文件、目录或使用 `--from-literal=` 参数从字符串字面值创建的常规键/值对，如下所示：

```
kubectl create secret generic mysecret --from-literal=key1=$3cr3t1
    --from-literal=key2=@3cr3t2
```

`docker-registry`

这是由 `kubelet` 在传递 `pod` 模板时使用的，如果有 `imagePullsecret`，则提供所需的凭据以进行私有 Docker 注册表的身份验证：

```
kubectl create secret docker-registry registryKey --docker-server
    myreg.azurecr.io --docker-username myreg --docker-password
    $up3r$3cr3tP@ssw0rd --docker-email ignore@dummy.com
```

`tls`

这将从有效的公共/私有密钥对创建一个传输层安全 (TLS) 秘密。 只要证书处于有效的 PEM 格式中，密钥对将被编码为秘密，并可以传递给 pod 用于 SSL/TLS 需求：

```
kubectl create secret tls www-tls --key=./path_to_key/wwwtls.key
    --cert=./path_to_crt/wwwtls.crt
```

秘密也仅在需要秘密的 pod 所在的节点上挂载到 tmpfs，并在需要秘密的 pod 被删除时删除。 这可以防止任何秘密遗留在节点磁盘上。 尽管这可能看起来很安全，但重要的是要知道，默认情况下，秘密以明文形式存储在 Kubernetes 的 etcd 数据存储中，系统管理员或云服务提供商必须努力确保 etcd 环境的安全性，包括 etcd 节点之间的 mTLS 和启用 etcd 数据的加密。 更近期的 Kubernetes 版本使用 etcd3，并具有启用 etcd 原生加密的能力； 但是，这是一个必须在 API 服务器配置中手动配置的过程，通过指定提供程序和适当的密钥介质来正确加密 etcd 中保存的秘密数据。 从 Kubernetes v1.10 开始 (在 v1.12 中已升级为 beta)，我们有 KMS 提供程序，它承诺通过使用第三方 KMS 系统来保持适当的密钥过程，从而提供更安全的密钥过程。

# 配置映射和密钥 API 的常见最佳实践

使用配置映射或秘密时出现的大多数问题源于对对象更新后数据处理方式的错误假设。 通过理解路规并添加一些技巧，以使遵守这些规则更容易，可以避免麻烦：

+   为了支持应用程序的动态变更而无需重新部署 Pod 的新版本，请将 ConfigMaps/Secrets 作为卷挂载，并配置应用程序使用文件监视器检测更改的文件数据并根据需要重新配置自身。以下代码展示了一个将 ConfigMap 和 Secret 文件作为卷挂载的 Deployment 示例：

```
apiVersion: v1
kind: ConfigMap
metadata:
    name: nginx-http-config
    namespace: myapp-prod
data:
  config: |
    http {
      server {
        location / {
        root /data/html;
        }

        location /images/ {
          root /data;
        }
      }
    }
```

```
apiVersion: v1
kind: Secret
metadata:
  name: myapp-api-key
type: Opaque
data:
  myapikey: YWRtd5thSaW4=
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mywebapp
  namespace: myapp-prod
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 8080
    volumeMounts:
    - mountPath: /etc/nginx
      name: nginx-config
    - mountPath: /usr/var/nginx/html/keys
      name: api-key
  volumes:
    - name: nginx-config
      configMap:
        name: nginx-http-config
        items:
        - key: config
          path: nginx.conf
    - name: api-key
      secret:
        name: myapp-api-key
        secretname: myapikey
```

###### 注意

使用 `volumeMounts` 时需要考虑几个事项。首先，一旦创建 ConfigMap/Secret，请将其作为 Pod 规范中的卷添加。然后将该卷挂载到容器的文件系统中。ConfigMap/Secret 中的每个属性名称将成为挂载目录中的一个新文件，并且每个文件的内容将是 ConfigMap/Secret 中指定的值。其次，避免使用 `volumeMounts.subPath` 属性来挂载 ConfigMaps/Secrets。这将阻止在更新 ConfigMap/Secret 时动态更新卷中的数据。

+   ConfigMaps/Secrets 必须在将要使用它们的 Pod 的命名空间中存在，然后才能部署 Pod。如果 ConfigMap/Secret 不存在，可以使用可选标志来防止 Pod 无法启动。

+   使用准入控制器来确保特定的配置数据或防止未设置特定配置值的部署。例如，如果您要求所有生产 Java 工作负载在生产环境中具有特定的 JVM 属性集。

+   如果您正在使用 Helm 将应用发布到您的环境中，可以使用生命周期钩子确保在应用 Deployment 之前部署 ConfigMap/Secret 模板。

+   有些应用程序要求将它们的配置作为单个文件（如 JSON 或 YAML 文件）应用。ConfigMap/Secret 允许通过使用 `|` 符号来传递整个原始数据块，如此示例所示：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-file
data:
  config: |
    {
      "iotDevice": {
        "name": "remoteValve",
        "username": "CC:22:3D:E3:CE:30",
        "port": 51826,
        "pin": "031-45-154"
      }
    }
```

+   如果应用程序使用系统环境变量来确定其配置，您可以使用 ConfigMap 数据的注入来创建环境变量映射到 Pod 中。有两种主要方法可以实现这一点：使用 `envFrom` 将 ConfigMap 中的每个键值对作为一系列环境变量挂载到 Pod 中，然后使用 `configMapRef` 或 `secretRef`；或者使用 `configMapKeyRef` 或 `secretKeyRef` 分配单独的键及其相应的值。

+   如果您使用 `configMapKeyRef` 或 `secretKeyRef` 方法，请注意，如果实际键不存在，这将阻止 Pod 启动。

+   如果您使用 `envFrom` 将 ConfigMap/Secret 中的所有键值对加载到 Pod 中，并且有些键被视为无效的环境值将会被跳过；但是，Pod 将被允许启动。Pod 的事件将包含一个原因为 `InvalidVariableNames` 的事件，并包含有关跳过的键的适当消息。以下代码展示了一个带有 ConfigMap 和 Secret 引用作为环境变量的 Deployment 示例：

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  mysqldb: myappdb1
  user: mysqluser1
```

```
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  rootpassword: YWRtJasdhaW4=
  userpassword: MWYyZDigKJGUyfgKJBmU2N2Rm
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-db-deploy
spec:
  selector:
    matchLabels:
      app: myapp-db
  template:
    metadata:
      labels:
        app: myapp-db
    spec:
      containers:
      - name: myapp-db-instance
        image: mysql
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 3306
        env:
          - name: MYSQL_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: rootpassword
          - name: MYSQL_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mysql-secret
                key: userpassword
          - name: MYSQL_USER
            valueFrom:
              configMapKeyRef:
                name: mysql-config
                key: user
          - name: MYSQL_DB
            valueFrom:
              configMapKeyRef:
                name: mysql-config
                key: mysqldb
```

+   如果需要向容器传递命令行参数，可以使用 `$(ENV_KEY)` 插值语法来源环境变量数据：

```
[...]
spec:
  containers:
  - name: load-gen
    image: busybox
    command: ["/bin/sh"]
args: ["-c", "while true; do curl $(WEB_UI_URL); sleep 10;done"]
    ports:
    - containerPort: 8080
    env:
    - name: WEB_UI_URL
      valueFrom:
        configMapKeyRef:
          name: load-gen-config
          key: url
```

+   在将 ConfigMap/Secret 数据作为环境变量消耗时，非常重要的是要理解，对 ConfigMap/Secret 中数据的更新 *不会* 在 Pod 中更新，需要重新启动 Pod。这可以通过删除 Pod 并让 ReplicaSet 控制器创建新的 Pod，或通过触发 Deployment 更新来完成，后者将遵循 Deployment 规范中声明的适当应用更新策略。

+   假设所有对 ConfigMap/Secret 的更改都需要更新整个 Deployment，这样可以确保即使使用环境变量或卷，代码也会采用新的配置数据。为了简化此过程，您可以使用 CI/CD 流水线更新 ConfigMap/Secret 的 `name` 属性，并同时更新 Deployment 中的引用，这将通过常规的 Kubernetes 更新策略触发 Deployment 更新。我们将在以下示例代码中探讨这一点。如果您使用 Helm 发布应用代码到 Kubernetes，您可以利用 Deployment 模板中的注释来检查 ConfigMap/Secret 的 `sha256` 校验和。当 ConfigMap/Secret 中的数据发生变化时，这将触发 Helm 使用 `helm upgrade` 命令更新 Deployment：

```
apiVersion: apps/v1
kind: Deployment
[...]
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml")
            . | sha256sum }}
[...]
```

# 专用于 Secrets 的最佳实践

由于 Secrets API 中敏感数据的特性，自然会有更具体的最佳实践，主要围绕数据本身的安全性：

+   如果您的工作负载不需要直接访问 Kubernetes API，最佳实践是阻止自动挂载服务账户（默认或操作员创建的）的 API 凭据。这将减少对 API 服务器的 API 调用，因为使用监视功能来更新 API 凭据数据，以便在凭据过期时更新。在非常大的集群或具有大量 Pod 的集群中，这将减少对控制平面的调用，从而减少可能导致性能下降的原因之一。可以在 ServiceAccount 或 Pod Spec 本身定义这一行为：

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app1-svcacct
automountServiceAccountToken: false
[...]
```

```
apiVersion: v1
kind: Pod
metadata:
  name: app1-pod
spec:
  serviceAccountName: app1-svcacct
  automountServiceAccountToken: false
[...]
```

+   Secrets API 的原始规范概述了一种可插拔的架构，允许根据需求配置实际的秘密存储。诸如 HashiCorp Vault、Aqua Security、Twistlock、AWS Secrets Manager、Google Cloud KMS 或 Azure Key Vault 等解决方案允许使用高级别的加密和审计功能的外部存储系统来存储秘密数据，这比 Kubernetes 本地提供的功能更强大。Linux 基金会项目 ExternalSecrets Operator 提供了一种本地方式来提供这种功能。

+   将 `imagePullSecrets` 分配给一个 `serviceaccount`，该 pod 将使用它来自动挂载密钥，而无需在 `pod.spec` 中声明它。您可以对应用程序所在命名空间的默认服务帐户进行修补，并直接添加 `imagePullSecrets` 到其中。这将自动将其添加到命名空间中的所有 pods：

```
Create the docker-registry secret first
kubectl create secret docker-registry registryKey --docker-server
myreg.azurecr.io --docker-username myreg --docker-password $up3r$3cr3tP@ssw0rd
--docker-email ignore@dummy.com

patch the default serviceaccount for the namespace you wish to configure
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name":
"registryKey"}]}'
```

+   使用 CI/CD 能力从安全保管库或加密存储中获取密钥，使用硬件安全模块（HSM）在发布流程中。这允许分离职责。安全管理团队可以创建和加密密钥，而开发人员只需引用预期的密钥名称。这也是确保更动态应用程序交付流程的首选 DevOps 流程。

# RBAC

在大型分布式环境中工作时，通常需要某种安全机制来防止对关键系统的未经授权访问。在计算机系统中，有许多关于如何限制资源访问的策略，但大多数都经历相同的阶段。通过类比常见的经历，如飞往外国的旅行者经历，可以帮助解释类似 Kubernetes 系统中发生的过程。我们可以利用护照、旅行签证和海关或边境警卫的共同旅行经验来展示这一过程：

护照（主体验证）

通常，您需要由某个政府机构颁发的护照，该机构将提供一些关于您身份的验证。这相当于 Kubernetes 中的用户帐户。Kubernetes 依赖外部授权机构来验证用户；然而，服务账户是 Kubernetes 直接管理的一种账户类型。

签证或旅行政策（授权）

各国将通过正式的短期协议如签证来接受持有其他国家护照的旅行者。签证还会概述访客可以做什么以及他们可以在访问国家停留多长时间，这取决于具体的签证类型。这相当于 Kubernetes 中的授权。Kubernetes 拥有不同的授权方法，但 RBAC 是最常用的一种。这允许对不同的 API 功能具有非常精细的访问控制。

边境巡逻或海关（准入控制）

进入外国时，通常有一个权威机构将检查必要的文件，包括护照和签证，并且在许多情况下检查进入该国家的物品，以确保其符合该国的法律。在 Kubernetes 中，这相当于准入控制器。准入控制器可以根据定义的规则和政策允许、拒绝或更改对 API 的请求。Kubernetes 拥有许多内置的准入控制器，如 PodSecurity、ResourceQuota 和 ServiceAccount 控制器。Kubernetes 还允许通过使用验证或变异准入控制器来实现动态控制器。

本节的重点是这三个领域中最不为人所理解和最被避免的：RBAC。在我们概述一些最佳实践之前，我们必须首先介绍 Kubernetes RBAC 的入门知识。

## RBAC 入门

Kubernetes 中的 RBAC 过程有三个主要组件需要定义：主题、规则和角色绑定。

### 主题

第一个组件是主题，实际上正在检查访问权限的项目。主题通常是用户、服务账户或组。如前所述，用户以及组由使用的授权模块在 Kubernetes 之外处理。我们可以将这些分类为基本认证、x.509 客户端证书或持有令牌。最常见的实现使用 x.509 客户端证书或类似 Azure Active Directory（Azure AD）、Salesforce 或 Google 等 OpenID Connect 系统的持有令牌。

###### 注意

Kubernetes 中的服务账户与用户账户不同，它们是命名空间绑定的，并且在 Kubernetes 内部存储；它们旨在代表进程，而不是人员，并由本地 Kubernetes 控制器管理。

### 规则

简单来说，这是可以在 API 中对特定对象（资源）或对象组执行的实际操作列表。动词与典型的创建、读取、更新和删除（CRUD）类型的操作对齐，但在 Kubernetes 中具有一些额外的功能，如`watch`、`list`和`exec`。对象与不同的 API 组件对齐，并按类别进行分组。例如，Pod 对象是核心 API 的一部分，并且可以通过`apiGroup: ""`引用，而部署则属于 app API 组。这是 RBAC 过程的真正力量，也可能是在创建适当的 RBAC 控制时让人感到恐惧和困惑的原因。

### 角色

角色允许定义规则定义的范围。Kubernetes 有两种类型的角色，`role`和`clusterRole`，它们的区别在于`role`特定于命名空间，而`clusterRole`是跨所有命名空间的集群范围角色。一个具有命名空间范围的角色定义示例如下：

```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-viewer
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

### RoleBindings

RoleBinding 允许将用户或组等主题映射到特定角色。绑定也有两种模式：`roleBinding`，它是特定于命名空间的；`clusterRoleBinding`，它是跨整个集群的。这里有一个命名空间范围内的示例 RoleBinding：

```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: noc-helpdesk-view
  namespace: default
subjects:
- kind: User
  name: helpdeskuser@example.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: pod-viewer # this must match the name of the Role or ClusterRole
                   # to bind to
  apiGroup: rbac.authorization.k8s.io
```

## RBAC 最佳实践

RBAC 是运行安全、可靠和稳定的 Kubernetes 环境的关键组件。RBAC 背后的概念可能很复杂；然而，遵循一些最佳实践可以减少一些主要障碍：

+   开发用于在 Kubernetes 中运行的应用程序几乎从不需要与它们关联的 RBAC 角色和 RoleBinding。只有当应用程序代码直接与 Kubernetes API 交互时，应用程序才需要 RBAC 配置。

+   如果应用程序确实需要直接访问 Kubernetes API，也许是根据添加到服务的端点来更改配置，或者需要列出特定命名空间中的所有 Pod，最佳实践是创建一个新的服务帐户，然后在 Pod 规范中指定它。然后，创建一个角色，该角色具有完成其目标所需的最少特权。

+   使用支持身份管理和如有需要的双因素认证的 OpenID Connect 服务。这将允许更高级别的身份验证。将用户组映射到具有完成工作所需的最少特权的角色。

+   除了上述实践，您应该使用即时（JIT）访问系统，允许站点可靠性工程师（SRE）、操作员以及可能需要在短时间内拥有升级特权以完成非常特定任务的人员。或者，这些用户应该有更受审计程度更高的不同身份来进行登录，并且这些帐户应该由用户帐户或绑定到角色的组分配更高的权限。

+   应为将应用程序部署到 Kubernetes 集群的 CI/CD 工具使用特定的服务帐户。这确保了集群内的审计性，并且能够理解谁可能已经在集群中部署或删除了任何对象。

+   如果您仍在使用 Helm v2 来部署应用程序，默认服务帐户是部署到`kube-system`的 Tiller。最好将 Tiller 部署到每个命名空间，并为该命名空间专门指定一个用于 Tiller 的服务帐户。在调用 Helm install/upgrade 命令的 CI/CD 工具中，作为预步骤，使用服务帐户和部署的特定命名空间初始化 Helm 客户端。每个命名空间可以使用相同的服务帐户名称，但命名空间应特定。建议迁移到 Helm v3，因为其核心原则之一是不再需要在集群中运行 Tiller。新架构完全基于客户端，并使用调用 Helm 命令的用户的 RBAC 访问权限。这与客户端基础工具对 Kubernetes API 的首选方法一致。

+   限制任何需要在**Secrets API**上执行`watch`和`list`操作的应用程序。这基本上允许应用程序或部署 Pod 的人查看该命名空间中的秘密。如果应用程序需要访问特定秘密的 Secrets API，则限制其仅使用`get`操作，而不是直接分配的那些秘密之外的其他秘密。

# 摘要

为了云原生交付应用程序的原则是另一个话题，但普遍认为，严格将配置与代码分离是成功的关键原则。通过适用于非敏感数据的本地对象，如 ConfigMap API，以及敏感数据的 Secrets API，Kubernetes 现在可以以声明性方法管理这一过程。随着越来越多的关键数据在 Kubernetes API 中原生表示和存储，通过适当的门控安全流程（如 RBAC 和集成认证系统），保护对这些 API 的访问至关重要。

正如你将在本书的其余部分中看到的那样，这些原则渗透到将服务正确部署到 Kubernetes 平台的每个方面，以构建一个稳定、可靠、安全和健壮的系统。
