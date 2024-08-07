# 第二十章：混沌测试、负载测试和实验

本章涵盖了在你的 Kubernetes 集群中测试应用程序的三种不同方法：混沌测试、负载测试和实验。所有这些工具都可以帮助你构建更有用、更具弹性和更高性能的应用程序。它们还可以为你的应用程序提供洞察，并帮助你更好地理解你的用户，并在广泛推出变更之前预测其影响。这些洞察力使你能够做出更好的决策，并识别未来改进的领域。接下来的几节将描述每种测试类型的详细信息、其目标以及开始每项测试之前所需的先决条件。

# 混沌测试

混沌测试，顾名思义，是测试你的应用程序对世界混乱的响应能力。但是混沌究竟意味着什么呢？广义上来说，对于一个应用程序而言，混沌意味着引入不寻常但并非完全意外的边缘条件，观察应用程序如何响应。这使你能够了解你的应用程序是否能够对这些边缘条件具有弹性，这些条件可能在应用程序开发过程中之前从未发生过，但在应用程序运行期间某个时刻可能会发生。通常情况下，我们的应用程序开发都在理想化的条件下进行。不幸的是，长时间暴露在真实世界中时，这些理想化条件会面临在初步开发阶段不存在的错误和故障。这些错误包括通信错误、网络断开、存储问题以及应用程序崩溃和失败。混沌测试就是在测试环境中人为地引入这些错误，观察你的应用程序如何处理它们的艺术。

## 混沌测试的目标

混沌测试的目标是将极端条件引入你的应用程序环境，并观察你的应用程序在这些条件下的行为，特别是它如何失败。以这种方式进行测试，似乎不寻常，因为我们预期并希望观察到失败。虽然总体上我们尽量避免应用程序的失败，但是在测试环境中观察这些失败要好得多，因为此时不会影响到客户或用户。我们希望通过混沌测试观察到失败，因为它们为我们在影响到用户或客户之前修复这些问题提供了机会。

当然，目标是在我们的应用程序中引入一个*现实的*错误水平，以查看它们的行为。引入一个在实践中预计不会发生的错误水平，虽然有趣，但不是时间或资源的好用途。过高的错误水平可以帮助我们加固应对极端环境的应用程序，但如果这样的极端情况从未发生，加固应用程序的努力就是浪费的。当然，每个应用程序都有其所需的不同程度的可变性和弹性水平。对于移动游戏所期望的弹性水平远低于对飞机或汽车所期望的弹性水平。了解应用程序的弹性要求和预期环境对于高质量的混沌测试是至关重要的先决条件。

## 混沌测试的先决条件

要构建一个有用的混沌测试，理解应用程序可能遇到的环境条件至关重要。这包括错误的预期频率以及可能发生的错误类型。例如，你的存储是否已经具备了弹性？如果你正在构建一个使用云支持的无状态应用程序，你可能不需要测试应用程序的硬盘故障，但你可能希望在与云存储解决方案的通信中引入混沌。

在开始混沌测试之前，要考虑你的应用程序中存在的风险，并确定希望引入错误的位置及其频率。在考虑频率时，请记住我们并不试图测试平均情况。平均情况已经在您现有的集成测试中得到很好的代表。相反，我们希望模拟可能仅在一年一次或十年一次发生的环境。你需要充分了解你的应用程序，以描述什么是合理的。

在了解你的应用程序方面，混沌测试的另一个重要先决条件是对你的应用程序的正确性和行为进行高质量的监控。引入混沌到你的环境中是一回事，但要使这种混沌有用，你还需要能够以足够详细的方式观察你的应用程序的运行情况，以确定混沌的影响，并识别你的应用程序需要加固的地方，以能够处理混沌。总的来说，这种监控对于任何生产应用程序都是必需的。除了在弹性方面的核心贡献之外，混沌测试还可以是一个测试，看看你的监控和日志记录是否足以处理真实的故障。

## 混沌测试你的应用程序通信

将混沌注入到应用程序通信中的最简单方法之一是在每个客户端和您的服务之间放置代理。该代理处理客户端和服务器之间的所有网络流量，并注入额外的延迟、断开连接或其他错误等随机故障。有几种不同的开源选项可以用作这种代理，但其中最受欢迎的之一是[ToxiProxy](https://oreil.ly/N8QNF)，由 Shopify 创建。将 ToxiProxy 添加到系统中的最简单方法是在集群中的每个实际服务前有效地运行一个 ToxiProxy 层。

为此，您首先需要将要添加混沌的每个服务重命名。以更详细地查看这一点，假设您有一个名为`backend`的服务，该服务在端口 8080 上提供流量。您可以更新名为`backend`的 Kubernetes 服务为`backend-real`。然后，您可以使用 ToxiProxy 命令行工具创建配置如下的新 ToxiProxy Pods 部署：

```
toxiproxy-cli create -l 0.0.0.0:8080 -u backend-real:8080 backend
```

当您为 ToxiProxy 的这个 Pod 部署构建 Pod 定义时，您可以将此命令作为 PostStart 生命周期钩子运行。此命令配置 ToxiProxy 在 Pod 内的 8080 端口上监听，然后将流量转发到您的实际后端服务，该服务具有 DNS 名称`backend-real`。

接下来，您可以创建一个名为`backend`的新服务来替换您重命名的服务，并将此服务指向您刚刚创建的 ToxiProxy Pods 的部署。这样，您应用程序中与`backend`通信的任何客户端都会自动开始与混沌代理进行通信。

最后，您可以使用 ToxiProxy 命令行工具向应用程序添加混沌，例如发出以下命令：

```
kubectl exec $SomeToxiProxyPod -- toxiproxy-cli toxic add -t latency
  -a latency=2000 backend
```

这将向通过此代理的所有流量添加 2000 毫秒的延迟。如果您在代理部署中创建多个 Pods，则需要为每个 Pod 运行此命令，或者使用脚本或代码进行自动化。

## 测试应用程序操作的混沌测试

除了在通信不稳定时测试应用程序的操作外，还可以考虑在基础设施运行不稳定或过载的情况下测试应用程序的运行情况。

开始基础设施故障的最简单方法是简单地删除 Pods。从单个 Deployment 开始，您可以根据其标签选择器删除 Deployment 中的随机 Pods，使用一个简单的 bash 脚本：

```
NAMESPACE="some-namespace"
LABEL=k8s-app=my-app
PODS=$(kubectl get pods --selector=${LABEL} -n ${NAMESPACE} --no-headers | awk
    '{print $1}')
for x in $PODS; do
    if [ $[ $RANDOM % 10 ] == 0 ]; then
        kubectl delete pods -n $NAMESPACE $x;
    fi;
done
```

当然，如果您更愿意拥有更完整的功能，您可以使用各种[Kubernetes 客户端](https://oreil.ly/Ib1kp)编写代码，甚至使用现有的开源工具如[Chaos Mesh](https://chaos-mesh.org)。

一旦您完成了应用程序中所有微服务部署的移动，您可以继续一次性删除不同服务中的 Pods。这会模拟更广泛的故障。您可以扩展先前的脚本，根据以下方式随机删除特定命名空间内 Deployment 的 Pods：

```
NAMESPACE="some-namespace"
PODS=$(kubectl get pods -n ${NAMESPACE} --no-headers | awk '{print $1}')
for x in $PODS; do
    if [ $[ $RANDOM % 10 ] == 0 ]; then
        kubectl delete pods -n $NAMESPACE $x;
    fi;
done
```

最后，您可以通过导致集群中整个节点失败来模拟您基础设施中的完全故障。有多种方法可以实现这一点。如果您在基于云的 Kubernetes 上运行，您可以使用云 VM API 来关闭或重新启动集群中的机器。如果您在物理基础设施上运行，您可以直接拔掉特定机器的电源插头，或者登录并运行命令来重新启动它。在物理和虚拟硬件上，您还可以通过运行 `sudo sh -c 'echo c > /proc/sysrq-trigger'` 来使您的内核发生紧急情况。

下面是一个简单的脚本，它将在 Kubernetes 集群中大约随机使 10% 的机器发生紧急情况：

```
NODES=$(kubectl get nodes -o jsonpath='{.items[*].status.addresses[0].address}')
for x in $NODES; do
  if [ $[ $RANDOM % 10 ] == 0 ]; then
    ssh $x sudo sh -c 'echo c > /proc/sysrq-trigger'
  fi
done
```

## 对应用程序进行模糊测试以确保安全性和弹性

与混沌测试精神相同的最后一种测试类型是模糊测试。模糊测试与混沌测试类似，因为它引入了随机性和混乱到您的应用程序中，但不是引入故障，而是专注于引入在某种方式上技术上合法但极端的输入。例如，您可以向端点发送一个合法的 JSON 请求，但包含重复字段或特别长或包含随机值的数据。模糊测试的目标是测试您的应用程序对随机极端或恶意输入的弹性。模糊测试最常用于安全测试的背景中，因为随机输入可能导致执行意外代码路径并引入漏洞或崩溃。模糊测试可以帮助您确保您的应用程序对来自恶意或错误输入的混沌具有弹性，除了环境中的故障。模糊测试可以在集群服务级别和单元测试级别都添加。

## 总结

混沌测试是在应用程序运行时引入意外但不是不可能的条件的艺术，并观察发生了什么。在任何故障影响实际应用程序使用之前，将潜在错误和故障引入环境有助于您在它们变得关键之前识别问题区域。

# 负载测试

负载测试用于确定您的应用程序在负载下的行为。使用负载测试工具生成等同于您的应用程序真实生产使用的实际应用流量的真实应用流量。这种流量可以是人工生成的，也可以是从实际生产流量中记录并重放的流量。负载测试可用于识别未来可能成为问题的领域，或者确保新代码和功能不会引起退步。

## 负载测试的目标

负载测试的核心目标是了解你的应用在负载下的行为。当你构建一个应用时，通常只有少数用户偶尔访问。这些流量足以理解应用的正确性，但无法帮助我们了解应用在实际负载下的表现。因此，为了了解你的应用在生产环境中的运行情况，负载测试是必要的。

负载测试的两个基本用途是估算当前容量和预防回归。预防回归是利用负载测试确保新版本软件可以与之前版本承受相同负载的使用。每次我们发布软件的新版本时，都会有新的代码和配置（如果没有，那发布的意义何在？）。虽然这些代码变更引入了新功能并修复了错误，但它们也可能引入性能退化：新版本无法像旧版本那样提供相同水平的负载。当然，有时这些性能退化是已知且预期的；例如，新功能可能使计算变得更复杂，因此更慢，但即使在这种情况下，负载测试也是必要的，以确定基础设施（例如 pod 数量、它们需要的资源）需要如何扩展以支持生产流量。

与回归预防相比，后者用于捕获新引入到应用程序中的问题，预测性负载测试用于在问题发生之前预测。对于许多服务来说，服务的使用是稳定增长的。每个月都会有更多的用户和对你的服务的请求。总体而言，这是一件好事，但是要保持这些用户的满意意味着不断改进基础设施以跟上新的负载。预测性负载测试利用应用程序的历史增长趋势，并使用它们来测试你的应用程序，仿佛它在未来运行一样。例如，如果你的应用程序流量每个月增长 10%，你可以在当前峰值流量的 110%处运行预测性负载测试，模拟你的应用程序在下个月的工作情况。虽然扩展你的应用程序可能只需添加更多的副本和资源，但通常应用程序中的基本瓶颈需要重新架构。预测性负载测试允许你预见未来并进行这些更改，而不会因负载增加而导致用户面临的紧急故障。

预测性负载测试也可以用于预测应用在发布前的行为。与使用历史信息不同，你可以使用对发布时使用情况的预测来确保发布成功而不是灾难。

## 负载测试的先决条件

负载测试用于确保您的应用程序在承受重大负载时仍能正常运行。此外，与混沌测试类似，负载测试也可能因负载而在您的应用程序中引入故障条件。因此，负载测试与应用程序的可观察性具有相同的先决条件。要成功使用负载测试，您需要验证应用程序是否正常运行，并且有足够的信息来了解故障发生的位置和原因（如果有的话）。

除了应用程序的核心可观察性之外，负载测试的另一个关键先决条件是能够为您的测试生成真实的负载。如果您的负载测试不能紧密模拟真实世界用户的行为，那么它将毫无用处。具体来说，想象一下，如果您的负载测试不断重复向单个用户发出请求，那么在许多应用程序中，这样的流量将导致不现实的缓存命中率，并且您的负载测试将似乎展示出一种在更真实的流量下是不可能实现的处理大量负载的能力。

## 生成真实流量

为了为您的应用程序生成真实世界的流量模式，方法会因应用程序的类型而异。例如，对于某些更多只读操作的网站，比如新闻网站，仅仅使用某种概率分布重复访问每个不同页面可能就足够了。但是对于许多应用程序，特别是涉及读写操作的应用程序，生成真实的负载测试的唯一方法是记录真实世界的流量并回放。其中一种最简单的方法是将每个 HTTP 请求的完整细节写入文件，然后稍后重新发送这些请求到服务器。

不幸的是，这种方法可能会带来复杂性。将所有请求记录到应用程序可能涉及用户隐私和安全的首要后果。在许多情况下，向应用程序发出的请求同时包含私人信息和安全令牌。如果将所有这些信息记录到文件以供回放，则必须非常小心地处理这些文件，以确保尊重用户的隐私和安全性。

记录并回放实际用户请求的另一个挑战与请求本身的及时性有关。例如，如果请求包含了关于最新新闻事件的搜索查询，这些请求在几周（或几个月）后的行为将会非常不同。旧新闻相关的消息会大大减少。及时性还会影响应用程序的正确行为。请求通常包含安全令牌，如果您在正确处理安全性，这些令牌的生命周期很短。这意味着验证时记录的令牌可能无法正确工作。

最后，当请求向后端存储系统写入数据时，修改存储的请求必须在生产存储基础设施的副本或快照中执行。如果在设置这一点时不小心，可能会导致客户数据的重大问题。

出于所有这些原因，简单地记录和回放请求虽然简单，但不是最佳实践。相反，更有用的使用请求的方式是建立服务使用方式的模型。有多少读请求？针对哪些资源？有多少写入操作？使用这个模型，您可以生成具有现实特征的合成负载。

## 测试您的应用程序负载

一旦生成了用于驱动负载测试的请求，将这种负载应用到您的服务就是一件简单的事情。不幸的是，实际应用中很少会这么简单。在大多数实际应用中，涉及到数据库和其他存储系统。为了正确模拟承受负载的应用程序，您还需要写入存储系统，但不能写入生产数据存储，因为这是人为负载。因此，要正确进行应用程序负载测试，您需要能够启动一个真实的应用程序副本及其所有依赖关系。

一旦您的应用克隆运行起来，发送所有请求就成了问题。事实证明，大规模负载测试也是分布式系统问题。您将希望使用大量不同的 Pod 来向您的应用程序发送负载。这样可以通过负载均衡器均匀分发请求，并使得发送超过单个 Pod 网络支持负载成为可能。您需要做出的选择之一是是否将这些负载测试 Pod 运行在与您的应用程序相同的集群中，还是在单独的集群中运行。在同一集群内运行 Pod 最大化了您可以发送到应用程序的负载，但它确实会对从互联网上引入流量到您的应用程序的边缘负载均衡器进行测试。根据您希望测试的应用程序的哪些部分，您可能希望在集群内运行负载，集群外运行，或者两者兼而有之。

在 Kubernetes 中运行分布式负载测试的两个流行工具是[JMeter](https://oreil.ly/MXBgj)和[Locust](https://locust.io)。两者都提供了描述要发送到您的服务的负载的方式，并允许您部署分布式负载测试机器人到 Kubernetes 中。

## 使用负载测试调整您的应用程序

除了使用负载测试来防止性能回归和预测未来的性能问题外，负载测试还可以用于优化您的应用程序的资源利用率。对于任何给定的服务，可以调整多个变量，并且可以影响系统性能。对于本讨论的目的，我们考虑了三个变量：Pod 的数量，核心数和内存。

起初，似乎一个应用在相同数量的副本乘以核心数的情况下表现相同。也就是说，一个每个有三个核心的五个 pod 的应用与一个每个有五个核心的三个 pod 的应用表现相同。在某些情况下，这是正确的，但在许多情况下，情况并非如此；服务的具体细节和其瓶颈的位置通常会导致行为上的差异，这些差异很难预料。例如，使用 Java、dotnet 或 Go 等语言构建的应用程序提供垃圾收集：如果只有一两个核心，应用程序会对垃圾收集器进行显着不同的调整，而如果有多个核心，则会有所不同。

同样适用于内存。更多的内存意味着可以将更多内容保留在缓存中，这通常会带来更好的性能，但这种好处有一个渐近限制。您不能简单地为服务抛出更多内存，并期望它继续提高性能。

通常，了解你的应用在不同配置下的行为方式的唯一方法是实际进行实验。要做到这一点，你可以设置一组实验性配置，其中包含不同的 pod、核心和内存值，并对每个配置运行负载测试。通过这些实验的数据，你通常可以识别出行为模式，这些模式可以洞察系统性能的特定细节，并且你可以使用结果来选择最高效的服务配置。

## 总结

性能是构建令用户满意的应用程序的关键部分。负载测试确保您不会引入影响性能并导致用户体验不佳的退化。负载测试还可以作为一台时光机，让您可以想象应用程序在未来的行为，并对架构进行更改以支持额外的增长。负载测试还可以帮助您理解和优化资源使用，降低成本并提高效率。

# 实验

与混沌测试和负载测试相比，实验不是用于发现服务架构和操作中的问题，而是用于识别改进用户如何使用您的服务的方式。实验是对您的服务的长期更改，通常涉及用户体验，在这种更改中，一小部分用户（例如，所有流量的 1%）将获得稍有不同的体验。通过检查控制组（没有更改的组）和实验组（经历了不同体验的组）之间的差异，您可以理解更改的影响，并决定是否继续进行实验或广泛推广更改。

## 实验的目标

当我们构建一个服务时，我们构建它是有一个目标的。那个目标往往是为用户或客户提供一个有用、易于使用和令人愉悦的东西。但我们怎么知道我们是否实现了这个目标呢？我们很容易看到我们的网站在混乱环境中会崩溃，或者在负载量较小时会失败，但了解用户体验我们服务的方式可能会比较棘手。

几种传统的了解用户体验的方法包括调查，通过这些调查您可以询问用户对当前服务的感受。虽然这在理解我们服务的当前表现方面很有用，但要使用调查来预测未来变化的影响则更加困难。就像性能回归一样，最好在变更完全部署之前就知道影响。这是任何实验的主要目标：以最小影响学习我们用户体验。

## 实验的先决条件

就像我们小时候在科学展览会上一样，每一个好的实验都始于一个好的假设，这也是我们服务实验的一个自然前提。我们考虑要进行某种变化，我们需要猜测它对用户体验的影响。

当然，为了理解用户体验的影响，我们还需要能够衡量用户体验。这些数据可以通过之前提到的调查来获取，通过这些调查可以收集到像满意度（“请您评价我们一到五分”）或净推荐值（“您多有可能向朋友推荐此服务？”）等指标。或者可以从与用户行为相关的被动指标中获取（“他们在我们网站上花费了多少时间？”或“他们点击了多少个页面？”等）。

一旦您有了假设和衡量用户体验的方法，您就可以开始实验了。

## 实验设置

有两种不同的方式可以设置实验。您采取的方法取决于正在测试的具体内容。您可以在单个服务中包含多种可能的体验，或者部署两个服务副本并使用服务网格来在它们之间分配流量。

第一种方法是将代码的两个版本都提交到您的发布二进制文件中，并使用服务收到请求的某些属性来在实验组和对照组之间进行切换。您可以使用 HTTP 头部、cookie 或查询参数来让用户明确选择参与实验。或者您可以使用请求的特征，如源 IP，随机选择用户进行实验。例如，您可以选择 IP 地址以某个数字结尾的用户进行实验。

实施实验的常见方式是使用显式功能标志，用户通过提供打开实验的查询参数或 Cookie 来选择参与实验。这是允许特定客户尝试新功能或演示新功能而不广泛发布的好方法。功能标志还可以在不稳定情况下快速启用或禁用功能。许多开源项目，例如 [Flagger](https://flagger.app)，可以用于实施功能标志。

将实验放置在与控制代码相同的二进制文件中的好处在于，将其推广到生产环境是最简单的，但这种简单性也带来了两个缺点。第一个是，如果实验代码不稳定并且崩溃，它也可能影响您的生产流量。另一个是，因为任何更改都与服务的完整发布相关联，因此更新实验或推出新实验的速度要慢得多。

实验的第二种方法是部署两个（或更多）不同版本的服务。在这种方法中，您有控制生产服务接收大部分流量，以及一个单独的实验部署服务，只接收部分流量。您可以使用服务网格（在 第九章 中描述）将小部分流量路由到此实验部署，而不是生产部署。尽管这种方法在实施上更复杂，但比将实验代码包含在生产二进制文件中要更灵活和更健壮。因为它需要完全新的代码部署，设置实验的前期成本增加了，但因为它只影响实验流量，所以您可以随时轻松地部署新版本的实验（甚至多个版本的实验），而不影响大部分流量。

此外，由于服务网格可以衡量请求是否成功，如果实验代码开始失败，可以迅速将其从使用中移除，并最小化用户影响。当然，检测这些失败可能是一个挑战。您需要确保实验基础设施独立监控，而不是与标准生产监控混为一谈；否则，实验失败可能会在当前生产基础设施处理的成功请求中丢失。理想情况下，Pod 的名称或部署应提供足够的上下文，以确定监控信号是来自生产还是实验。

通常来说，使用单独的部署以及某种流量路由器如服务网格是进行实验的最佳实践，但这需要大量的基础设施来设置。对于你的初步实验，或者如果你是一个已经相当敏捷的小团队，可能直接提交实验性代码是最简便的实验和迭代路径。

## 总结

实验能让你在将更改推广到广泛用户群之前了解这些变化对用户体验的影响。实验在帮助我们迅速了解可行的变更及如何更新我们的服务以更好地为用户服务方面起到关键作用。实验使得改进我们的服务变得更加容易、快速和安全。

# 混沌测试、负载测试和实验总结

在本章中，我们涵盖了多种学习服务更加弹性、性能更好和更有用的不同方式。正如使用单元测试来测试你的代码是软件开发过程的关键部分一样，使用混沌、负载和实验来测试你的服务是服务设计和运营的关键部分。
