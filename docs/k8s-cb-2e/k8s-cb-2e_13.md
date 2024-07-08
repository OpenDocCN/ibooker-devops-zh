# 第十三章：服务网格

本章重点介绍了使在 Kubernetes 上开发分布式、基于微服务的应用更加轻松的构建块之一：服务网格。像 Istio 和 Linkerd 这样的服务网格可以执行监控、服务发现、流量控制和安全等任务。通过将这些责任交给网格，应用开发者可以专注于提供附加值，而不是通过解决横向基础设施问题来重新发明轮子。

服务网格的一个主要优点是，它们可以在不需要服务（客户端和服务器）知道自己是服务网格的一部分的情况下，透明地对服务应用策略。

在本章中，我们将使用 Istio 和 Linkerd 分别进行基本示例。对于每个服务网格，我们将展示如何使用 Minikube 快速启动并实现网格内服务到服务的通信，同时使用简单但有说明性的服务网格策略。在这两个示例中，我们将部署一个基于 NGINX 的服务，并且我们的客户端调用服务将是一个 `curl` pod。这两者都将被添加到网格中，并且服务之间的交互将由网格管理。

# 13.1 安装 Istio 服务网格

## 问题

您的组织正在使用或计划使用微服务架构，并且希望通过卸载构建安全性、服务发现、遥测、部署策略等非功能性关注点的需要来减轻开发者的负担。

## 解决方案

在 Minikube 上安装 Istio。Istio 是目前最广泛采用的服务网格，可以减轻许多微服务开发者的责任，同时为运维人员提供集中式的安全与运维治理。

首先，您需要启动 Minikube，并为运行 Istio 分配足够的资源。确切的资源需求取决于您的平台，您可能需要调整资源分配。我们已经成功地使用了将近 8 GB 的内存和四个 CPU：

```
$ minikube start --memory=7851 --cpus=4

```

您可以使用 Minikube 隧道作为 Istio 的负载均衡器。要启动它，请在新的终端中运行以下命令（它将锁定终端以显示输出信息）：

```
$ minikube tunnel

```

使用以下命令下载并解压最新版本的 Istio（适用于 Linux 和 macOS）：

```
$ curl -L https://istio.io/downloadIstio | sh -

```

对于 Windows，你可以使用 `choco` 进行安装，或者直接从可下载的归档中提取 *.exe*。获取 Istio 的更多信息，请查阅 Istio 的[入门指南](https://oreil.ly/5uFlk)。

切换到 Istio 目录。您可能需要根据安装的 Istio 版本适应目录名称：

```
$ cd istio-1.18.0

```

`istioctl` 命令行工具旨在帮助调试和诊断您的服务网格，您将在其他示例中使用它来检查您的 Istio 配置。它位于 *bin* 目录中，请像这样将其添加到您的路径中：

```
$ export PATH=$PWD/bin:$PATH

```

现在你可以安装 Istio 了。以下 YAML 文件包含一个示例演示配置。它有意禁用了将 Istio 作为入口或出口网关使用，因为我们这里不会使用 Istio 作为入口。将此配置保存在名为 *istio-demo-config.yaml* 的文件中：

```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: demo
  components:
    ingressGateways:
    - name: istio-ingressgateway
      enabled: false
    egressGateways:
    - name: istio-egressgateway
      enabled: false

```

现在使用 `istioctl` 将此配置应用到 Minikube：

```
$ istioctl install -f istio-demo-config.yaml -y
✔ Istio core installed
✔ Istiod installed
✔ Installation complete

```

最后，请确保 Istio 配置为自动注入 Envoy sidecar 代理到您部署的服务中。您可以使用以下命令为默认命名空间启用此功能：

```
$ kubectl label namespace default istio-injection=enabled
namespace/default labeled

```

## 讨论

本指南使用底层项目（如 Kubernetes 和 Istio）的默认（有时意味着最新）版本。

您可以根据您当前生产环境的版本自定义这些版本。例如，要设置您想要使用的 Istio 版本，请在下载 Istio 时使用 `ISTIO_VERSION` 和 `TARGET_ARCH` 参数。例如：

```
$ curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.18.0 \
    TARGET_ARCH=x86_64 sh -

```

## 参见

+   官方 Istio [入门指南](https://oreil.ly/AKCYs)

# 13.2 部署带有 Istio sidecar 的微服务

## 问题

您想将新服务部署到服务网格中，这意味着应自动向服务的 pod 注入 sidecar。sidecar 将拦截所有服务的进出流量，并允许实施路由、安全和监控策略（等等），而无需修改服务本身的实现。

## 解决方案

我们将使用 NGINX 作为一个简单的服务进行工作。首先创建一个 NGINX 的部署：

```
$ kubectl create deployment nginx --image nginx:1.25.2
deployment.apps/nginx created

```

然后将其暴露为 Kubernetes 服务：

```
$ kubectl expose deploy/nginx --port 80
service/nginx exposed

```

###### 注意

Istio 不会在 Kubernetes 上创建新的 DNS 条目，而是依赖于 Kubernetes 或您可能正在使用的任何其他服务注册表中注册的现有服务。在本章后面，您将部署一个 `curl` pod，该 pod 调用 `nginx` 服务，并将 `curl` 主机设置为 `nginx` 以进行 DNS 解析，但随后 Istio 将通过拦截请求并允许您定义额外的流量控制策略来发挥其魔力。

现在列出默认命名空间中的 pods。服务的 pod 中应该有两个容器：

```
$ kubectl get po
NAME                          READY   STATUS    RESTARTS   AGE
nginx-77b4fdf86c-kzqvt        2/2     Running   0          27s

```

如果您调查此 pod 的详细信息，您会发现 Istio sidecar 容器（基于 Envoy 代理）已注入到 pod 中：

```
$ kubectl get pods -l app=nginx -o yaml
apiVersion: v1
items:
- apiVersion: v1
  kind: Pod
  metadata:
    labels:
      app: nginx
      ...
  spec:
    containers:
    - image: nginx:1.25.2
      imagePullPolicy: IfNotPresent
      name: nginx
      resources: {}

    ...
kind: List
metadata:
  resourceVersion: ""

```

## 讨论

本文假设您已经使用命名空间标记技术在命名空间中启用了自动 sidecar 注入，如 Recipe 13.1 所示。但是，并非必须将 sidecar 注入到命名空间中的每个 pod 中。在这种情况下，您可以手动选择应包含 sidecar 并因此添加到网格中的 pods。您可以在 [官方 Istio 文档](https://oreil.ly/VbHz_) 中了解有关手动 sidecar 注入的更多信息。

## 参见

+   有关如何 [安装和配置 sidecar](https://oreil.ly/E-omC) 的更多信息

+   关于 [Istio 中 sidecar 的角色](https://oreil.ly/TperP) 的更多信息

# 13.3 使用 Istio 虚拟服务进行流量路由

## 问题

你想在集群中部署另一个服务，该服务将调用之前部署的`nginx`服务，但你不想在服务本身中编写任何路由或安全逻辑。你还希望尽可能地解耦客户端和服务器。

## 解决方案

我们将通过部署一个将被添加到 mesh 中并调用`nginx`服务的`curl` pod 来模拟服务网格内的服务间通信。

为了将`curl` pod 与运行`nginx`的特定 pod 解耦，你将创建一个 Istio 虚拟服务。`curl` pod 只需要知道虚拟服务即可。Istio 及其 sidecars 将拦截并路由从客户端到服务的流量。

在名为*virtualservice.yaml*的文件中创建以下虚拟服务规范：

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx-vs
spec:
  hosts:
  - nginx
  http:
  - route:
    - destination:
        host: nginx

```

创建虚拟服务：

```
$ kubectl apply -f virtualservice.yaml

```

然后运行一个`curl` pod，你将用它来调用该服务。因为你已经在`default`命名空间中部署了`curl` pod，并且在该命名空间中激活了自动注入 sidecar（Recipe 13.1），`curl` pod 将自动获得一个 sidecar 并添加到 mesh 中：

```
$ kubectl run mycurlpod --image=curlimages/curl -i --tty -- sh

```

###### 注意

如果你意外退出了`curl` pod 的 shell，你可以使用`kubectl exec`命令再次进入该 pod：

```
$ kubectl exec -i --tty mycurlpod -- sh

```

现在你可以从`curl` pod 中调用`nginx`虚拟服务：

```
$ curl -v nginx
*   Trying 10.152.183.90:80...
* Connected to nginx (10.152.183.90) port 80 (#0)
> GET / HTTP/1.1
> Host: nginx
> User-Agent: curl/8.1.2
> Accept: */*
>
> HTTP/1.1 200 OK
> server: envoy
...

```

你将看到来自`nginx`服务的响应，但请注意 HTTP 头`server: envoy`表明响应实际上来自运行在`nginx` pod 中的 Istio sidecar。

###### 注意

为了从`curl`引用虚拟服务，我们使用引用 Kubernetes 服务名称（在本例中为`nginx`）的简短名称。在幕后，这些名称被转换为完全限定域名，如`nginx.default.svc.cluster.local`。正如你所见，完全限定名称包括命名空间名称（在本例中为`default`）。为了安全起见，在生产用例中建议你明确使用完全限定名称以避免配置错误。

## 讨论

本文重点介绍了服务网格内的服务间通信（也称为*东西向通信*），这是该技术的亮点。然而，Istio 和其他服务网格也能够执行网关职责（也称为*入口*和*南北向通信*），例如在网格外（或 Kubernetes 集群外）运行的客户端与网格内运行的服务之间的交互。

在撰写本文时，Istio 的网关资源正在逐渐被新的[Kubernetes Gateway API](https://gateway-api.sigs.k8s.io)所取代。

## 参见

+   [Istio 虚拟服务](https://oreil.ly/Lth6l)的官方参考文档。

+   了解更多关于 Kubernetes Gateway API 预计将[取代 Istio 的 Gateway](https://oreil.ly/6vHQv)的信息。

# 13.4 使用 Istio 虚拟服务重写 URL

## 问题

一个旧版客户端正在使用一个不再有效的服务 URL 和路径。您希望动态重写路径，以便正确调用该服务，而无需更改客户端。

您可以通过在`curl`容器中调用*/legacypath*路径来模拟此问题，这将产生 404 Not Found 响应：

```
$ curl -v nginx/legacypath
*   Trying 10.152.183.90:80...
* Connected to nginx (10.152.183.90) port 80 (#0)
> GET /legacypath HTTP/1.1
> Host: nginx
> User-Agent: curl/8.1.2
> Accept: */*
>
< HTTP/1.1 404 Not Found
< server: envoy
< date: Mon, 26 Jun 2023 09:37:43 GMT
< content-type: text/html
< content-length: 153
< x-envoy-upstream-service-time: 20
<
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.25.1</center>
</body>
</html>

```

## 解决方案

使用 Istio 重写旧路径，以便它达到服务的有效端点，在我们的示例中将是`nginx`服务的根目录。

更新虚拟服务以包含 HTTP 重写：

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: nginx-vs
spec:
  hosts:
  - nginx
  http:
  - match:
    - uri:
        prefix: /legacypath
    rewrite:
      uri: /
    route:
    - destination:
        host: nginx
  - route:
    - destination:
        host: nginx

```

然后应用更改：

```
$ kubectl apply -f virtualservice.yaml

```

更新后的虚拟服务包含一个`match`属性，该属性将查找旧路径并将其重写为简单地指向根端点。

现在，从`curl`容器调用旧路径将不再产生 404 错误，而是产生 200 OK 响应：

```
$ curl -v nginx/legacypath
*   Trying 10.152.183.90:80...
* Connected to nginx (10.152.183.90) port 80 (#0)
> GET /legacypath HTTP/1.1
> Host: nginx
> User-Agent: curl/8.1.2
> Accept: */*
>
< HTTP/1.1 200 OK

```

## 讨论

虚拟服务的主要作用是定义从客户端到上游服务的路由。如需对上游服务的请求有额外的控制，请参考[Istio 关于目标规则的文档](https://oreil.ly/Yu4xW)。

## 参见

+   [Istio `HTTPRewrite`文档](https://oreil.ly/EGAFs)

# 13.5 安装 Linkerd 服务网格

## 问题

您的项目需要小型占地面积和/或不需要 Istio 提供的所有功能，例如对非 Kubernetes 工作负载的支持或对 egress 的本地支持。

## 解决方案

您可能有兴趣尝试 Linkerd，它定位为 Istio 的更轻量级替代品。

首先，如果您直接按照 Istio 的示例操作，您可以使用类似`kubectl delete all --all`的命令重置您的环境（请注意，这会从您的集群中删除*所有*内容！）。

您可以通过执行以下命令并按终端中的说明操作，手动安装 Linkerd：

```
$ curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh

```

前一个命令的输出将包括额外的步骤，包括更新您的`PATH`以及其他检查和安装命令，这些命令对完成安装 Linkerd 至关重要。在撰写本文时，以下片段显示了这些说明：

```
...
Add the linkerd CLI to your path with:

  export PATH=$PATH:/Users/jonathanmichaux/.linkerd2/bin

Now run:

  linkerd check --pre                     # validate that Linkerd can be inst...
  linkerd install --crds | kubectl apply -f - # install the Linkerd CRDs
  linkerd install | kubectl apply -f -    # install the control plane into the...
  linkerd check                           # validate everything worked!
...

```

当您运行这些`install`命令中的第二个时，您可能会收到错误消息，建议您使用额外的参数重新运行该命令，如下所示：

```
linkerd install --set proxyInit.runAsRoot=true | kubectl apply -f -

```

安装结束时，您将被要求运行一个命令来检查所有内容是否正常运行：

```
$ linkerd check
...
linkerd-control-plane-proxy
---------------------------
√ control plane proxies are healthy
√ control plane proxies are up-to-date
√ control plane proxies and cli versions match

Status check results are √

```

您还应该能够看到在`linkerd`命名空间中运行的 Linkerd pods：

```
$ kubectl get pods -n linkerd
NAME                                     READY   STATUS    RESTARTS   AGE
linkerd-destination-6b8c559b89-rx8f7     4/4     Running   0          9m23s
linkerd-identity-6dd765fb74-52plg        2/2     Running   0          9m23s
linkerd-proxy-injector-f54b7f688-lhjg6   2/2     Running   0          9m22s

```

确保 Linkerd 配置为自动注入 Linkerd 代理到您部署的服务中。您可以通过以下命令为默认命名空间启用此功能：

```
$ kubectl annotate namespace default linkerd.io/inject=enabled
namespace/default annotate

```

## 讨论

威廉·摩根，Buoyant Inc.的联合创始人兼 CEO，于 2016 年首次提出了*服务网格*一词。自那时起，Buoyant 的 Linkerd 社区一直专注于提供一个功能齐全、性能卓越的产品。

正如问题陈述中提到的，在撰写本文时，Linkerd 的主要限制之一是它只能网格化运行在 Kubernetes 上的服务。

## 另请参阅

+   Linkerd 官方的[入门指南](https://oreil.ly/zx-Wx)

# 13.6 将服务部署到 Linkerd 网格中

## 问题

您想将一个服务部署到 Linkerd 网格中，并向其 pod 注入一个 sidecar。

## 解决方案

让我们部署与 Istio 相同的`nginx`服务，它会响应其根端点上的 HTTP GET 请求，并在其他请求上返回 404 响应。

首先创建一个 NGINX 部署：

```
$ kubectl create deployment nginx --image nginx:1.25.2
deployment.apps/nginx created

```

然后将其公开为 Kubernetes 服务：

```
$ kubectl expose deploy/nginx --port 80
service/nginx exposed

```

现在列出默认命名空间中的 pod。在`nginx`服务的 pod 中应该有两个容器：

```
$ kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
nginx-748c667d99-fjjm4   2/2     Running   0          13s

```

如果您调查此 pod 的详细信息，您会发现已注入两个 Linkerd 容器到该 pod 中。其中一个是初始化容器，在路由 TCP 流量到 pod 以及在其他 pod 启动之前终止时起作用。另一个容器是 Linkerd 代理本身：

```
$ kubectl describe pod -l app=nginx | grep Image:
    Image:          cr.l5d.io/linkerd/proxy-init:v2.2.1
    Image:          cr.l5d.io/linkerd/proxy:stable-2.13.5
    Image:          nginx

```

## 讨论

与 Istio 类似，Linkerd 依赖于一个 sidecar 代理，也被称为[大使容器](https://oreil.ly/ooN52)，它被注入到 pod 中并为其运行的服务提供附加功能。

Linkerd CLI 提供了`linkerd inject`命令作为一种有用的替代方法，用于决定何时以及何地将 Linkerd 代理容器注入到应用程序 pod 中，而无需自行操纵标签。您可以在[Linkerd 文档](https://oreil.ly/KJfxJ)中了解更多信息。

## 另请参阅

+   更多关于如何[配置自动 sidecar 注入](https://oreil.ly/TexFs)的信息

+   更多关于[Linkerd 架构](https://oreil.ly/nTiTn)的信息

# 13.7 将流量路由到 Linkerd 中的服务

## 问题

您想将一个服务部署到网格中，该服务将调用您在上一个示例中部署的`nginx`服务，并验证 Linkerd 及其 sidecar 是否拦截和路由流量。

## 解决方案

我们将通过部署一个`curl` pod 来模拟服务网格内的服务间通信，该 pod 将被添加到网格中并调用`nginx`服务。正如您在本示例中所见，Linkerd 中的路由策略定义方式不同。

首先运行一个`curl` pod，您将用它来调用服务。因为您在默认命名空间中启动了`curl` pod，并且在此命名空间中启用了自动注入 sidecar（Recipe 13.5），`curl` pod 将自动获取一个 sidecar 并添加到网格中：

```
$ kubectl run mycurlpod --image=curlimages/curl -i --tty -- sh
Defaulted container "linkerd-proxy" out of: linkerd-proxy, mycurlpod,
linkerd-init (init)
error: Unable to use a TTY - container linkerd-proxy did not allocate one
If you don't see a command prompt, try pressing enter.

```

###### 注意

由于 Linkerd 修改了网格化 pod 中默认的容器排序，之前的`run`命令将失败，因为它试图进入 Linkerd 代理而不是我们的`curl`容器。

要绕过此问题，您可以使用 CTRL-C 解除终端阻塞，然后使用`-c`标志运行命令以连接到正确的容器：

```
$ kubectl attach mycurlpod -c mycurlpod -i -t

```

现在您可以从`curl` pod 中调用`nginx`服务：

```
$ curl -v nginx
*   Trying 10.111.17.127:80...
* Connected to nginx (10.111.17.127) port 80 (#0)
> GET / HTTP/1.1
> Host: nginx
> User-Agent: curl/8.1.2
> Accept: */*
>
< HTTP/1.1 200 OK
< server: nginx/1.25.1
...
<
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...

```

###### 注意

您将看到`nginx`服务的响应，但与 Istio 不同的是，目前还没有明确的指示表明 Linkerd 已成功拦截此请求。

要开始向`nginx`服务添加 Linkerd 路由策略，请在名为*linkerd-server.yaml*的文件中定义 Linkerd 的`Server`资源，如下所示：

```
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  podSelector:
    matchLabels:
      app: nginx
  port: 80

```

然后创建服务器：

```
$ kubectl apply -f linkerd-server.yaml
server.policy.linkerd.io/nginx created

```

现在，如果您从`curl` pod 再次调用服务，您将收到确认信息表明 Linkerd 正在拦截此请求，因为默认情况下它将拒绝未关联授权策略的服务器的请求：

```
$ curl -v nginx
*   Trying 10.111.17.127:80...
* Connected to nginx (10.111.17.127) port 80 (#0)
> GET / HTTP/1.1
> Host: nginx
> User-Agent: curl/8.1.2
> Accept: */*
>
< HTTP/1.1 403 Forbidden
< l5d-proxy-error: client 10.244.0.24:53274: server: 10.244.0.23:80:
unauthorized request on route
< date: Wed, 05 Jul 2023 20:33:24 GMT
< content-length: 0
<

```

## 讨论

正如您所看到的，Linkerd 使用 pod 选择器标签来确定哪些 pod 受到 mesh 策略的管理。相比之下，Istio 的`VirtualService`资源直接通过名称引用服务。

# 13.8 针对 Linkerd 服务器的流量授权

## 问题

您已经将类似`nginx`的服务添加到了 mesh 中，并声明其为 Linkerd 服务器，但现在由于默认情况下 mesh 要求所有声明的服务器进行授权，因此您正在收到 403 Forbidden 响应。

## 解决方案

Linkerd 提供了不同的策略来定义哪些客户端可以联系哪些服务器。在这个例子中，我们将使用 Linkerd 的`AuthorizationPolicy`来指定哪些服务账户可以调用`nginx`服务。

在您的开发环境中，`curl` pod 正在使用`default`服务账户，除非另有指定。在生产环境中，您的服务将具有专用的服务账户。

首先创建一个名为*linkerd-auth-policy.yaml*的文件，如下所示：

```
apiVersion: policy.linkerd.io/v1alpha1
kind: AuthorizationPolicy
metadata:
  name: nginx-policy
spec:
  targetRef:
    group: policy.linkerd.io
    kind: Server
    name: nginx
  requiredAuthenticationRefs:
    - name: default
      kind: ServiceAccount

```

此策略声明任何使用`default`服务账户的客户端将能够访问您在前一篇中创建的名为`nginx`的 Linkerd 服务器。

应用此策略：

```
$ kubectl apply -f linkerd-auth-policy.yaml
authorizationpolicy.policy.linkerd.io/nginx-policy created

```

现在您可以从`curl` pod 调用`nginx`服务并获得 200 OK：

```
$ curl -v nginx
*   Trying 10.111.17.127:80...
* Connected to nginx (10.111.17.127) port 80 (#0)
> GET / HTTP/1.1
> Host: nginx
> User-Agent: curl/8.1.2
> Accept: */*
>
< HTTP/1.1 200 OK
...
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

```

## 讨论

控制对服务器访问的另一种方法包括基于 TLS 身份的策略、基于 IP 的策略、通过使用 pod 选择器特定引用客户端，以及这些方法的任意组合。

此外，可以应用[默认策略](https://oreil.ly/LwiQ_)，限制访问未经 Linkerd `Server`资源正式引用的服务。

## 参见

+   [Linkerd 授权策略文档](https://oreil.ly/FOtW1)
