# 第十八章：从常见编程语言访问 Kubernetes

尽管本书大部分内容都致力于使用声明性 YAML 配置，直接通过 `kubectl` 或者像 Helm 这样的工具，但有些情况下必须直接从编程语言与 Kubernetes API 进行交互。例如，[Helm 工具](https://helm.sh) 的作者们就需要用编程语言来编写该应用程序。更普遍地说，如果你需要编写额外的工具，比如 `kubectl` 插件，或者像 Kubernetes 运算符这样的更复杂的代码，这是很常见的情况。

Kubernetes 生态系统的大部分内容都是用 Go 编程语言编写的。因此，Go 语言拥有最丰富和最广泛的客户端。不过，对于大多数常见的编程语言（甚至一些不常见的语言），都有高质量的客户端。由于已经有了大量关于如何使用 Go 客户端的文档和示例，本章将涵盖使用 Python、Java 和 .NET 示例与 Kubernetes API 服务器进行交互的基础知识。

# Kubernetes API：客户端视角

归根结底，Kubernetes API 服务器只是一个 HTTP(S) 服务器，每个客户端库都会以此方式来看待它，尽管每个客户端都有大量的附加逻辑来实现各种 API 调用，并且在 JSON 序列化和反序列化方面。因此，你可能会倾向于简单地使用普通的 HTTP 客户端来处理 Kubernetes API，但客户端库会将这些不同的 HTTP 调用封装成有意义的 API（例如 `readNamespacedPod(...)`），以使你的代码更易读，并提供有意义的类型化对象模型，从而促进静态类型检查，减少错误（例如 `Deployment`）。也许更重要的是，客户端库还实现了 Kubernetes 特定的功能，如从 *kubeconfig* 文件或 Pod 环境中加载授权信息。客户端还提供了 Kubernetes API 表面的非 RESTful 部分的实现，如端口转发、日志和监听功能。我们将在后续章节中描述这些高级功能。

## OpenAPI 和生成的客户端库

Kubernetes API 中的资源和功能集合非常庞大。不同的 API 组中有许多不同的资源，以及每个资源上的许多不同操作。如果开发者需要手工编写所有这些 API 调用，那将是一个非常庞大（并且无疑令人枯燥）的任务。特别是考虑到客户端必须在各种编程语言中手工编写。相反，客户端采用了不同的方法，与 Kubernetes API 服务器进行交互的基础都是由一个类似于反向编译器的计算机程序生成的。API 客户端的代码生成器采用 Kubernetes API 的数据规范，并使用这些规范为特定语言生成客户端。

Kubernetes API 采用称为 OpenAPI 的格式表示，这是表示 RESTful API 的最常见模式。为了让你感受到 Kubernetes API 的规模，GitHub 上的 [OpenAPI 规范](https://oreil.ly/3gRIW) 文件大小超过四兆字节。这是一个相当大的文本文件！所有官方的 Kubernetes 客户端库都是使用相同的核心代码生成逻辑生成的，你可以在 [GitHub](https://oreil.ly/F39uK) 上找到这些逻辑。虽然你不太可能需要自己生成客户端库，但了解生成这些库的过程仍然是有用的。特别是因为大部分客户端代码是生成的，因此无法直接在生成的客户端代码中进行更新和修复，因为下次生成 API 时会被覆盖。因此，当发现客户端中的错误时，需要修复 OpenAPI 规范（如果错误在规范本身中）或代码生成器（如果错误在生成的代码中）。尽管这个过程看起来过于复杂，但这是少数 Kubernetes 客户端作者能够跟上 Kubernetes API 广度的唯一方式。

## 但是 `kubectl x` 呢？

当你开始实现自己的逻辑与 Kubernetes API 交互时，可能不久就会想知道如何执行 `kubectl x`。大多数人在学习 Kubernetes 时都是从 `kubectl` 工具开始的，因此他们预期 `kubectl` 和 Kubernetes API 之间有一对一的映射关系。虽然某些命令在 Kubernetes API 中直接表示（例如 `kubectl get pods`），但大多数更复杂的功能实际上是通过多个具有复杂逻辑的 API 调用在 `kubectl` 工具中实现的。

自 Kubernetes 起初以来，客户端和服务器端功能之间的平衡一直是一个设计折衷。现在在 API 服务器上实现的许多功能最初是在 `kubectl` 中作为客户端实现的。例如，现在由 Deployment 资源在服务器上实现的发布功能，以前是在客户端上实现的。同样，直到最近，`kubectl apply ...` 只能在命令行工具内使用，但已迁移到服务器作为服务器端的 `apply` 功能，将在本章后面讨论。

尽管总体上朝向服务器端实现发展，仍然有一些显著功能留在客户端。这些功能必须在每个客户端库中重新实现。不同语言之间的与 `kubectl` 命令行工具的兼容性也各不相同。特别是 Java 客户端已经构建了一个模拟大部分 `kubectl` 功能的厚客户端。

如果在您的客户端库中找不到所需的功能，一个有用的技巧是在您的 `kubectl` 命令中添加 `--v=10` 标志。这将打开详细日志记录，包括发送到 Kubernetes API 服务器的所有 HTTP 请求和响应。您可以使用此日志来重构 `kubectl` 所做的大部分工作。如果您仍然需要深入挖掘，`kubectl` 的源代码也可以在 Kubernetes 仓库中找到。

# 编程 Kubernetes API

现在您对 Kubernetes API 如何工作以及客户端和服务器如何交互有了更深入的理解。在接下来的章节中，我们将介绍如何对 Kubernetes API 服务器进行认证并与资源进行交互。最后，我们将涉及从编写运算符到与 Pod 进行交互操作的高级主题。

## 安装客户端库

在开始使用 Kubernetes API 进行编程之前，您需要找到客户端库。我们将使用 Kubernetes 项目本身生产的官方客户端库，尽管也有许多作为独立项目开发的高质量客户端。这些客户端库都托管在 GitHub 上的 kubernetes-client 仓库下：

+   [Python](https://oreil.ly/ku6mT)

+   [Java](https://oreil.ly/aUSkD)

+   [.NET](https://oreil.ly/9J8iy)

这些项目中的每一个都具有兼容性矩阵，显示客户端的哪些版本适用于 Kubernetes API 的哪些版本，并提供使用特定编程语言的软件包管理器（例如 `npm`）安装库的说明。^(1)

## 认证到 Kubernetes API

如果允许世界上任何人访问 Kubernetes API 并读取或写入其编排的资源，那么 Kubernetes API 服务器将不会很安全。因此，编程访问 Kubernetes API 的第一步是连接到 API 并进行身份验证。由于 API 服务器在其核心是一个 HTTP 服务器，所以这些身份验证方法是核心的 HTTP 身份验证方法。最初的 Kubernetes 实现使用基本的 HTTP 身份验证通过用户和密码组合进行身份验证，但是这种方法已被更现代的身份验证基础设施所取代。

如果您一直使用 `kubectl` 命令行工具与 Kubernetes 交互，可能未考虑过认证的实现细节。幸运的是，客户端库通常使连接到 API 变得简单。但是，了解 Kubernetes 认证的基本工作原理仍然有助于在出现问题时进行调试。

`kubectl` 工具和客户端获取认证信息的两种基本方式：来自 kubeconfig 文件和来自 Kubernetes 集群中 Pod 的上下文。

不在 Kubernetes 集群内运行的代码需要一个 kubeconfig 文件来提供认证所需的信息。默认情况下，客户端会在 *${HOME}/.kube/config* 或 `$KUBECONFIG` 环境变量指定的位置搜索此文件。如果存在 `KUBECONFIG` 变量，则优先于默认的主目录位置中的任何配置文件。kubeconfig 文件包含访问 Kubernetes API 服务器所需的所有信息。客户端都有易于使用的调用方式，可以从默认位置或代码中提供的 kubeconfig 文件创建客户端：

*Python*

```
config.load_kube_config()
```

*Java*

```
ApiClient client = Config.defaultClient();
Configuration.setDefaultApiClient(client);
```

*.NET*

```
var config = KubernetesClientConfiguration.BuildDefaultConfig();
var client = new Kubernetes(config);
```

###### 注意

许多云提供商的认证通过一个外部可执行文件完成，该文件知道如何为 Kubernetes 集群生成令牌。此可执行文件通常作为云提供商命令行工具的一部分安装。当您编写与 Kubernetes API 交互的代码时，需要确保在代码运行的上下文中也可执行此可执行文件以获取令牌。

在 Kubernetes 集群中 Pod 的上下文中，运行在 Pod 中的代码可以访问与该 Pod 关联的 Kubernetes 服务账户。包含相关令牌和证书授权的文件在创建 Pod 时由 Kubernetes 作为卷放置在 Pod 中。在 Kubernetes 集群中，API 服务器始终位于固定的 DNS 名称下，通常为 `kubernetes`。因为 Pod 中存在所有必要的数据，所以不需要 kubeconfig 文件，客户端可以从其上下文中合成其配置。客户端都有易于使用的调用方式来创建这样的“集群内”客户端：

*Python*

```
config.load_incluster_config()
```

*Java*

```
ApiClient client = ClientBuilder.cluster().build();
Configuration.setDefaultApiClient(client);
```

*.NET*

```
var config = KubernetesClientConfiguration.InClusterConfig()
var client = new Kubernetes(config);
```

###### 注意

与 Pod 相关联的默认服务账户被授予了最低的角色（RBAC）。这意味着，默认情况下，运行在 Pod 中的代码对 Kubernetes API 的操作有限。如果你遇到授权错误，可能需要调整服务账户，选择一个特定于你的代码并具有集群中必要角色权限的账户。

## 访问 Kubernetes API

人们与 Kubernetes API 交互的最常见方式是通过基本操作，如创建、列出和删除资源。因为所有客户端都是从相同的 OpenAPI 规范生成的，它们都遵循相同的大致模式。在深入代码之前，还有几个关于 Kubernetes API 的细节是必须理解的。

在 Kubernetes 中，有命名空间和集群级别的资源区别。*命名空间* 资源存在于 Kubernetes 命名空间内；例如，Pod 或 Deployment 可能存在于 `kube-system` 命名空间中。*集群级别* 资源则只存在于整个集群中的一个实例。最明显的例子是 Namespace，但其他集群级别的资源还包括 CustomResourceDefinitions 和 ClusterRoleBindings。这种区别很重要，因为它在你访问资源时的函数调用中得以体现。例如，要在 Python 中列出 `default` 命名空间中的 Pods，你需要编写 `api.list_namespaced_pods('default')`。要列出命名空间，你需要编写 `api.list_namespaces()`。

第二个你需要理解的概念是*API 组*。在 Kubernetes 中，所有资源都被分组到不同的 API 集合中。尽管使用 `kubectl` 工具的用户可能不太会注意到这一点，但你可能在 Kubernetes 对象的 YAML 规范的 `apiVersion` 字段中看到过。当针对 Kubernetes API 进行编程时，这种分组变得很重要，因为通常每个 API 组都有其自己的客户端来与该组资源进行交互。例如，要创建一个用于与 Deployment 资源交互的客户端（该资源存在于 `apps/v1` API 组和版本中），你需要创建一个 `new AppsV1Api()` 对象，它知道如何与 `apps/v1` API 组和版本中的所有资源进行交互。如何为 API 组创建客户端的示例将在以下部分展示。

## 将所有内容汇总：在 Python、Java 和 .NET 中列出和创建 Pods

现在我们已经准备好实际编写一些代码了。首先创建一个客户端对象，然后使用它来列出“default”命名空间中的 Pods；以下是在 Python、Java 和 .NET 中实现的代码：

*Python*

```
config.load_kube_config()
api = client.CoreV1Api()
pod_list = api.list_namespaced_pod('default')
```

*Java*

```
ApiClient client = Config.defaultClient();
Configuration.setDefaultApiClient(client);
CoreV1Api api = new CoreV1Api();
V1PodList list = api.listNamespacedPod("default");
```

*.NET*

```
var config = KubernetesClientConfiguration.BuildDefaultConfig();
var client = new Kubernetes(config);
var list = client.ListNamespacedPod("default");
```

一旦你弄清楚如何列出、读取和删除对象，下一个常见任务就是创建新对象。调用 API 来创建对象是相对容易理解的（例如，在 Python 中使用 `create_namespaced_pod`），但实际定义新 Pod 资源可能更加复杂。

下面是在 Python、Java 和 .NET 中创建 Pod 的方法：

*Python*

```
container = client.V1Container(
     name="myapp",
     image="my_cool_image:v1",
 )

pod = client.V1Pod(
    metadata = client.V1ObjectMeta(
      name="myapp",
    ),
    spec=client.V1PodSpec(containers=[container]),
)
```

*Java*

```
V1Pod pod =
    new V1PodBuilder()
        .withNewMetadata().withName("myapp").endMetadata()
        .withNewSpec()
          .addNewContainer()
            .withName("myapp")
            .withImage("my_cool_image:v1")
          .endContainer()
        .endSpec()
        .build();
```

*.NET*

```
var pod = new V1Pod()
{
    Metadata = new V1ObjectMeta{ Name = "myapp", },
    Spec = new V1PodSpec
    {
        Containers = new[] {
          new V1Container() {
            Name = "myapp", Image = "my_cool_image:v1",
          },
        },
    }
 };
```

## 创建和修补对象

当你探索 Kubernetes 的客户端 API 时，你会注意到似乎有三种不同的方法来操作资源，分别是`create`、`replace`和`patch`。这三个动词代表着与资源交互的略微不同的语义：

Create

正如名称所示，这将创建一个新的资源。但是，如果资源已经存在，则会失败。

Replace

这将完全替换现有资源，而不查看现有资源。当使用`replace`时，必须指定完整的资源。

Patch

这修改了现有资源，未改变资源的未更改部分。在使用`patch`时，您使用特殊的 Patch 资源而不是发送要修改的资源（例如 Pod）。

###### 注意

对资源进行修补可能会很复杂。在许多情况下，直接替换会更容易。然而，在某些情况下，特别是对于大型资源，修补资源在网络带宽和 API 服务器处理方面可能更高效。此外，多个操作者可以同时修补资源的不同部分，而无需担心写冲突，这减少了开销。

要修补 Kubernetes 资源，必须创建一个表示要对资源进行的更改的 Patch 对象。Kubernetes 支持三种此补丁的格式：JSON Patch、JSON Merge Patch 和策略性合并补丁。前两种补丁格式是其他地方使用的 RFC 标准，第三种是 Kubernetes 开发的补丁格式。每种补丁格式都有优缺点。在这些示例中，我们将使用 JSON Patch，因为它最简单易懂。

下面是如何将 Deployment 修补以增加副本数到三个的方法：

*Python*

```
deployment.spec.replicas = 3

api_response = api_instance.patch_namespaced_deployment(
    name="my-deployment",
    namespace="some-namespace",
    body=deployment)
```

*Java*

```
// JSON-patch format
static String jsonPatch =
  "[{\"op\":\"replace\",\"path\":\"/spec/replicas\",\"value\":3}]";

V1Deployment patched =
          PatchUtils.patch(
              V1Deployment.class,
              () ->
                  api.patchNamespacedDeploymentCall(
                      "my-deployment",
                      "some-namespace",
                      new V1Patch(jsonPatchStr),
                      null,
                      null,
                      null,
                      null,
                      null),
              V1Patch.PATCH_FORMAT_JSON_PATCH,
              api.getApiClient());
```

*.NET*

```
var jsonPatch = @"
[{
 ""op"": ""replace"",
 ""path"": ""/spec/replicas"",
 ""value"": 3
}]";

client.PatchNamespacedPod(
  new V1Patch(patchStr, V1Patch.PatchType.JsonPatch),
  "my-deployment",
  "some-namespace");
```

在这些代码示例中，Deployment 资源已被修补以将部署中的副本数设置为三。

## 监视 Kubernetes API 的变化

Kubernetes 中的资源是声明性的。它们代表系统的期望状态。为了使这个期望的状态成为现实，程序必须监视期望的状态以进行更改，并采取行动使世界的当前状态与期望的状态匹配。

由于这种模式，针对 Kubernetes API 编程时最常见的任务之一是监视资源的变化，然后根据这些变化采取某些操作。最简单的方法是通过轮询来实现。*轮询* 简单地在固定的间隔（如每 60 秒）调用上述列出资源的函数，并枚举代码感兴趣的所有资源。虽然这种代码编写起来很容易，但对客户端代码和 API 服务器都有很多缺点。轮询引入不必要的延迟，因为等待轮询周期导致在上一次轮询完成后发生的更改存在延迟。此外，轮询会导致 API 服务器负载加重，因为它反复返回未更改的资源。虽然许多简单的客户端开始使用轮询，但太多客户端轮询 API 服务器可能会使其超载并增加延迟。

为解决这个问题，Kubernetes API 还提供了 *watch* 或基于事件的语义。使用 `watch` 调用，您可以向 API 服务器注册对特定更改的兴趣，而不是重复轮询，API 服务器会在发生更改时发送通知。在实际操作中，客户端执行一个持续的 GET 到 HTTP API 服务器。支持这个 HTTP 请求的 TCP 连接在整个 watch 期间保持打开状态，服务器在每次更改时向该流写入响应（但不关闭流）。

从程序角度来看，`watch` 语义支持基于事件的编程，将重复轮询的 `while` 循环改为一组回调函数。以下是监视 Pod 变化的示例：

*Python*

```
config.load_kube_config()
api = client.CoreV1Api()
w = watch.Watch()

for event in w.stream(v1.list_namespaced_pods, "some-namespace"):
  print(event)
```

*Java*

```
    ApiClient client = Config.defaultClient();
    CoreV1Api api = new CoreV1Api();

    Watch<V1Namespace> watch =
        Watch.createWatch(
            client,
            api.listNamespacedPodCall(
                "some-namespace",
                null,
                null,
                null,
                null,
                null,
                Integer.MAX_VALUE,
                null,
                null,
                60,
                Boolean.TRUE);
            new TypeToken<Watch.Response<V1Pod>>() {}.getType());

    try {
      for (Watch.Response<V1Pod> item : watch) {
        System.out.printf(
          "%s : %s%n", item.type, item.object.getMetadata().getName());
      }
    } finally {
      watch.close();
    }
```

*.NET*

```
var config = KubernetesClientConfiguration.BuildConfigFromConfigFile();
var client = new Kubernetes(config);

var watch =
  client.ListNamespacedPodWithHttpMessagesAsync("default", watch: true);
using (watch.Watch<V1Pod, V1PodList>((type, item) =>
{
  Console.WriteLine(item);
}
```

在这些示例中，与重复轮询循环不同，`watch` API 调用将每个资源的每个变化传递给用户提供的回调函数。这既减少了延迟，也减少了 Kubernetes API 服务器的负载。

## 与 Pod 交互

Kubernetes API 还提供了直接与运行在 Kubernetes Pod 中的应用程序进行交互的函数。`kubectl` 工具提供了许多与 Pod 交互的命令，包括 `logs`、`exec` 和 `port-forward`，也可以从自定义代码中使用这些命令。

###### 注意

由于 `logs`、`exec` 和 `port-forward` API 在 RESTful 视角上是非标准的，它们在客户端库中需要定制逻辑，因此在不同的客户端之间可能不太一致。不幸的是，除了学习每种语言的实现之外，没有其他选择。

当获取 Pod 的日志时，您必须决定是读取 Pod 日志以获取其当前状态的快照，还是将其流式传输以在发生时接收新的日志。如果流式传输日志（相当于 `kubectl logs -f ...`），则会创建到 API 服务器的开放连接，并且新的日志行将像写入 Pod 一样写入到此流中。否则，您只会接收日志的当前内容。

这里是如何同时读取和流式传输日志的方法：

*Python*

```
config.load_kube_config()
api = client.CoreV1Api()
log = api_instance.read_namespaced_pod_log(
  name="my-pod", namespace="some-namespace")
```

*Java*

```
V1Pod pod = ...; // some code to define or get a Pod here
PodLogs logs = new PodLogs();
InputStream is = logs.streamNamespacedPodLog(pod);
```

*.NET*

```
IKubernetes client = new Kubernetes(config);
var response = await client.ReadNamespacedPodLogWithHttpMessagesAsync(
    "my-pod", "my-namespace", follow: true);
var stream = response.Body;
```

另一个常见的任务是在 Pod 中执行某个命令并获取运行该任务的输出。您可以在命令行上使用 `kubectl exec ...` 命令。在幕后，实现此功能的 API 正在创建到 API 服务器的 WebSocket 连接。WebSocket 可以在同一 HTTP 连接上同时存在多个数据流（在本例中是 `stdin`、`stdout` 和 `stderr`）。如果您从未使用过 WebSocket，不用担心；客户端库会处理与 WebSocket 的交互细节。

这里是如何在 Pod 中执行 `ls /foo` 命令的方法：

*Python*

```
cmd = [ 'ls', '/foo' ]
response = stream(
    api_instance.connect_get_namespaced_pod_exec,
    "my-pod",
    "some-namespace",
    command=cmd,
    stderr=True,
    stdin=False,
    stdout=True,
    tty=False)
```

*Java*

```
ApiClient client = Config.defaultClient();
Configuration.setDefaultApiClient(client);
Exec exec = new Exec();
final Process proc =
  exec.exec("some-namespace",
            "my-pod",
            new String[] {"ls", "/foo"},
            true,
            true /*tty*/);
```

*.NET*

```
var config = KubernetesClientConfiguration.BuildConfigFromConfigFile();
IKubernetes client = new Kubernetes(config);
var webSocket =
    await client.WebSocketNamespacedPodExecAsync(
      "my-pod", "some-namespace", "ls /foo", "my-container-name");
var demux = new StreamDemuxer(webSocket);
demux.Start();
var stream = demux.GetStream(1, 1);
```

除了在 Pod 中运行命令外，您还可以将网络连接从 Pod 端口转发到运行在本地机器上的代码。与 `exec` 类似，端口转发的流量通过 WebSocket 进行。由您的代码决定如何处理此端口转发的套接字。您可以简单地发送一个请求并作为字节字符串接收响应，或者您可以构建一个完整的代理服务器（类似于 `kubectl port-forward`），通过此代理处理任意请求。

无论您打算如何处理连接，这里是如何设置端口转发的方法：

*Python*

```
pf = portforward(
    api_instance.connect_get_namespaced_pod_portforward,
    'my-pod', 'some-namespace',
    ports='8080',
)
```

*Java*

```
PortForward fwd = new PortForward();

List<Integer> ports = new ArrayList<>();
int localPort = 8080;
int targetPort = 8080;
ports.add(targetPort);
final PortForward.PortForwardResult result =
    fwd.forward("some-namespace", "my-pod", ports);
```

*.NET*

```
var config = KubernetesClientConfiguration.BuildConfigFromConfigFile();
IKubernetes client = new Kubernetes(config);
var webSocket = await client.WebSocketNamespacedPodPortForwardAsync(
  "some-namespace", "my-pod", new int[] {8080}, "v4.channel.k8s.io");
var demux = new StreamDemuxer(webSocket, StreamType.PortForward);
demux.Start();
var stream = demux.GetStream((byte?)0, (byte?)0);
```

每个示例都在 Pod 中的端口 8080 上创建到程序端口 8080 的连接。该代码返回必要的字节流，通过这种端口转发通道进行通信。您可以使用这些流来发送和接收消息。

## 概要

Kubernetes API 提供了丰富且强大的功能，让您编写自定义代码。将您的应用程序编写成最适合任务或个人角色的语言，与尽可能多的 Kubernetes 用户分享编排 API 的功能。当您准备好超越对 `kubectl` 可执行文件的脚本调用时，Kubernetes 客户端库提供了一种方式，深入 API 构建操作员、监控代理、新用户界面或您的想象力可以梦想到的任何东西。

^(1) 为了简洁起见，我们没有包含 [JavaScript](https://oreil.ly/8mw5F) 示例，但它也在积极开发中。
