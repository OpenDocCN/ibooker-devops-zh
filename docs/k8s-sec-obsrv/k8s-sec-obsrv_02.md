# 第二章：基础设施安全

许多 Kubernetes 配置在默认情况下是不安全的。在本章中，我们将探讨如何在基础设施级别保护 Kubernetes。通过加固主机使托管 Kubernetes 的服务器或虚拟机更加安全，加固集群以保护 Kubernetes 控制平面组件，并进行网络安全设置以将集群与周围基础设施集成，可以使其更加安全。请注意，本章讨论的概念适用于自托管 Kubernetes 集群以及托管 Kubernetes 集群。

主机加固

这涵盖了操作系统的选择，避免在主机上运行非必要进程以及基于主机的防火墙设置。

集群加固

这涵盖了一系列配置和策略设置，用于加固控制平面，包括配置 TLS 证书，锁定 Kubernetes 数据存储，加密静态密钥，凭证轮换，以及用户认证和访问控制。

网络安全

这涵盖了安全地将集群与周围基础设施集成，特别是允许集群与周围基础设施之间的网络交互，用于控制平面、主机和工作负载流量。

让我们详细看看每个方面的细节，并探讨构建安全基础设施以保护您的 Kubernetes 集群所需的内容。

# 主机加固

安全的主机是安全 Kubernetes 集群的重要基石。当您考虑主机时，它是在构成 Kubernetes 集群的工作负载背景下。现在我们将探讨确保主机具有强大安全姿态的技术。

## 操作系统的选择

许多企业在其所有基础设施上标准化使用单一操作系统，这意味着可能已经为您做出了选择。但是，如果有选择操作系统的灵活性，那么考虑专为容器设计的现代不可变 Linux 发行版是值得的。这些发行版有以下优势：

+   它们通常具有更新的内核，包括最新的漏洞修复以及最新技术（如 eBPF）的实现，这些技术可以被 Kubernetes 网络和安全监控工具利用。

+   它们设计为不可变，这在安全方面带来了额外的好处。在此上下文中，不可变性意味着根文件系统被锁定，应用程序无法更改。只能使用容器安装应用程序。这将应用程序与根文件系统隔离，并显著减少了恶意应用程序破坏主机的能力。

+   它们通常包括自动更新到较新版本的能力，上游版本已准备好快速发布以解决安全漏洞。

设计用于容器的现代不可变 Linux 发行版的两个流行示例是 Flatcar Container Linux（最初基于 CoreOS Container Linux）和 Bottlerocket（最初由亚马逊创建和维护）。

无论您选择哪种操作系统，监视上游安全公告以便了解新发现的安全漏洞，并确保您有适当的流程来更新您的集群到更新版本以解决关键的安全漏洞，是一个良好的实践。根据您对这些漏洞的评估，您将需要决定是否将集群升级到操作系统的新版本。在考虑操作系统选择时，您还必须考虑来自主机操作系统的共享库，并了解它们对将部署在主机上的容器的影响。

另一个安全最佳实践是确保应用开发人员不依赖特定版本的操作系统或内核，因为这样将无法根据需要更新主机操作系统。

## 非必要进程

每个运行的主机进程都是黑客的潜在攻击向量。从安全的角度来看，最好是删除默认情况下可能正在运行的任何非必要进程。如果一个进程对于成功运行 Kubernetes、管理您的主机或保护您的主机没有必要，则最好不要运行该进程。如何禁用进程将取决于您的特定设置（例如，通过 systemd 配置更改或从 /etc/init.d/ 中删除初始化脚本）。

如果您正在使用针对容器优化的不可变 Linux 发行版，则已经消除了非必要进程，您只能作为容器运行额外的进程/应用程序。

## 防火墙主机化

为了进一步锁定托管 Kubernetes 的服务器或虚拟机，可以为主机本身配置本地防火墙规则，以限制允许与主机交互的 IP 地址范围和端口。

根据您的操作系统，可以使用传统的 Linux 管理工具，例如 iptables 规则或 firewalld 配置来实现这一点。重要的是确保这些规则与 Kubernetes 控制平面和您计划使用的 Kubernetes 网络插件兼容，以免阻塞 Kubernetes 控制平面、Pod 网络或 Pod 网络控制平面。正确配置这些规则并随时间保持其更新可能是一个耗时的过程。此外，如果使用不可变的 Linux 发行版，您可能无法直接使用这些工具。

幸运的是，一些 Kubernetes 网络插件可以帮助解决这个问题。例如，几个 Kubernetes 网络插件，如 Weave Net、Kube-router 和 Calico，包括应用网络策略的能力。你应该审查这些插件，并选择一个也支持将网络策略应用于主机本身（而不仅仅是 Kubernetes Pod）。这样可以大大简化集群中主机的安全性，并且基本上与操作系统无关，包括与不可变 Linux 发行版一起使用。

## 始终研究最新的最佳实践

随着安全研究社区确定新的漏洞或攻击向量，安全最佳实践随时间而演变。许多这些都有详细的在线文档，并且可以免费获取。

例如，互联网中心安全性维护了免费的 PDF 指南，详细说明了安全地配置许多常见操作系统所需的行动。称为 CIS 基准，它们是确保你覆盖了保护主机操作系统所需的许多重要行动的极好资源。你可以在他们的网站上找到 [最新的 CIS 基准列表](https://oreil.ly/dpUnC)。请注意，还有针对 Kubernetes 的特定基准，我们将在本章后面讨论。

# 集群加固

Kubernetes 默认情况下是不安全的。因此除了加固组成集群的主机外，加固集群本身也非常重要。这可以通过 Kubernetes 组件和配置管理、身份验证和基于角色的访问控制（RBAC），以及保持集群更新到最新的 Kubernetes 版本来实现，以确保集群具有最新的漏洞修复。

## 保护 Kubernetes 数据存储

Kubernetes 使用 etcd 作为其主要数据存储。这是存储所有集群配置和期望状态的地方。访问 etcd 数据存储与在所有节点上拥有 root 登录权限基本上是等效的。如果恶意用户获取了对 etcd 数据存储的访问权限，几乎所有你在集群内实施的其他安全措施都将失效。他们将完全控制你的集群，包括在任何节点上以提升的权限运行任意容器的能力。

保护 etcd 的主要方法是使用 etcd 本身提供的安全功能。这些功能基于 x509 公钥基础设施（PKI），使用密钥和证书的组合。它们确保所有传输中的数据都使用 TLS 加密，并且所有访问都受到强密码的限制。最好为不同 etcd 实例之间的对等通信配置一组凭据（密钥对和证书），并为来自 Kubernetes API 的客户端通信配置另一组凭据。作为此配置的一部分，etcd 还必须配置证书颁发机构（CA）的详细信息，用于生成客户端凭据。

一旦正确配置了 etcd，只有拥有有效证书的客户端才能访问它。然后，您必须配置 Kubernetes API 服务器，使其能够使用客户端证书、密钥和证书颁发机构来访问 etcd。

您还可以使用网络级防火墙规则限制对 etcd 的访问，以便只能从 Kubernetes 控制节点（托管 Kubernetes API 服务器的节点）访问它。根据您的环境，可以使用传统防火墙、虚拟云防火墙或在 etcd 主机上设置规则（例如支持主机端点保护的网络策略实施）来阻止流量。这最好是作为深度防御策略的一部分，与使用 etcd 的安全功能一起使用，因为仅通过防火墙规则限制访问并不能解决加密 Kubernetes 敏感数据在传输中的安全需求。

除了为 Kubernetes 安全地访问 etcd，建议不要将 Kubernetes etcd 数据存储用于除 Kubernetes 之外的任何其他用途。换句话说，不要在数据存储中存储非 Kubernetes 数据，也不要让其他组件访问 etcd 集群。如果正在运行使用 etcd 作为数据存储的应用程序或基础架构（在集群内部或集群外部），最佳实践是为其设置单独的 etcd 集群。可以辩称的例外情况是，如果应用程序或基础架构具有足够的特权，以至于其数据存储的妥协也会导致 Kubernetes 的完全妥协。还非常重要的是要保持 etcd 的备份，并确保备份的安全性，以便能够从失败（例如升级失败或安全事件）中恢复。

## 保护 Kubernetes API 服务器的安全性

在 etcd 数据存储之上的另一层，需要保护的下一个关键是 Kubernetes API 服务器。与 etcd 一样，可以使用 x509 PKI 和 TLS 来实现安全性。如何以这种方式引导集群的详细步骤因您使用的 Kubernetes 安装方法而异，但大多数方法包括创建所需的密钥和证书并将其分发到其他 Kubernetes 集群组件的步骤。值得注意的是，某些安装方法可能为某些组件启用不安全的本地端口，因此重要的是熟悉每个组件的设置，以识别潜在的未安全流量，以便采取适当的安全措施。

## 在静止状态下加密 Kubernetes Secrets

Kubernetes 可以配置为加密存储在 etcd 中的敏感数据，例如 Kubernetes secrets。这样可以防止任何攻击者访问 etcd 或其离线副本（例如离线备份）时泄露这些机密。

默认情况下，Kubernetes 不会对存储的秘密进行加密，当启用加密时，它仅在将秘密写入 etcd 时进行加密。因此，在启用静态加密时，重写所有秘密（通过标准 kubectl apply 或 update 命令）以触发其在 etcd 中的加密至关重要。

Kubernetes 支持各种加密提供程序。根据加密最佳实践选择推荐的加密方法非常重要。主推的选择是基于 AES-CBC 和 PKCS #7 的加密。这使用 32 字节密钥提供非常强大的加密，并且相对较快。有两种不同的提供程序支持此加密：

+   完全在 Kubernetes 中运行并使用本地配置密钥的本地提供程序。

+   使用外部密钥管理服务（KMS）的 KMS 提供程序来管理密钥。

本地提供程序将其密钥存储在 API 服务器的本地磁盘上。因此，如果 API 服务器主机受到攻击，则所有您的秘密都会受到威胁。根据您的安全立场，这可能是可以接受的。

KMS 提供程序使用一种称为*信封加密*的技术。使用信封加密时，每个秘密都使用动态生成的数据加密密钥（DEK）进行加密。然后，DEK 使用由 KMS 提供的密钥加密密钥（KEK）进行加密，并将加密的 DEK 与 etcd 中加密的秘密一起存储。KEK 始终由 KMS 托管为中心的信任根，永远不会存储在集群中。大多数大型公共云提供商提供基于云的 KMS 服务，可用作 Kubernetes 中的 KMS 提供程序。对于本地集群，可以使用第三方解决方案，如 HashiCorp 的 Vault，作为集群的 KMS 提供程序。由于详细的实施方式有所不同，评估 KMS 如何验证 API 服务器以及 API 服务器主机的妥协是否可能威胁到您的秘密，因此仅能提供有限的优势，相比本地加密提供程序。

如果预计有异常高量的加密存储读/写操作，则使用 secretbox 加密提供程序可能会更快。但是，secretbox 是一种较新的标准，在撰写时的审查较少。因此，在需要高水平审查的环境中，可能不被认为是可接受的。此外，KMS 提供程序尚不支持 secretbox，必须使用本地提供程序，将密钥存储在 API 服务器上。

加密 Kubernetes 秘密是最常见的必须在静态时加密要求，但请注意，如果需要，还可以配置 Kubernetes 以加密其他 Kubernetes 资源的存储。

值得注意的是，如果您有超出 Kubernetes secrets 能力范围的需求，可以使用第三方的秘密管理解决方案。其中一个这样的解决方案，已被提及作为 Kubernetes 中信封加密的潜在 KMS 提供者，是 HashiCorp 的 Vault。除了为 Kubernetes 提供安全的秘密管理外，Vault 还可以在企业范围内更广泛地管理秘密，如果需要的话。Vault 还是早期 Kubernetes 版本中填补重要差距的非常受欢迎的选择，这些版本不支持加密处于静态状态的秘密。

## 频繁轮换凭据

频繁地轮换凭据使攻击者更难利用可能获得的任何被泄露的凭据。因此，设置任何 TLS 证书或其他凭据的短寿命并自动化其轮换是最佳实践。大多数身份验证提供程序可以控制签发证书或服务令牌的有效期多长时间，尽可能使用短的生命周期是最佳的，例如每天轮换密钥或如果特别敏感则更频繁。这需要包括用于外部集成或作为集群引导过程的任何服务令牌。

完全自动化凭据的轮换可能需要定制的 DevOps 开发工作，但通常与尝试定期手动轮换凭据相比，代表了一个很好的投资。

在对存储在静态状态下的 Kubernetes secrets 进行密钥轮换时（如前一节讨论的），本地提供程序支持多个密钥。第一个密钥始终用于加密任何新的秘密写入。对于解密，密钥按顺序尝试，直到成功解密秘密。由于密钥仅在写入时进行加密，因此重写所有秘密（通过标准的 kubectl apply 或更新命令）以触发使用最新密钥进行加密是很重要的。如果秘密的轮换是完全自动化的，则写入将作为此过程的一部分而发生，无需单独步骤。

当使用 KMS 提供者（而不是本地提供者）时，可以轻松地轮换 KEK 而无需重新加密所有秘密，这可以减少重新加密大量大小适当的秘密时的性能影响。

## 认证和 RBAC

在前面的部分中，我们主要关注在集群内部保障程序化/代码访问的安全性。同样重要的是要遵循最佳实践，以保障用户与集群的交互。这包括为每个用户创建单独的用户账户，并使用 Kubernetes RBAC 为用户授予他们执行角色所需的最小访问权限，遵循最小特权访问原则。通常最好使用组和角色来实现这一点，而不是为个别用户分配 RBAC 权限。这样做可以更轻松地随着时间调整不同用户组的权限，同时减少定期审查/审核集群中用户权限是否正确和最新的工作量。

Kubernetes 对于用户有限的内置认证能力，但可以与大多数外部企业认证提供者集成，如公共云提供商的 IAM 系统或本地认证服务，可以直接或通过第三方项目（如最初由 CoreOS 创建的 Dex）。通常建议与其中一个外部认证提供者集成，而不是使用 Kubernetes 基本认证或服务账户令牌，因为外部认证提供者通常具有更友好的凭据轮换支持，包括指定密码强度和轮换频率时间段的能力。

## 限制云元数据 API 访问

大多数公共云提供了一个可从每个主机/VM 实例本地访问的元数据 API。这些 API 提供对实例的云凭据、IAM 权限和实例可能具有的其他潜在敏感信息的访问。默认情况下，这些 API 可由运行在实例上的 Kubernetes Pod 访问。任何被攻击的 Pod 都可以使用这些凭据提升其在集群内的预期特权级别，或者访问实例可能具有权限访问的其他云提供商托管服务。

解决这个安全问题的最佳实践是：

+   根据云提供商的推荐机制提供任何必需的 Pod IAM 凭据。例如，Amazon EKS 允许您为服务账户分配唯一的 IAM 角色，Microsoft Azure 的 AKS 允许您为 Pod 分配托管标识，Google Cloud 的 GKE 允许您通过工作负载标识分配 IAM 权限。

+   限制每个实例的云权限至最低限度，以减少从实例访问元数据 API 的任何被攻击影响。

+   Use network policies to block pod access to the metadata API. This can be done with per-namespace Kubernetes network policies, or preferably with extensions to Kubernetes network policies such as those offered by Calico, which enable a single network policy to apply across the whole of the cluster (without the need to create a new Kubernetes network policy each time a new namespace is added to the cluster). This topic is covered in more depth in the Default Deny and Default App Policy section of Chapter 7.

## Enable Auditing

Kubernetes auditing provides a log of all actions within the cluster with configurable scope and levels of detail. Enabling Kubernetes audit logging and archiving audit logs on a secure service is recommended as an important source of forensic details in the event of needing to analyze a security breach.

The forensic review of the audit log can help answer questions such as:

+   What happened, when, and where in the cluster?

+   Who or what initiated it and from where?

In addition, Kubernetes audit logs can be actively monitored to alert on suspicious activity using your own custom tooling or third-party solutions. There are many enterprise products that you can use to monitor Kubernetes audit logs and generate alerts based on configurable match criteria.

The details of what events are captured in Kubernetes audit logs are controlled using policy. The policy determines which events are recorded, for which resources, and with what level of detail.

Each action being performed can generate a series of events, defined as stages:

RequestReceived

Generated as soon as the audit handler receives the request

ResponseStarted

Generated when the response headers are sent, but before the response body is sent (generated only for long-running requests such as watches)

ResponseComplete

Generated when the response body has been completed

Panic

Generated when a panic occurs

The level of detail recorded for each event can be one of the following:

None

Does not log the event at all

Metadata

Logs the request metadata (user, timestamp, resource, verb, etc.) but not the request details or response body

Request

Logs event metadata and the request body but not the response body

RequestResponse

Logs the full details of the event, including the metadata and request and response bodies

Kubernetes’ audit policy is very flexible and well documented in the main Kubernetes documentation. Included here are just a couple of simple examples to illustrate.

To log all requests at the metadata level:

```
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
```

To omit the `RequestReceived` stage and to log pod changes at the `RequestResponse` level and configmap and secrets changes at the metadata level:

```
apiVersion: audit.k8s.io/v1 # This is required.
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods"]
  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]
```

第二个示例说明了审计日志中敏感数据的重要考虑因素。根据对审计日志访问安全性的要求，确保审计策略不记录机密或其他敏感数据可能是至关重要的。就像对静止状态的加密密码的做法一样，通常建议始终从审计日志中排除敏感细节。

## 限制对 alpha 或 beta 功能的访问

每个 Kubernetes 发布版本都包含 alpha 和 beta 功能。可以通过为各个 Kubernetes 组件指定功能门标志来控制它们的启用。由于这些功能正在开发中，它们可能存在限制或错误，从而导致安全漏洞。因此，良好的做法是确保所有不打算使用的 alpha 和 beta 功能都被禁用。

Alpha 功能通常（但并非总是）默认禁用。它们可能存在错误，并且该功能的支持可能会在不向后兼容的情况下发生根本性变化，或在未来的版本中被删除。一般建议仅将其用于测试集群，而不是生产集群。

Beta 功能通常默认启用。它们在版本发布之间可能会以不向后兼容的方式更改，但通常被认为经过合理测试且安全可用。但与任何新功能一样，它们天生更有可能存在漏洞，因为它们的使用较少且审核较少。

评估 alpha 或 beta 功能可能提供的价值，对比其带来的安全风险，以及在发布版本之间功能非向后兼容变更可能带来的操作风险是非常重要的。

## 频繁升级 Kubernetes

在任何大型软件项目中，随着时间的推移，新的漏洞是不可避免的。Kubernetes 开发者社区在及时响应新发现的漏洞方面表现良好。严重漏洞通常在封禁期间修复，这意味着漏洞的知识不会在开发者有时间制定修复方案之前公开。随着时间的推移，较旧的 Kubernetes 版本中已公开已知漏洞的数量增加，这可能会增加较旧集群的安全风险。

为减少集群被妥协的风险，定期升级集群非常重要，并确保能够紧急升级以应对发现的严重漏洞。

所有 Kubernetes 安全更新和漏洞报告（一旦有任何封禁结束）通过公开且免费加入的 *kubernetes-announce* 邮件组。强烈建议任何希望跟踪已知漏洞以最小化安全风险的人加入此组。

# 使用托管的 Kubernetes 服务

减少执行本章建议所需工作的一种方式是使用主要的公共云托管 Kubernetes 服务之一，例如 EKS、AKS、GKE 或 IKS。使用这些服务将安全性从完全由您自己负责减少到共享责任模型。共享责任意味着服务默认包含了许多安全元素，或者可以轻松配置以支持这些安全元素，但您仍然需要自己负责某些元素，以确保整体安全。

具体细节因您使用的公共云服务而异，但毫无疑问，所有这些服务都会做大量的重活，这降低了与自行安装和管理集群相比所需的安全工作量。此外，公共云提供商和第三方提供了大量资源，详细说明了作为共享责任模型的一部分，您需要采取哪些措施来全面确保您的集群安全。例如，其中一个资源就是 CIS 基准，接下来将讨论它。

## CIS 基准

正如本章前面讨论的那样，CIS 维护了免费的 PDF 指南，提供了对许多常见操作系统进行安全配置指导。这些指南称为 CIS 基准，可帮助您进行主机硬化，是一种无价的资源。

除了帮助进行主机硬化外，还有针对 Kubernetes 本身的 CIS 基准，包括对许多流行的托管 Kubernetes 服务的配置指导，这可以帮助您实施本章指南中的大部分内容。例如，GKE CIS 基准包括指导如何确保集群配置了节点的自动升级，并且使用 Google Groups 管理 Kubernetes 身份验证和 RBAC。这些指南是高度推荐的资源，以获取关于如何确保 Kubernetes 集群安全所需步骤的最新实用建议。

除了指南本身外，还有第三方工具可用来评估运行中的集群与许多这些基准的状态。一个流行的工具是 kube-bench，这是一个开源项目，最初由 Aqua Security 团队创建。或者，如果您更喜欢一个更成熟的解决方案，那么许多企业产品在其集群管理仪表板中内置了 CIS 基准和其他安全合规工具和警报功能。在理想情况下，这些工具会定期自动运行，有助于验证集群的安全姿态，并确保在集群创建时可能采取的谨慎安全措施在管理和更新集群的过程中不会意外丢失或被破坏。

## 网络安全

当在集群与集群范围之外的周围基础设施安全地集成时，网络安全是首要考虑因素。有两个方面需要考虑：如何保护集群免受集群外部的攻击，以及如何保护集群内部任何受损元素对集群外部基础设施的影响。这适用于集群工作负载级别（即 Kubernetes pod）和集群基础设施级别（即 Kubernetes 控制平面以及运行 Kubernetes 集群的主机）。

首先要考虑的是集群是否需要从公共互联网访问，直接（例如，一个或多个节点具有公共 IP 地址）或间接（例如，通过可从互联网访问的负载均衡器或类似设备）。如果集群可以从互联网访问，那么黑客的攻击或探测活动数量将大幅增加。因此，如果集群不需要强制要求从互联网访问，强烈建议在可路由级别上不允许任何访问（即确保互联网上的数据包无法到达集群）。在企业内部环境中，这可能涉及选择用于集群的 IP 地址范围及其在企业网络中的可路由性。如果使用公共云托管的 Kubernetes 服务，您可能会发现这些设置仅在集群创建时可以设置。例如，在 GKE 中，是否可以从互联网访问 Kubernetes 控制平面可以在集群创建时设置。

集群内的网络策略是下一道防线。网络策略可用于限制集群与集群外部基础设施之间的工作负载和主机通信。它具有两大优势：既能意识到工作负载（即限制构成单个微服务的一组 pod 的通信能力），又能在平台上不可知（即可以在任何环境中使用相同的技术和策略语言，无论是企业内部环境还是公共云中）。网络策略将在专门章节中深入讨论。

最后，强烈建议使用周边防火墙，或其云等效物，如安全组，以限制流向/流出集群的流量。在大多数情况下，这些防火墙不了解 Kubernetes 工作负载，因此无法理解单个 pod，并且通常仅能以集群整体的方式进行粒度控制。尽管存在这些限制，它们作为防御策略的一部分仍然具有价值，但仅靠它们本身可能不足以满足任何注重安全的企业需求。

如果需要更强的周边防御，有策略和第三方工具可以使周边防火墙或其云等效物更加有效：

+   一种方法是将集群中的少数特定节点指定为具有特定访问级别的节点，而这种访问级别不授予集群中的其他节点。然后可以使用 Kubernetes 的污点（taints），确保只有需要该特殊访问级别的工作负载被调度到这些节点上。这样一来，可以基于特定节点的 IP 地址设置外围防火墙规则，允许所需的访问，而集群中的所有其他节点则被拒绝在集群外访问。

+   在本地环境中，一些 Kubernetes 网络插件允许您使用可路由的 Pod IP 地址（非覆盖网络），并控制支持特定微服务的一组 Pod 使用的 IP 地址范围。这使得外围防火墙可以根据 IP 地址范围执行类似于传统非 Kubernetes 工作负载的操作。例如，您需要选择一个在本地环境中支持可路由的非覆盖网络，并具有灵活 IP 地址管理能力以促进此类方法的网络插件。

+   在不实际将 Pod IP 地址路由到集群外部不可行的任何环境中（例如使用覆盖网络时），前述方法的变体非常有用。在这种情况下，从 Pod 发出的网络流量似乎来自于节点的 IP 地址，因为它使用源网络地址转换（SNAT）。为了解决这个问题，可以使用支持出口 NAT 网关的 Kubernetes 网络插件。一些 Kubernetes 网络插件支持出口 NAT 网关功能，允许更改此行为，使一组 Pod 的出口流量通过集群内执行 SNAT 的特定网关路由，因此流量似乎来自于网关，而不是托管 Pod 的节点。根据使用的网络插件，网关可以分配给特定的 IP 地址范围或特定节点，从而允许外围防火墙规则比将整个集群视为单个实体更为选择性地执行。支持此功能的几个选项包括：Red Hat 的 OpenShift SDN、Nirmata 和 Calico 都支持出口网关功能。

+   最终，一些防火墙支持一些插件或第三方工具，使防火墙能够更加了解 Kubernetes 工作负载，例如通过自动填充防火墙中的 IP 地址列表，其中包括 pod 的 IP 地址（或者承载特定 pod 的节点的节点 IP 地址）。或者在云环境中，有自动编程规则允许安全组选择性地对进出 Kubernetes pod 的流量进行操作，而不仅仅是在节点级别操作。这种集成对于你的集群非常重要，可以帮助补充防火墙提供的安全保障。市场上有几种工具可以实现这种类型的集成。选择一个支持与主要防火墙供应商的集成并且对 Kubernetes 本地化的工具非常重要。

在前面的部分，我们讨论了网络安全的重要性，以及如何利用网络安全保护来自集群外部来源的流量访问你的 Kubernetes 集群，以及如何控制从集群内部源发往集群外部主机的访问。

# 结论

在本章中，我们讨论了以下关键概念，你应该使用这些概念来确保你的 Kubernetes 集群有一个安全的基础设施。

+   你需要确保主机运行的操作系统是安全的，没有关键性漏洞。

+   你需要在主机上部署访问控制以控制对主机操作系统的访问，并部署用于进出主机的网络流量的控制。

+   你需要确保你的 Kubernetes 集群有一个安全的配置；保护数据存储和 API 服务器对于确保你拥有安全的集群配置至关重要。

+   最后，你需要部署网络安全来控制源自集群内部 pod 并且要传输到集群内部 pod 的网络流量。
