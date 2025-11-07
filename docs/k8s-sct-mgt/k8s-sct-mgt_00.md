# 前言

## 前言

作为技术人员，我们自然会寻求创新的方法来解决各种问题——无论是通过使用新的或现有的方法、框架或技术。过去几年中，两位作者都着迷于一种技术，那就是 Kubernetes。虽然 Docker 将容器引入了大众市场，但正是 Kubernetes 为大规模运行容器提供了一个可扩展的平台。

我们从不同的角度接近 Kubernetes：一个是从基础设施心态出发，了解构建 Kubernetes 集群需要什么，另一个则是专注于应用程序，希望利用底层基础设施提供的功能。有几个交织的主题适用于基础设施和应用导向的个人；其中一个始终如一的区域，无论是否使用 Kubernetes，那就是安全性。

安全性是那些虽然至关重要，但常常与其他兴趣领域相比被降级或忽视的话题之一。我们在与组织和开发者合作的过程中发现，他们可能不理解哪些类型的资源需要保护，或者如何进行保护。那些开始使用 Kubernetes 的人也会惊讶地发现，Kubernetes 在最初发布时在安全性方面几乎没有什么。被称为 Secrets 的保护机制是在 1.0 版本之前作为为敏感资产提供某种形式保护的一种解决方案而开发的。因此，Secrets 提供了最低级别的安全性，考虑到资源的名称，这可能让人感到惊讶。

对于应该保护哪些类型的资产以及如何保护它们的陌生感，原生 Kubernetes 功能提供的虚假安全感，以及这个领域可用的众多解决方案，这些都预示着可能是一场灾难的潜在配方。我们编写这本书的目标是强调“左移”的心态，在这种心态下，当与 Kubernetes 合作时，安全性成为一个关键关注点，并解决包含的工具、替代解决方案以及它们如何在不同的交付过程中被整合的能力和陷阱。我们并不打算——也不太可能——解决所有可能的安全选项，但 *Kubernetes Secrets Management* 中讨论的概念和实现应该能够使你在与 Kubernetes 合作时更加成功和安全。

## 致谢

在这些充满挑战的时期，我想感谢 Santa（飞吧，飞吧）、Uri（感谢所有的对话）、Guiri（Vive Le Tour）、Gavina、Gabi（感谢啤酒）、以及 Edgar 和 Ester（是的，今天是星期五）；我的工作伙伴 Edson、Sebi、Natale、Ana-Maria、Elder，当然还有 Burr 和 Kamesh（无论你在哪里，你都会在我们团队中）——我们是最棒的团队！此外，感谢 Jonathan Vila、Abel Salgado 和 Jordi Sola 关于 Java 和 Kubernetes 的精彩对话。

特别感谢所有阅读我们的手稿并提供了如此宝贵反馈的审稿人：Alain Lompo、Atila Kaya、Chris Devine、Clifford Thurber、Deepak Sharma、Giuseppe Catalano、John Guthrie、Jon Moore、Michael Bright、Mihaela Barbu、Milorad Imbra、Peter Reisinger、Robin Coe、Sameer Wadhwa、Satadru Roy、Sushant Bhadkamar、Tobias Ammann 和 Werner Dijkerman；你们的贡献帮助提高了这本书的质量。

最后但同样重要的是，我想感谢 Anna，因为她在这里；我的父母，Mili 和 Ramon，他们为我买了第一台电脑；以及我的女儿们 Alexandra 和 Ada，“我是你眼中的灵魂。”

—Alex Soto Bueno

在全球大流行期间，在处理其他责任的同时撰写一本书可能是一项具有挑战性的艰难任务。我想感谢那些帮助强化各种安全概念的人，包括 Raffaele Spazzoli、Bob Callaway 和 Luke Hinds。此外，所有在开源社区中帮助构建知识和保持联系的人。

但最重要的是，我想感谢我的父母，AnneMarie 和 A.J.，他们是我坚实的支持柱，无论逆境如何，都让我保持脚踏实地和专注。

—Andrew Block

## 关于这本书

*Kubernetes Secrets Management* 的编写旨在帮助您了解如何在开发过程中管理密钥，并将应用程序发布到 Kubernetes 集群。我们首先介绍 Kubernetes 以及设置运行书中示例的环境。介绍之后，我们讨论开发过程中如何管理密钥以及如何正确存储它们，无论是在代码仓库中还是在 Kubernetes 集群内部。最后，我们展示了在云 Kubernetes 原生方式下实现持续集成和交付以及管理管道中的密钥。

### 应该阅读这本书的人？

*Kubernetes Secrets Management* 适合那些在 Kubernetes 方面经验有限的资深开发者，他们希望扩展他们对 Kubernetes 和密钥管理的知识。这本书也适合那些想学习如何管理密钥的运维人员，包括如何配置、部署和适当地存储这些密钥。虽然网上有大量相关的文档和博客文章，但 *Kubernetes Secrets Management* 将所有这些信息汇集到一本清晰、易于遵循的文本中，以便读者可以逐步了解安全威胁以及如何应对它们。

### 本书是如何组织的：路线图

本书分为三部分，共涵盖八章：

第一部分解释了安全和秘密的基本原理，以及理解本书其余部分所必需的基本 Kubernetes 概念。

+   第一章介绍了什么是秘密，什么不是秘密，为什么保持秘密的秘密性很重要，以及 Kubernetes 的概述。

+   第二章进一步介绍了 Kubernetes，其架构以及部署包含秘密数据的应用程序的基本概念。它还讨论了为什么标准的 Kubernetes Secrets 不足以提供足够的安全性。

第二部分涵盖了在将应用程序部署到 Kubernetes 的开发和部署过程中可能遇到的安全问题以及如何解决这些问题。此外，第二部分还涵盖了使用秘密存储来管理 Kubernetes 基础设施之外的应用程序秘密。

+   第三章介绍了可以安全存储 Kubernetes Secrets 的工具和方法，并说明了声明式定义 Kubernetes 资源的好处。

+   第四章涵盖了在 Kubernetes 集群内部秘密的加密以及与密钥管理服务的集成。

+   第五章重点介绍了使用秘密管理工具（如 HashiCorp Vault）的重要性，以安全地存储和管理部署到 Kubernetes 的应用程序的敏感资产。它还演示了如何配置应用程序和 Vault 以实现无缝集成。

+   第六章在第五章介绍的外部秘密管理工具的概念基础上进行了扩展，这次聚焦于云秘密存储，包括 Google Secret Manager、Azure Key Vault 和 AWS Secrets Manager。

第三部分介绍了使用 Tekton 和 Argo CD 实现 Kubernetes 原生持续集成和持续交付的方法，并正确管理秘密。

+   第七章涵盖了快速交付高质量应用程序以尽早进入市场的方法，并且更好地管理整个管道中的秘密，以确保在开发阶段这一阶段没有秘密泄露。

+   第八章介绍了使用 Argo CD 快速交付高质量应用程序到 Kubernetes 集群的方法，通过使用 GitOps 方法部署和发布服务，在整个管道中正确管理秘密，并确保在开发阶段这一阶段没有秘密泄露。

如果读者对 Kubernetes（例如，部署、服务、卷和 ConfigMaps）和 minikube 有很好的了解，他们可以跳过第二章。

### 关于代码

本书包含许多源代码示例，既有编号列表，也有与普通文本混排。在这两种情况下，源代码都使用`固定宽度字体`格式化，以便与普通文本区分。有时代码也会**`加粗`**，以突出显示章节中从先前步骤更改的代码，例如当新功能添加到现有代码行时。

在许多情况下，原始源代码已被重新格式化；我们添加了换行符并重新整理了缩进，以适应书籍中的可用页面空间。在极少数情况下，即使这样也不够，列表中还包括了行续接标记（➥）。此外，当代码在文本中描述时，源代码中的注释通常已从列表中删除。许多列表都有代码注释，突出显示重要概念。

您可以从本书的 liveBook（在线）版本中获取可执行的代码片段 [`livebook.manning.com/book/kubernetes-secrets-management`](https://livebook.manning.com/book/kubernetes-secrets-management)。书中示例的完整代码可在 Manning 网站 www.manning.com 和 GitHub [`github.com/lordofthejars/kubernetes-secrets-source`](https://github.com/lordofthejars/kubernetes-secrets-source) 上下载。

### liveBook 讨论论坛

购买 *Kubernetes Secrets Management* 包括对 Manning 的在线阅读平台 liveBook 的免费访问。使用 liveBook 的独特讨论功能，您可以在全球范围内或特定章节或段落中添加评论。为自己做笔记、提问和回答技术问题，以及从作者和其他用户那里获得帮助都非常简单。要访问论坛，请访问 [`livebook.manning.com/book/kubernetes-secrets-management/discussion`](https://livebook.manning.com/book/kubernetes-secrets-management/discussion)。您还可以在 [`livebook.manning.com/discussion`](https://livebook.manning.com/discussion) 了解更多关于 Manning 论坛和行为准则的信息。

Manning 对读者的承诺是提供一个场所，让读者之间以及读者与作者之间可以进行有意义的对话。这不是对作者参与特定数量承诺的承诺，作者对论坛的贡献仍然是自愿的（且未付费）。我们建议您尝试向作者提出一些挑战性的问题，以免他们的兴趣分散！只要本书有售，论坛和先前讨论的存档将可通过出版商的网站访问。

## 关于作者

![](img/Alex_Sotobueno_photo.png)

Alex Soto Bueno 是 Red Hat 的开发者体验总监。他对 Java 世界、软件自动化充满热情，并相信开源软件模式。Alex 是 *Testing Java Microservices*、*Quarkus Cookbook*、*Securing Kubernetes Secrets* 的合著者，并为多个开源项目做出了贡献。自 2017 年以来，Alex 一直是 Java 冠军，也是国际演讲者、Onda Cero 的广播合作者，以及 Salle URL 大学教师。您可以通过 Twitter (@alexsotob) 关注 Alex，以了解 Kubernetes 和 Java 世界正在发生的事情。

![](img/Andrew_Block_photo.png)

安德鲁·布洛克（Andrew Block）是红帽（Red Hat）的杰出架构师，他与组织合作设计和实施利用云原生技术的解决方案。他专注于持续集成和持续交付方法，以减少交付时间并自动化环境的构建和维护。他还是《Learn Helm》一书的合著者，该书介绍了如何在 Kubernetes 环境中打包应用程序进行部署。安德鲁也是几个开源项目的贡献者，并强调共同努力构建和维护实践社区的好处。您可以在 Twitter 上找到安德鲁，用户名是@sabre1041，他经常分享该领域最新和最好的头条新闻和技巧。

## 关于封面插图

《Kubernetes Secrets Management》封面上的图像被标注为“Femme Mokschane”，或“Mokshan woman”，取自雅克·格拉塞·德·圣索沃尔（Jacques Grasset de Saint-Sauveur）的作品集，该作品集于 1797 年出版。每一幅插图都是手工精心绘制和着色的。

在那些日子里，仅凭人们的服饰就能轻易识别出他们居住的地方以及他们的职业或社会地位。曼宁通过基于几个世纪前丰富多样的地域文化的书封面，庆祝计算机行业的创新精神和主动性，这些文化通过如这一系列图片的图片被重新带回生活。
