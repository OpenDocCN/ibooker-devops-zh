# 第六章：Linkerd CLI

Linkerd 命令行界面（CLI）是与 Linkerd 控制平面进行交互的有用工具。CLI 可以帮助你检查 Linkerd 实例的健康状态，查看有关代理和证书的详细信息，排除异常行为并查看策略。这是直接与 Linkerd 交互的推荐方式。它处理你需要处理的所有主要任务，并为验证和检查 Linkerd 提供重要工具。

在本章中，我们将介绍 CLI 可以执行的一些最有用的操作，并说明如何最好地利用它。当然，随着新的 Linkerd 发布的推出，CLI 当前也在不断发展，因此随时关注[官方文档](https://oreil.ly/0GjuM)是非常重要的。

# 安装 CLI

CLI 与 Linkerd 的其他部分一起进行版本管理，因此当你安装 CLI 时，需要选择使用哪个发布通道。

要从稳定通道安装，请参考供应商说明（例如[Linkerd 的 Buoyant Enterprise](https://oreil.ly/6apOU)）。

要从 edge 通道完全安装开源的 Linkerd，你可以参考[Linkerd 快速入门](https://oreil.ly/A3Lyl)。在撰写本文时，操作如下：

```
$ curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install-edge | sh
```

无论哪种情况，一旦安装了 CLI，你都需要将其以适合你的 shell 的方式添加到 `PATH` 中。例如，如果你使用 `bash`，你可以直接修改 `PATH` 变量：

```
$ export PATH=$HOME/.linkerd2/bin:$PATH
```

## 更新 CLI

要更新 CLI，只需重新运行安装命令。随着时间的推移，你会在本地存储多个版本，并可以在它们之间进行选择。

## 安装特定版本

通常，Linkerd CLI 安装程序（任何通道都是如此）将安装最新版本的 CLI。你可以通过设置 `LINKERD2_VERSION` 环境变量来强制安装特定版本，例如，在使用 edge 通道时：

```
$ curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install-edge \
    | LINKERD2_VERSION="stable-2.13.12" sh
```

# 设置 LINKERD2_VERSION 环境变量来执行 sh 命令，而不是 curl。

注意在上述命令中设置 `LINKERD2_VERSION` 环境变量的位置：它需要设置为执行 `curl` 下载的脚本的 `sh` 命令，而不是 `curl` 命令本身。为 `curl` 命令设置环境变量不会起作用。

## 其他安装方式

如果你使用的是 Mac，[Homebrew](https://brew.sh) 是安装 CLI 的简单方法：只需 `brew install linkerd`。你也可以直接从[Linkerd 发布页面](https://oreil.ly/vcUOa)下载 CLI。

# 使用 CLI

CLI 的工作方式与任何其他 Go CLI 类似，例如 `kubectl`：

```
$ linkerd *`command`* *`[``options``]`*
```

`*command*` 告诉 CLI 你希望做什么；`*options*` 是特定命令的可选参数。你可以随时使用 `--help` 选项获取帮助。例如，`linkerd --help` 将告诉你可用的命令：

```
$ linkerd --help
```

```
linkerd manages the Linkerd service mesh.

Usage:
  linkerd [command]

Available Commands:
  authz        List authorizations for a resource
  check        Check the Linkerd installation for potential problems
  completion   Output shell completion code for the specified shell (bash, zsh
               or fish)
  diagnostics  Commands used to diagnose Linkerd components
  help         Help about any command
  identity     Display the certificate(s) of one or more selected pod(s)
  inject       Add the Linkerd proxy to a Kubernetes config
  install      Output Kubernetes configs to install Linkerd
  install-cni  Output Kubernetes configs to install Linkerd CNI
  jaeger       jaeger manages the jaeger extension of Linkerd service mesh
  multicluster Manages the multicluster setup for Linkerd
  profile      Output service profile config for Kubernetes
  prune        Output extraneous Kubernetes resources in the linkerd control
               plane
  uninject     Remove the Linkerd proxy from a Kubernetes config
  uninstall    Output Kubernetes resources to uninstall Linkerd control plane
  upgrade      Output Kubernetes configs to upgrade an existing Linkerd control
               plane
  version      Print the client and server version information
  viz          viz manages the linkerd-viz extension of Linkerd service mesh

Flags:
      --api-addr string            Override kubeconfig and communicate directly
                                   with the control plane at host:port (mostly
                                   for testing)
      --as string                  Username to impersonate for Kubernetes
                                   operations
      --as-group stringArray       Group to impersonate for Kubernetes
                                   operations
      --cni-namespace string       Namespace in which the Linkerd CNI plugin is
                                   installed (default "linkerd-cni")
      --context string             Name of the kubeconfig context to use
  -h, --help                       help for linkerd
      --kubeconfig string          Path to the kubeconfig file to use for CLI
                                   requests
  -L, --linkerd-namespace string   Namespace in which Linkerd is installed
                                   ($LINKERD_NAMESPACE) (default "linkerd")
      --verbose                    Turn on debug logging

Use "linkerd [command] --help" for more information about a command.
```

正如这个输出所示，你还可以获取特定命令的帮助信息。例如，`linkerd check --help` 将获取 `check` 命令的帮助，如下所示：

```
$ linkerd check --help
```

```
Check the Linkerd installation for potential problems.

The check command will perform a series of checks to validate that the linkerd
CLI and control plane are configured correctly. If the command encounters a
failure it will print additional information about the failure and exit with a
non-zero exit code.

Usage:
  linkerd check [flags]

Examples:
  # Check that the Linkerd control plane is up and running
  linkerd check

  # Check that the Linkerd control plane can be installed in the "test"
  # namespace
  linkerd check --pre --linkerd-namespace test

  # Check that the Linkerd data plane proxies in the "app" namespace are up and
  # running
  linkerd check --proxy --namespace app

Flags:
      --cli-version-override string   Used to override the version of the cli
                                      (mostly for testing)
      --crds                          Only run checks which determine if the
                                      Linkerd CRDs have been installed
      --expected-version string       Overrides the version used when checking
                                      if Linkerd is running the latest version
                                      (mostly for testing)
  -h, --help                          help for check
      --linkerd-cni-enabled           When running pre-installation checks
                                      (--pre), assume the linkerd-cni plugin is
                                      already installed, and a NET_ADMIN check
                                      is not needed
  -n, --namespace string              Namespace to use for --proxy checks
                                      (default: all namespaces)
  -o, --output string                 Output format. One of: table, json, short
                                      (default "table")
      --pre                           Only run pre-installation checks, to
                                      determine if the control plane can be
                                      installed
      --proxy                         Only run data-plane checks, to determine
                                      if the data plane is healthy
      --wait duration                 Maximum allowed time for all tests to pass
                                      (default 5m0s)

Global Flags:
      --api-addr string            Override kubeconfig and communicate directly
                                   with the control plane at host:port (mostly
                                   for testing)
      --as string                  Username to impersonate for Kubernetes
                                   operations
      --as-group stringArray       Group to impersonate for Kubernetes
                                   operations
      --cni-namespace string       Namespace in which the Linkerd CNI plugin is
                                   installed (default "linkerd-cni")
      --context string             Name of the kubeconfig context to use
      --kubeconfig string          Path to the kubeconfig file to use for CLI
                                   requests
  -L, --linkerd-namespace string   Namespace in which Linkerd is installed
                                   ($LINKERD_NAMESPACE) (default "linkerd")
      --verbose                    Turn on debug logging
```

# 选定的命令

`linkerd` CLI 支持许多命令。正如始终如此的[官方文档](https://oreil.ly/M3qdg)所述，这里我们将总结一些最广泛使用的命令。这些是您应该随时掌握的命令。

## linkerd version

关于了解的第一个命令是 `linkerd version`，它简单地报告 `linkerd` CLI 的运行版本和（如果可能的话）Linkerd 控制平面的版本：

```
$ linkerd version
Client version: stable-2.14.6
Server version: stable-2.14.6
```

如果您的集群中没有运行 Linkerd，`linkerd version` 将显示服务器版本为 `unavailable`。

如果 `linkerd version` 无法与您的集群通信，它将将其视为错误。您可以使用 `--client` 选项仅检查 CLI 本身的版本，甚至不尝试与集群通信：

```
$ linkerd version --client
Client version: stable-2.14.6
```

# CLI 版本与控制平面版本的对比

非常重要的是记住 CLI 版本与控制平面版本*独立*。一些 CLI 命令非常复杂，进行了许多微妙的操作，因此确保您的 CLI 版本与控制平面版本匹配至关重要。主要版本相差一个版本是可以的，但不支持超过一个版本的差异。

## linkerd check

`linkerd check` 命令提供了一个快速查看集群中 Linkerd 健康状态的视图。它会检测许多已知的失败条件，并允许您运行特定于扩展的健康检查。这个看似简单的命令实际上提供了许多强大的工具，用于验证和检查当前网格的状态。

使用 `linkerd check` 的最简单也是最完整的方法是不带参数地运行它：

```
$ linkerd check
```

这将运行一组默认检查，既相当详尽，也在合理的时间内完成，包括（除了许多其他事项）：

+   确保 Linkerd 在默认命名空间中正确安装。

+   检查 Linkerd 证书是否有效

+   运行所有已安装扩展的检查

+   双重检查必要的权限

运行此命令将为您提供有关集群中 Linkerd 当前状态的许多见解，事实上，如果您需要针对 Linkerd 提交 bug 报告，您将*总是*被要求包含 `linkerd check` 的输出。

### linkerd check --pre

预检选项运行一组检查，确保您的 Kubernetes 环境准备好安装 Linkerd：

```
$ linkerd check --pre
```

这是唯一使用 `linkerd check` 的情况，**不要求** Linkerd 已经安装。预检查确保您的集群符合运行 Linkerd 的最低技术要求，并且您有执行核心 Linkerd 安装所需的适当权限。这是准备在新集群上安装 Linkerd 的一个有用步骤。

# 预检和 CNI 插件

如果您计划安装带有 CNI 插件的 Linkerd，则需要运行 `linkerd check --pre --linkerd-cni-enabled`，以便 `linkerd check` 不尝试检查 `NET_ADMIN` 权限。

### linkerd check --proxy

你还可以告诉 `linkerd check` 明确检查数据平面：

```
$ linkerd check --proxy
```

代理检查运行了基本 `linkerd check` 命令执行的许多检查，但它还运行了特定于数据平面的额外检查，例如验证 Linkerd 代理是否正在运行。

### Linkerd 扩展检查

每个安装的 Linkerd 扩展都有自己特定的一组检查，它将在 `linkerd check` 运行期间运行。如果需要，你还可以仅针对特定扩展运行检查，例如，你可以仅运行 Linkerd Viz 扩展的检查：

```
$ linkerd viz check
```

# 为什么要限制检查？

请记住，`linkerd check` 不带任何参数将运行所有已安装扩展的检查。将检查限制到单个扩展主要有助于减少 `linkerd check` 运行的时间。

### `linkerd check` 的附加选项

`linkerd check` 命令遵循所有全局 CLI 覆盖项，例如更改 Linkerd 安装的命名空间 (`--namespace`)、修改你的 `KUBECONFIG` (`--kubeconfig`) 或 Kubernetes 上下文 (`--context`)。此外：

+   `--output` 允许你指定输出类型，如果想要覆盖默认的表格输出，这将非常有用。选项包括 `json`、`basic`、`short` 和 `table`。如果打算以程序化方式消费检查数据，输出 JSON 尤其有帮助。

+   `--wait` 可以覆盖检查等待的时间，在出现问题时。默认值为 5 分钟，在许多情况下可能会显得过长。

## linkerd inject

`linkerd inject` 命令读取 Kubernetes 资源并输出已修改以适当添加 Linkerd 代理容器的新版本。`linkerd inject` 命令：

+   从其标准输入、本地文件或 HTTPS URL 读取资源

+   可以同时操作多个资源

+   知道只修改 Pods，并且不会触及其他类型的资源

+   允许你配置代理

+   将修改后的资源输出到标准输出，实际应用它们的任务留给了你

最后一点值得重申：`linkerd inject` 永远不会直接修改其来源之一。相反，它会输出修改后的 Kubernetes 资源，以便你自行应用它们，将它们包含在 Git 存储库中，或者根据环境进行其他适当操作。这种“输出而不覆盖”的习惯用语在整个 `linkerd` CLI 中很常见。

使用 `linkerd inject` 可以非常简单地进行如下操作：

```
$ linkerd inject https://url.to/yml | kubectl apply -f -
```

如常，你可以通过运行 `linkerd inject --help` 查找更多示例并查看完整文档。

# 你必须处理应用注入的资源

关于 `linkerd inject` 最重要的一点是，它本身并不会对你的集群做出任何更改。你始终需要自行将 `linkerd` CLI 的输出应用到你的集群中，可以通过简单地将输出传递给 `kubectl apply`，或者提交到 GitOps 进行处理，或者其他方式。

### 注入到入口模式

`--ingress` 标志为工作负载设置入口模式注解。在在您的入口上设置此标志或相应的注解之前，请验证是否需要。您可以查看[入口文档](https://oreil.ly/OgAej)以获取有关入口模式的更多详细信息。

### 手动注入

默认情况下，`linkerd inject` 只会向您的工作负载 Pod 添加 `linkerd.io/inject` 注解，信任代理注入器来完成重型工作。设置 `--manual` 标志会指示 CLI 直接向您的清单添加 sidecar 容器，绕过代理注入器。

`--manual` 标志为在需要控制一些常规配置机制不支持的代理配置时提供了一个有价值的工具。但是，请注意在直接修改代理配置时要小心，因为您可能会迅速使自己的代理配置脱离整体配置。

### 注入调试 sidecar

设置 `--enable-debug-sidecar` 将向您的工作负载添加一个注解，这会导致代理注入器向您的 Pod 添加一个额外的调试 sidecar。在尝试使用调试 sidecar 之前，务必阅读第十五章和[调试容器文档](https://oreil.ly/CVc6-)。

## `linkerd identity`

`linkerd identity` 命令为故障排除 Pod 证书提供了一个有用的工具。它允许您查看任何 Pod 或 Pods 的证书详细信息；例如，这里是如何获取属于 Linkerd 目标控制器的 Pod 的身份：

```
$ linkerd identity -n linkerd linkerd-destination-7447d467f8-f4n9w
```

```
POD linkerd-destination-7447d467f8-f4n9w (1 of 1)

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 3 (0x3)
    Signature Algorithm: ECDSA-SHA256
        Issuer: CN=identity.linkerd.cluster.local
        Validity
            Not Before: Apr 5 13:51:13 2023 UTC
            Not After : Apr 6 13:51:53 2023 UTC
        Subject: CN=linkerd-destination.linkerd.serviceaccount.identity.linkerd.cluster.local
        Subject Public Key Info:
            Public Key Algorithm: ECDSA
                Public-Key: (256 bit)
                X:
                    98:41:63:15:e1:0e:99:81:3c:ee:18:a5:55:fe:a5:
                    40:bd:cf:a2:cd:c2:e8:30:09:8c:8a:c6:8a:20:e7:
                    3c:cf
                Y:
                    53:7e:3c:05:d4:86:de:f9:89:cb:73:e9:37:98:08:
                    8f:e5:ec:39:c3:6c:c7:42:47:f0:ea:0a:c7:66:fe:
                    8d:a5
                Curve: P-256
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Authority Key Identifier:
                keyid:37:C0:12:A1:AC:2D:A9:36:2D:35:83:6B:5C:99:9A:A2:5E:9C:E5:C5
            X509v3 Subject Alternative Name:
                DNS:linkerd-destination.linkerd.serviceaccount.identity.linkerd.cluster.local

    Signature Algorithm: ECDSA-SHA256
         30:45:02:20:4a:fb:02:db:17:e7:df:64:a4:7b:d2:08:a2:2e:
         66:e5:a4:74:14:35:d5:1a:f7:fc:15:95:9b:73:60:dd:78:a4:
         02:21:00:8c:12:fb:bf:80:7a:c4:25:91:0c:ac:03:37:ca:e0:
         82:d5:9d:9b:54:f1:20:b0:f0:14:e0:ef:ae:a8:ba:70:00
```

# 您的 Pod 身份将不同

如果您尝试此命令，则您的 Pod ID 和具体的证书信息将不同。但是，`linkerd identity` 提供的所有信息都不是敏感信息；它只显示公共信息。运行它是完全安全的。

您可以使用此命令的输出检查给定 Pod 证书的有效性。它还会告诉您哪个权威签署了证书，因此您可以检查它是否由正确的中介和根 CA 签署。

## `linkerd diagnostics`

`linkerd diagnostics` 命令是一个强大的工具，允许平台运营商直接从 Linkerd 中收集信息。它将允许您直接从各种 Linkerd 组件的指标端点中抓取详细信息。

该命令还允许您诊断诸如 Linkerd 的快速失败错误之类的难以识别的条件，方法是列出给定服务的端点。这里给出了一些示例；还可以查看 Linkerd 网站上的[最新文档](https://oreil.ly/egNPA)。

### 收集指标

`linkerd diagnostics` 命令可以直接从控制平面和数据平面的指标端点收集数据。要收集控制平面指标，请使用此命令：

```
$ linkerd diagnostics controller-metrics
```

```
#
# POD linkerd-destination-8498c6764f-96tqr (1 of 5)
# CONTAINER destination
#
# HELP cluster_store_size The number of linked clusters in the remote discove...
# TYPE cluster_store_size gauge
cluster_store_size 0
# HELP endpoint_profile_updates_queue_overflow A counter incremented whenever...
# TYPE endpoint_profile_updates_queue_overflow counter
endpoint_profile_updates_queue_overflow 0
# HELP endpoint_updates_queue_overflow A counter incremented whenever the end...
# TYPE endpoint_updates_queue_overflow counter
endpoint_updates_queue_overflow{service="kubernetes.default.svc.cluster.local...
# HELP endpoints_cache_size Number of items in the client-go endpoints cache
# TYPE endpoints_cache_size gauge
endpoints_cache_size{cluster="local"} 17
...
```

要收集给定代理或一组代理的指标数据，请使用以下命令：

```
$ linkerd diagnostics proxy-metrics -n emojivoto deploy/web
```

```
#
# POD web-5b97875957-xn269 (1 of 1)
#
# HELP inbound_http_authz_allow_total The total number of inbound HTTP reques...
# TYPE inbound_http_authz_allow_total counter
inbound_http_authz_allow_total{target_addr="0.0.0.0:4191",target_ip="0.0.0.0"...
inbound_http_authz_allow_total{target_addr="0.0.0.0:4191",target_ip="0.0.0.0"...
# HELP identity_cert_expiration_timestamp_seconds Time when the this proxy's ...
# TYPE identity_cert_expiration_timestamp_seconds gauge
identity_cert_expiration_timestamp_seconds 1705071458
# HELP identity_cert_refresh_count The total number of times this proxy's mTL...
# TYPE identity_cert_refresh_count counter
identity_cert_refresh_count 1
# HELP request_total Total count of HTTP requests.
# TYPE request_total counter
request_total{direction="inbound",target_addr="0.0.0.0:4191",target_ip="0.0.0...
request_total{direction="inbound",target_addr="0.0.0.0:4191",target_ip="0.0.0...
...
```

`linkerd diagnostics`生成原始的 Prometheus 指标，因此如果您使用这些命令，您需要已经知道您要查找的信息。此外，请注意由于空间原因，示例输出已被截断——这些命令生成的输出*比这里展示的要多得多*（通常是数百行或更多）。

### 检查端点

在 Linkerd 中最难调试的问题之一是当`linkerd2-proxy`发出表明其处于*failfast*状态的消息时。`failfast`状态在第十五章中有更详细的讨论，但落入`failfast`的一个非常常见的原因是某个服务没有任何有效的端点。您可以使用`linkerd diagnostics endpoints`检查这种情况。例如，在这里我们检查[emojivoto 示例应用程序](https://oreil.ly/ZnYsL)的`emoji-svc`服务的端点。

```
$ linkerd diagnostics endpoints emoji-svc.emojivoto.svc.cluster.local:8080
```

```
NAMESPACE   IP           PORT   POD                     SERVICE
emojivoto   10.42.0.15   8080   emoji-5b97875957-xn269  emoji-svc.emojivoto
```

请注意，您必须提供服务的完全合格的 DNS 名称以及端口号。如果找不到有效的端点，`linkerd diagnostics endpoints`将报告`未找到端点`，并且重要的是，请求将进入`failfast`状态。

### 诊断策略

从 Linkerd 2.13 开始，有一个新的`linkerd diagnostics policy`命令，可以提供有关 Linkerd 高级路由策略引擎的洞察。例如，您可以查看应用于`smiley`服务在`faces`命名空间端口 80 上的策略（如果您运行的是[Faces 演示应用程序](https://oreil.ly/a4OnB)）：

```
$ linkerd diagnostics policy -n faces svc/smiley 80 > smiley-diag.json
```

`linkerd diagnostics policy`的输出是*非常*详细的 JSON，因此几乎总是将其重定向到文件（如我们在这里所做的）或`less`、`bat`或类似工具中是一个好主意。您将看到`http1.1`、`http2`等各个部分，并且每个部分都将有一个非常详细的——再次强调，详细到令人叹为观止——策略细分。

举例来说，您可能会看到类似于示例 6-1 中的输出，描述未应用高级策略时 HTTP/2 流量的情况。

##### 示例 6-1\. 没有高级策略的 HTTP/2 输出块

```
http2:
  routes:
  - metadata:
      Kind:
        Default: http
    rules:
    - backends:
        Kind:
          FirstAvailable:
            backends:
            - backend:
                Kind:
                  Balancer:
                    Load:
                      PeakEwma:
                        decay:
                          seconds: 10
                        default_rtt:
                          nanos: 30000000
                    discovery:
                      Kind:
                        Dst:
                          path: smiley.faces.svc.cluster.local:80
                metadata:
                  Kind:
                    Default: service
                queue:
                  capacity: 100
                  failfast_timeout:
                    seconds: 3
      matches:
      - path:
          Kind:
            Prefix: /
```

或者，假设您将显示在示例 6-2 中的 HTTPRoute 资源应用于分割发送到`smiley`的流量，使一半的流量继续发送到`smiley`工作负载，另一半被重定向到`smiley2`。（HTTPRoutes 在第九章中有更详细的讨论。）

##### 示例 6-2\. HTTPRoute 流量分割

```
apiVersion: policy.linkerd.io/v1beta3
kind: HTTPRoute
metadata:
  name: smiley-split
  namespace: faces
spec:
  parentRefs:
    - name: smiley
      kind: Service
      group: core
      port: 80
  rules:
  - backendRefs:
    - name: smiley
      port: 80
      weight: 50
    - name: smiley2
      port: 80
      weight: 50
```

在生效的 HTTPRoute 条件下，`linkerd diagnostics policy`可能生成一个`http2`块，如示例 6-3 所示，显示确实正在分割流量。

##### 示例 6-3\. 带有流量分割的 HTTP/2 输出块

```
http2:
  routes:
  - metadata:
      Kind:
        Resource:
          group: policy.linkerd.io
          kind: HTTPRoute
          name: smiley-split
          namespace: faces
    rules:
    - backends:
        Kind:
          RandomAvailable:
            backends:
            - backend:
                backend:
                  Kind:
                    Balancer:
                      Load:
                        PeakEwma:
                          decay:
                            seconds: 10
                          default_rtt:
                            nanos: 30000000
                      discovery:
                        Kind:
                          Dst:
                            path: smiley.faces.svc.cluster.local:80
                  metadata:
                    Kind:
                      Resource:
                        group: core
                        kind: Service
                        name: smiley
                        namespace: faces
                        port: 80
                  queue:
                    capacity: 100
                    failfast_timeout:
                      seconds: 3
              weight: 50
            - backend:
                backend:
                  Kind:
                    Balancer:
                      Load:
                        PeakEwma:
                          decay:
                            seconds: 10
                          default_rtt:
                            nanos: 30000000
                      discovery:
                        Kind:
                          Dst:
                            path: smiley2.faces.svc.cluster.local:80
                  metadata:
                    Kind:
                      Resource:
                        group: core
                        kind: Service
                        name: smiley2
                        namespace: faces
                        port: 80
                  queue:
                    capacity: 100
                    failfast_timeout:
                      seconds: 3
              weight: 50
      matches:
      - path:
          Kind:
            Prefix: /
```

随着 Linkerd 的发展，这些输出会发生变化，因此要对这些示例持保留态度。`linkerd diagnostics policy` 的目的是提供足够的细节，使您能够理解 Linkerd 如何管理特定工作负载的流量，无论源代码如何更改。

# 概要

`linkerd` CLI 不仅提供安装 Linkerd 所需的工具，还提供关键的运维工具，简化了在您的集群中运行 Linkerd 的流程。虽然完全可以使用 Linkerd 而不运行 `linkerd` CLI，但 CLI 是处理许多现实场景的最直接、最有效的方式。
