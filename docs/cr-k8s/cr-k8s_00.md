# 前置材料

## 前言

我们编写这本书是为了赋予那些想要将他们的 K8s（Kubernetes）知识提升到下一个层次的人力量，通过立即深入研究与存储、网络和工具相关的各种主题的模糊细节。

尽管我们不试图提供 K8s API 中每个功能的全面指南（因为这是不可能的），但我们真的相信，在阅读这本书之后，用户将对如何在生产集群中推理复杂的基础设施相关问题以及如何在更广泛的环境中思考 Kubernetes 生态系统的整体发展有新的直觉。

存在一些书籍可以让用户学习 Kubernetes 的基础知识，但我们想编写一本教授构成 Kubernetes 核心技术的书籍。网络、控制平面和其他主题以低级细节进行覆盖，这将帮助您理解 Kubernetes 内部的工作方式。了解系统的工作原理将使您成为一名更好的 DevOps 或软件工程师。

我们还希望在这个过程中激励新的 Kubernetes 贡献者。请通过 Twitter（@jayunit100，@chrislovecnm）联系我们，以更多地参与更广泛的 Kubernetes 社区或帮助我们向与本书相关的 GitHub 存储库添加更多示例。

## 致谢

我们想感谢维护 Kubernetes 的社区和企业。没有他们及其持续的工作，这款软件将不会存在。我们可以提及许多人的名字，但我们知道我们可能会遗漏一些。

我们想感谢 SIG Network（Mikael Cluseau、Khaled Hendiak、Tim Hockins、Antonio Ojea、Ricardo Katz、Matt Fenwick、Dan Winship、Dan Williams、Casey Calendero、Casey Davenport、Andrew Sy 以及许多其他人）中的朋友和导师；SIG Network 和 SIG Windows 社区不知疲倦的开源贡献者（Mark Rosetti、James Sturevant、Claudio Belu、Amim Knabben）；Kubernetes 的原始创始人（Joe Beda、Brendan Burns、Ville Aikas 和 Craig McLuckie）；以及随后加入他们的 Google 工程师，包括 Brian Grant 和 Tim Hockin。

这份致谢包括社区牧羊人 Tim St. Clair、Jordan Liggit、Bridget Kromhaut 以及许多其他人。我们还想感谢 Rajas Kakodar、Anusha Hedge 和 Neha Lohia，他们组建了一个新兴的 SIG Network India 团队，这个团队激励了我们希望添加到本书下一版（或可能的续集）中的大量内容，当我们深入网络或服务器代理 `kube-proxy` 时。

Jay 还想感谢 Clint Kitson 和 Aarthi Ganesan，他们赋予他在 VMware 员工的身份下从事这本书的工作，以及他在 VMware 的团队（Amim 和 Zac），他们始终在创新，并一直支持我们。当然，还有 Frances Buran、Karen Miller 以及许多帮助我们将这本书审查并投入生产的 Manning Publications 的工作人员。

最后，感谢所有审稿人：Al Krinker、Alessandro Campeis、Alexandru Herciu、Amanda Debler、Andrea Cosentino、Andres Sacco、Anupam Sengupta、Ben Fenwick、Borko Djurkovic、Daria Vasilenko、Elias Rangel、Eric Hole、Eriks Zelenka、Eugen Cocalea、Gandhi Rajan、Iryna Romanenko、Jared Duncan、Jeff Lim、Jim Amrhein、Juan José Durillo Barrionuevo、Matt Fenwick、Matt Welke、Michał Rutka、Riccardo Marotti、Rob Pacheco、Rob Ruetsch、Roman Levchenko、Ryan Bartlett、Ubaldo Pescatore 和 Wesley Rolnick。你们的建议帮助使这本书变得更好。

## 关于这本书

### 适合阅读这本书的人

想要了解更多关于 Kubernetes 内部结构、如何推理其故障模式以及如何扩展以实现自定义行为的人将能从这本书中获得最大收益。如果你不知道 Pod 是什么，你可能想买这本书，但首先应该购买另一本能够给你提供这种理解的书。

此外，对于那些希望更好地理解与 IT 部门、CTO 和其他组织领导者讨论如何采用 Kubernetes 所需方言的日常操作员，同时保留在容器诞生之前就存在的核心基础设施原则，会发现这本书真正有助于弥合新旧基础设施设计决策之间的差距。或者至少，这是我们希望的结果！

### 本书是如何组织的：路线图

本书包含 15 章：

+   第一章：在这里，我们为新用户提供 Kubernetes 的高级概述。

+   第二章：我们探讨 Pod 作为应用程序的原子构建块的概念，并介绍后续章节将深入探讨的底层 Linux 细节的合理性。

+   第三章：这是我们深入探讨如何使用底层 Linux 原语在 Kubernetes 中构建高级概念，包括 Pod 实现的细节。

+   第四章：我们现在全力以赴深入 Linux 进程和隔离的内部细节，这些是 Kubernetes 景观中一些不太为人所知的细节。

+   第五章：在覆盖 Pod 细节（主要是）之后，我们深入探讨 Pod 的网络，并查看它们如何在不同的节点之间连接起来。

+   第六章：这是我们的第二篇关于网络的文章，我们探讨了 Pod 和网络代理（`kube-proxy`）网络的更广泛方面，以及如何对其进行故障排除。

+   第七章：这是我们关于存储的第一章，它对 Kubernetes 存储的理论基础、CSI（容器存储接口）以及它与 kubelet 的交互进行了广泛的介绍。

+   第八章：在我们的第二章中，我们探讨存储的一些更实用的细节，包括 emptyDir、Secrets 和 PersistentVolumes/dynamic 存储等是如何工作的。

+   第九章：我们现在深入探讨 kubelet，并查看它如何启动 Pod 以及管理它们的细节，包括对 CRI、节点生命周期和 ImagePullSecrets 等概念的分析。

+   第十章：Kubernetes 中的 DNS 是一个复杂的话题，几乎在所有基于容器的应用程序中都被用来本地访问内部服务。我们探讨了 CoreDNS，这是 Kubernetes 的 DNS 服务实现，以及不同的 Pod 如何满足 DNS 请求。

+   第十一章：我们在早期章节中提到的控制平面，现在将详细讨论，包括调度器、控制器管理器和 API 服务器的工作概述。这些构成了 Kubernetes 的“大脑”，在涉及前几章讨论的较低级别概念时，将它们全部整合在一起。

+   第十二章：因为我们已经涵盖了控制平面逻辑，我们现在深入 etcd，这是 Kubernetes 的坚如磐石的共识机制，以及它是如何发展到满足 Kubernetes 控制平面需求的。

+   第十三章：我们概述了 NetworkPolicies、RBAC 以及 Pod 和节点级别的安全性，这对于生产场景中的管理员来说应该了解。本章还讨论了 Pod 安全策略 API 的整体进展。

+   第十四章：在这里，我们探讨节点级别的安全性、云安全性以及 Kubernetes 安全性的其他基础设施相关方面。

+   第十五章：我们以一个通用的应用工具概述作为结束，以 Carvel 工具包为例，该工具包用于管理 YAML 文件，构建类似 Operator 的应用程序，以及管理应用程序的长期生命周期。

### 关于代码

我们在 GitHub 仓库 ([`github.com/jayunit100/k8sprototypes/`](https://github.com/jayunit100/k8sprototypes/)) 中为本书提供了几个示例，特别是在以下方面

+   使用 `kind` 在本地集群上安装 Calico、Antrea 或 Cillium 的真实网络

+   在现实世界中查看 Prometheus 指标

+   使用 Carvel 工具包构建应用程序

+   各种 RBAC 相关实验

本书还提供了许多代码示例。这些示例遍布文本中，并以单独的代码片段形式出现。代码以 `fixed-width` `font` `like` `this` 的形式出现，因此您在看到它时就会知道。

在许多情况下，原始源代码已被重新格式化；我们添加了换行并重新调整了缩进，以适应书籍中的可用页面空间。在极少数情况下，即使这样也不够，代码片段中包括行续行标记（➥）。代码注释伴随许多列表，突出显示重要概念。您可以从本书的 liveBook（在线）版本中获取可执行的代码片段，网址为 [`livebook.manning.com/book/core-kubernetes`](https://livebook.manning.com/book/core-kubernetes)，以及 GitHub 上的 [`github.com/jayunit100/k8sprototypes/`](https://github.com/jayunit100/k8sprototypes/)。

### liveBook 讨论论坛

购买 *Core Kubernetes* 包含对 liveBook 的免费访问，Manning 的在线阅读平台。使用 liveBook 的独家讨论功能，您可以在全球范围内或特定章节或段落中附加评论。为自己做笔记、提问和回答技术问题，以及从作者和其他用户那里获得帮助都非常简单。要访问论坛，请访问 [`livebook.manning.com/book/core-kubernetes/discussion`](https://livebook.manning.com/book/core-kubernetes/discussion)。您还可以在 [`livebook.manning.com/discussion`](https://livebook.manning.com/discussion) 上了解更多关于 Manning 论坛和行为准则的信息。

Manning 对读者的承诺是提供一个平台，让读者之间以及读者与作者之间可以进行有意义的对话。这不是对作者参与特定数量活动的承诺，作者对论坛的贡献仍然是自愿的（且未付费）。我们建议您尝试向作者提出一些挑战性的问题，以免他们的兴趣分散！只要本书有售，论坛和先前讨论的存档将可通过出版商的网站访问。

## 关于作者

![FM_F02_Vyas](img/FM_F02_Vyas.png)

Jay Vyas，博士，目前是 VMware 的员工工程师，曾参与过多个商业和开源 Kubernetes 发行版和平台，包括 OpenShift、VMware Tanzu、Black Duck 的内部多角度 Kubernetes 安装平台，以及他咨询公司 Rocket Rudolf, LLC 的客户定制的 Kubernetes 安装。在过去的几年里，他作为 Apache 软件基金会 PMC（项目管理委员会）的成员，在 BigData 领域的多个项目中工作。自 Kubernetes 诞生以来，他一直在不同角色中参与 Kubernetes，目前大部分时间都在 SIG-Windows 和 SIG-network 社区中度过。他在完成生物信息学数据集市（将联邦数据库集成到人类和病毒蛋白质组挖掘平台）的博士学业期间开始了分布式系统的研究。这使他进入了大数据领域，并扩展数据处理系统，最终转向 Kubernetes。

如果您有兴趣在 . . . 任何事情上合作，可以在 Twitter 上联系 Jay，用户名 @jayunit100。他的日常锻炼是一英里冲刺和直到力竭的引体向上。他还拥有几台合成器，包括 Prophet-6，其声音听起来像一艘宇宙飞船。

![FM_F02_Love](img/FM_F02_Love.png)

克里斯·洛夫是一位谷歌云认证专家，也是 Lionkube 的联合创始人。他在包括谷歌、甲骨文、VMware、思科、强生和其他公司在内的公司拥有超过 25 年的软件和 IT 工程经验。作为 Kubernetes 和 DevOps 社区中的思想领袖，克里斯·洛夫为许多开源项目做出了贡献，包括 Kubernetes、kops（前 AWS SIG 负责人）、Bazel（为 Kubernetes 规则做出贡献）和 Terraform（VMware 插件的早期贡献者）。他的专业兴趣包括 IT 文化转型、容器化技术、自动化测试框架和实践、Kubernetes、Golang（又称 Go）和其他编程语言。洛夫还喜欢在世界各地演讲关于 DevOps、Kubernetes 和技术，以及指导 IT 和软件行业的人士。

在工作之外，洛夫喜欢滑雪、排球、瑜伽以及其他与科罗拉多州生活相关的户外活动。他也是一名超过 20 年的武术实践者。

如果你想与克里斯进行虚拟咖啡或对他有疑问，你可以在 Twitter 或 LinkedIn 上通过@chrislovecnm 联系他。

## 关于封面插图

《核心 Kubernetes》封面上的图像是“斯特恩，掌舵的船员”，取自阿尔弗雷多·卢索罗的一幅画作雕刻，发表于《意大利插画》第 19 期，1880 年 5 月 9 日。

在那些日子里，仅凭人们的服饰就能轻易识别出他们居住的地方以及他们的职业或社会地位。曼宁通过基于几个世纪前丰富多样的地域文化的书封面，庆祝当今计算机行业的创新精神和主动性，这些文化通过如这幅雕刻画一样的图片被重新呈现出来。
