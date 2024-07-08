# 第一章：Kubernetes 和 OpenShift 概述

过去几年来，Kubernetes 已经成为管理、编排和提供基于容器的云原生计算应用的事实标准平台。*云原生计算应用*基本上是由一组较小的服务（微服务）构建而成的应用程序，并利用云计算环境通常提供的开发速度和可扩展性能力。随着时间的推移，Kubernetes 已经成熟到可以提供管理更高级和有状态工作负载所需的控制，如数据库和 AI 服务。Kubernetes 生态系统继续经历爆炸式增长，并且该项目因为是一个基于多供应商和技术精英的开源项目，背靠坚实的治理政策以及为贡献者提供公平竞争的平台而受益良多。

尽管有许多 Kubernetes 发行版供客户选择，但 Red Hat OpenShift Kubernetes 发行版特别引人注目。OpenShift 已经在各行各业广泛采用；全球超过一千家企业客户目前使用它来托管其业务应用程序并推动数字转型努力。

本书侧重于让您成为在生产环境中运行传统 Kubernetes 和 OpenShift Kubernetes 发行版的专家。在本章中，我们从广义上概述了 Kubernetes 和 OpenShift 的历史起源。然后，我们回顾了使 Kubernetes 和 OpenShift 成为创建和部署云原生应用程序的主导平台的关键特性和能力。

# Kubernetes：编排容器化应用的云基础设施

Docker 在 2013 年的出现向许多开发人员介绍了容器和基于容器的应用程序开发。容器被提出作为创建自包含部署单元的虚拟机（VM）的替代方案。容器依赖于 Linux 操作系统的先进安全性和资源管理功能，以在进程级别提供隔离，而不是依赖于 VM 来创建可部署的软件单元。Linux 进程在启动应用程序镜像或创建新的镜像快照等常见活动中比 VM 轻量得多，效率更高几个数量级。由于这些优势，开发人员青睐容器作为创建新软件应用程序的自包含部署软件单元的理想方法。随着容器的流行，对于在集群中提供、管理和编排容器的共同平台的需求也在增长。十多年来，谷歌一直支持使用 Linux 容器作为其云中部署应用程序的基础。^(1) 谷歌在规模上编排和管理容器方面拥有丰富的经验，并开发了三代容器管理系统：Borg、Omega 和 [Kubernetes](https://kubernetes.io)。作为基于 Borg 和 Omega 经验教训重新设计的最新一代容器管理系统，Kubernetes 作为开源项目提供。Kubernetes 提供了几个关键特性，显著改进了开发和部署可扩展的基于容器的云应用程序的体验：

声明式部署模型

Kubernetes 发布之前存在的大多数云基础设施采用了基于脚本语言（如 Ansible、Chef、Puppet 等）的过程化方法来自动化将应用程序部署到生产环境。相比之下，Kubernetes 使用了声明式方法来描述系统的期望状态。Kubernetes 基础设施负责在需要时启动新的容器（例如容器失败时）以达到所声明的期望状态。声明式模型在传达所需的部署操作方面更加清晰，与试图读取和解释脚本来确定所需部署状态相比，这一方法是一个巨大的进步。

内置的副本和自动伸缩支持

在一些存在于 Kubernetes 之前的云基础设施中，应用程序的复制和自动扩展能力并不是核心基础设施的一部分，并且在某些情况下，由于平台或架构限制，从未真正实现。*自动扩展* 是指云环境能够识别应用程序的使用情况增加，并自动增加应用程序的容量，通常是通过在云环境中的额外服务器上创建更多应用程序副本来实现的。自动扩展功能作为 Kubernetes 的核心特性提供，显著改进了其编排能力的健壮性和可用性。

内置的滚动升级支持

大多数云基础设施不提供应用程序升级支持。相反，它们假设操作员将使用像 Chef、Puppet 或 Ansible 这样的脚本语言来处理升级。与此不同的是，Kubernetes 实际上提供了内置支持，用于应用程序的滚动升级。例如，Kubernetes 的升级可配置为利用额外资源进行更快的无停机滚动，或者执行较慢的滚动来进行金丝雀测试，通过将软件释放给少数用户来减少风险并验证新软件，确保应用程序的新版本是稳定的。Kubernetes 还支持暂停、恢复和回滚应用程序的版本。

改进的网络模型

Kubernetes 将单个 IP 地址映射到一个 *pod*，这是 Kubernetes 最小的容器部署、聚合和管理单元。这种方法使网络身份与应用程序身份保持一致，并简化了在 Kubernetes 上运行软件的过程。^(2)

内置的健康检查支持

Kubernetes 提供了容器健康检查和监控功能，降低了识别故障发生时间的复杂性。

尽管 Kubernetes 提供了许多创新功能，但许多企业公司仍然对采用这项技术持谨慎态度，因为它是由单一供应商支持的开源项目。企业公司对于愿意采纳的开源项目非常谨慎，他们希望像 Kubernetes 这样的开源项目有多个供应商参与贡献；他们还期望开源项目是基于精英治理并且具有公平竞争的政策。2015 年，云原生计算基金会（CNCF）成立以解决 Kubernetes 面临的这些问题。

# CNCF 加速 Kubernetes 生态系统的增长

2015 年，Linux 基金会发起了 CNCF 的创建。^(3) CNCF 的使命是使云原生计算普及化。^(4) 为了支持这个新基金会，谷歌将 Kubernetes 捐赠给 CNCF 作为其种子技术。以 Kubernetes 为生态系统核心，CNCF 已经发展到拥有超过 440 家成员公司，包括谷歌云、IBM 云、红帽、亚马逊网络服务（AWS）、Docker、微软 Azure、VMware、英特尔、华为、思科、阿里云等。^(5) 此外，CNCF 生态系统已经托管了 26 个开源项目，包括 Prometheus、Envoy、gRPC、etcd 等。最后，CNCF 还培育了几个早期项目，并已有八个项目被接纳到其用于新兴技术的 Sandbox 计划中。

在厂商中立的 CNCF 基金会的支持下，Kubernetes 每年的贡献者已经超过 3,200 人，来自各行各业。^(6) 除了托管几个云原生项目外，CNCF 还提供培训、技术监督委员会、治理委员会、社区基础设施实验室以及几个认证项目，以促进 Kubernetes 和相关项目的生态系统。由于这些努力，目前已经有超过一百个经过认证的 Kubernetes 发行版。其中，最受欢迎的 Kubernetes 发行版之一，尤其是企业客户，就是红帽的 OpenShift Kubernetes。在接下来的部分中，我们将介绍 OpenShift，并概述它为开发人员和 IT 运营团队提供的关键优势。

# OpenShift：红帽的 Kubernetes 发行版

尽管许多公司都为 Kubernetes 做出了贡献，但红帽的贡献尤为引人注目。红帽从 Kubernetes 作为开源项目诞生之初就成为其生态系统的一部分，并且一直是 Kubernetes 的第二大贡献者。基于对 Kubernetes 的实践经验，红帽提供了自己的 Kubernetes 发行版，称为*OpenShift*。OpenShift 是企业中部署最广泛的 Kubernetes 发行版。它提供了一个符合标准的 Kubernetes 平台，并通过各种工具和功能来提高开发人员和 IT 运营的生产力。

OpenShift 最初发布于 2011 年。[⁷] 那时，它有自己的平台特定的容器运行时环境。[⁸] 2014 年初，红帽团队与谷歌的容器编排团队会面，了解到了一个新的容器编排项目，最终成为 Kubernetes。红帽团队对 Kubernetes 非常印象深刻，因此 OpenShift 被重写以使用 Kubernetes 作为其容器编排引擎。由于这些努力，OpenShift 能够在其 2015 年 6 月的版本 3 发布中交付一个完全符合标准的 Kubernetes 平台。[⁹]

红帽 OpenShift 容器平台是 Kubernetes 带有额外支持功能的实现，使其符合企业需求。Kubernetes 社区为发布提供长达 12 个月的修复补丁支持。OpenShift 通过提供长期支持（三年或更长时间）的主要 Kubernetes 发布版本、安全补丁以及覆盖操作系统和 OpenShift Kubernetes 平台的企业支持合同，使其在各个分布中区别于其他发行版。红帽企业 Linux（RHEL）长期以来一直是各大小组织的事实上 Linux 发行版。红帽 OpenShift 容器平台在 RHEL 的基础上构建，以确保从主机操作系统到集群上所有容器化功能的一致 Linux 发行。除了所有这些好处之外，OpenShift 通过提供各种工具和能力来增强 Kubernetes，重点是提高开发人员和 IT 运营的生产力。以下各节描述了这些好处。

## 开发者的 OpenShift 好处

虽然 Kubernetes 在为容器镜像的供应和管理提供了许多功能，但它并未提供太多支持来自基础镜像创建新镜像、将镜像推送到注册表或识别新版本可用的功能。此外，Kubernetes 提供的网络支持使用起来可能相当复杂。为了填补这些空白，OpenShift 为开发者提供了超越核心 Kubernetes 平台所提供的几个优势：

源到镜像

当使用基本 Kubernetes 时，云原生应用程序开发人员负责创建自己的容器镜像。通常，这涉及找到合适的基础镜像，并创建一个包含所有必要命令的 `Dockerfile`，以便将基础镜像与开发人员的代码合并，创建一个 Kubernetes 可以部署的组装镜像。这要求开发人员学习一系列用于镜像组装的 Docker 命令。通过其源码到镜像（S2I）能力，OpenShift 能够处理将云原生开发人员的代码合并到基础镜像中。在许多情况下，可以配置 S2I，使开发人员只需将其更改提交到 Git 仓库，S2I 将看到更新的更改，并将其与基础镜像合并以创建新的组装镜像以供部署。

推送镜像到注册表

当云原生开发人员在基本 Kubernetes 中使用时，必须执行的另一个关键步骤是将新组装的容器镜像存储在像 Docker Hub 这样的镜像注册表中。在这种情况下，开发人员需要创建和管理仓库。相比之下，OpenShift 提供了自己的私有注册表，开发人员可以选择使用该选项，或者可以配置 S2I 将组装后的镜像推送到第三方注册表。

图像流

当开发人员创建云原生应用程序时，开发工作会导致大量配置更改，以及对应用程序容器镜像的更改。为了解决这种复杂性，OpenShift 提供了图像流功能，该功能监视配置或图像更改，并根据更改事件执行自动构建和部署。这一功能减轻了开发人员在发生更改时手动执行这些步骤的负担。

基础镜像目录

OpenShift 提供了一个基础镜像目录，其中包含大量有用的基础镜像，适用于各种工具和平台，如 WebSphere Liberty、JBoss、PHP、Redis、Jenkins、Python、.NET、MariaDB 等。该目录提供了来自已知源代码打包的受信内容。

路由

在基础 Kubernetes 中配置网络可能会相当复杂。OpenShift 具有路由结构，与 Kubernetes 服务进行交互，并负责将 Kubernetes 服务添加到外部负载均衡器中。路由还为应用程序提供可读的 URL，并提供多种负载均衡策略，以支持蓝绿、金丝雀和 A/B 测试部署等多种部署选项。^(10)

虽然 OpenShift 对开发人员有很多好处，但其最大的差异化优势在于它为 IT 运维提供的好处。在下一节中，我们描述了运行 OpenShift 在生产环境中自动化日常运营的几个核心能力。

## OpenShift 对 IT 运维的益处

2019 年 5 月，Red Hat 宣布发布 OpenShift 4。¹¹ Red Hat 收购了 CoreOS，后者对管理 Kubernetes 的生命周期行为具有非常自动化的方法，并且早期倡导“运营者”概念。这个新版本的 OpenShift 完全重写，借鉴了 CoreOS 创新的管理实践和 OpenShift 3 在可靠性方面的声誉，大大改善了 OpenShift 平台的安装、升级和管理方式。^(11) 为了提供这些重要的生命周期改进，OpenShift 在其架构中大量使用了最新的 Kubernetes 创新和最佳实践来自动化管理资源。因此，OpenShift 4 能够为 IT 运维带来以下好处：

自动化安装

OpenShift 4 支持创新的自动化安装方法，可靠且可重复。^(12) 此外，OpenShift 4 安装过程支持完全堆栈的自动化部署，并能处理安装完整基础设施，包括 DNS 和虚拟机等组件。

自动化操作系统和 OpenShift 平台更新

OpenShift 与轻量级的 RHEL CoreOS 操作系统紧密集成，后者专为运行 OpenShift 和云原生应用程序进行了优化。由于 OpenShift 与特定版本的 RHEL CoreOS 紧密耦合，OpenShift 平台能够在其集群管理操作中管理操作系统更新。对 IT 运维的关键价值在于，它支持自动化、自我管理的远程更新。这使得 OpenShift 能够支持云原生和无人值守的运维操作。

自动化集群大小管理

OpenShift 支持自动增减其管理的集群大小。与所有 Kubernetes 集群一样，OpenShift 集群有一定数量的工作节点，用于部署容器应用程序。在典型的 Kubernetes 集群中，增加工作节点是一个 IT 运维必须手动处理的操作。相比之下，OpenShift 提供了一个称为*机器操作器*的组件，能够自动向集群添加工作节点。IT 运维人员可以使用`MachineSet`对象声明集群所需的机器数量，OpenShift 将自动执行新工作节点的供应和安装，以达到所需状态。

自动化集群版本管理

OpenShift，像所有的 Kubernetes 发行版一样，由大量组件组成。每个组件都有自己的版本号。为了管理这些组件的更新，OpenShift 依赖于 Kubernetes 创新称为*operator construct*。OpenShift 使用集群版本号来标识正在运行的 OpenShift 版本，该集群版本号表示需要安装哪些个别 OpenShift 平台组件的版本。通过其自动化的集群版本管理，OpenShift 能够在集群更新到新版本时自动安装所有这些组件的适当版本，以确保更新正确进行。

多云管理支持

许多使用 OpenShift 的企业客户拥有多个集群，这些集群部署在多个云中或多个数据中心中。为了简化多个集群的管理，OpenShift 4 引入了一个新的统一云控制台，允许客户查看和管理多个 OpenShift 集群。^(13)

正如我们将在本书后面看到的那样，当到达运行和 IT 运营人员需要解决运营和安全相关问题的时候，OpenShift 及其提供的功能变得非常显著。

# 摘要

在本章中，我们概述了 Kubernetes 和 OpenShift，包括这两个平台的历史起源。然后，我们介绍了 Kubernetes 和 OpenShift 提供的主要优势，这些优势推动了这些平台在全球范围内的巨大增长。这帮助我们更加重视 Kubernetes 和 OpenShift 为云原生应用开发者和 IT 运营团队提供的价值。因此，毫不奇怪这些平台在多个行业中经历了爆炸式增长。在第二章，我们将建立一个扎实的基础概述 Kubernetes 和 OpenShift，介绍 Kubernetes 架构，讨论如何在生产环境中运行 Kubernetes 和 OpenShift，并介绍几个关键的 Kubernetes 和 OpenShift 概念，这些对成功运行至关重要。

^(1) Brendan Burns 等，“Borg、Omega 和 Kubernetes：从十年三个容器管理系统的经验中汲取教训”，*ACM Queue* 14（2016 年）：70–93，[*http://bit.ly/2vIrL4S*](http://bit.ly/2vIrL4S)。

^(2) Brendan Burns 等，“Borg、Omega 和 Kubernetes：从十年三个容器管理系统的经验中汲取教训”，*ACM Queue* 14（2016 年）：70–93，[*http://bit.ly/2vIrL4S*](http://bit.ly/2vIrL4S)。

^(3) Steven J. Vaughan-Nicholls，“云原生计算基金会寻求在云和容器领域实现统一”，*ZDNet*（2015 年 7 月 21 日），[*https://oreil.ly/WEoE0*](https://oreil.ly/WEoE0)。

^(4) Linux Foundation，CNCF Chapter（更新于 2018 年 12 月 10 日），[*https://oreil.ly/tHHvr*](https://oreil.ly/tHHvr)。

^(5) [CNCF 会员页面](https://oreil.ly/Tj3Vw)提供了 CNCF 成员增长的更多详情。

^(6) 查看[Kubernetes 公司表格仪表板](https://oreil.ly/mkSTm)以获取当前列表。

^(7) Joe Fernandes，“为什么 Red Hat 选择 Kubernetes 作为 OpenShift”，红帽 OpenShift 博客（2016 年 11 月 7 日），[*https://oreil.ly/r66GM*](https://oreil.ly/r66GM)。

^(8) Anton McConville 和 Olaph Wagoner，“Kubernetes、OpenShift 和 IBM 的简史”，IBM Developer 博客（2019 年 8 月 1 日），[*https://oreil.ly/IugtP*](https://oreil.ly/IugtP)。

^(9) “Red Hat 提供 OpenShift Enterprise 3 以支持新的 Web 规模分布式应用平台” [新闻稿]，红帽（2015 年 6 月 24 日），[*https://oreil.ly/jlane*](https://oreil.ly/jlane)。

^(10) 欲了解更多有关 OpenShift 路由的详情，请参阅[使用基于路由的部署策略](https://oreil.ly/hmpJz)的 OpenShift 文档。

^(11) Joe Fernandes，“介绍 Red Hat OpenShift 4：企业级 Kubernetes”，红帽 OpenShift 博客（2019 年 5 月 8 日），[*https://oreil.ly/yNb8s*](https://oreil.ly/yNb8s)。

^(12) Christian Hernandez，“OpenShift 4.1 Bare Metal Install Quickstart”，红帽 OpenShift 博客（2019 年 7 月 31 日），[*https://oreil.ly/yz4pR*](https://oreil.ly/yz4pR)。

^(13) Fernandes，“介绍 Red Hat OpenShift 4：企业级 Kubernetes”。
