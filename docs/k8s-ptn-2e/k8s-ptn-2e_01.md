# 前言

近年来，微服务和容器的主流采用彻底改变了软件设计、开发和运行的方式。今天的应用程序优化了可用性、可扩展性和上市速度。在新需求的推动下，现代应用程序需要一组不同的模式和实践。本书旨在帮助开发人员探索和学习使用 Kubernetes 创建云原生应用程序的最常见模式。首先，让我们简要了解本书的两个主要组成部分：Kubernetes 和设计模式。

# Kubernetes

*Kubernetes* 是一个容器编排平台。Kubernetes 的起源可以追溯到谷歌数据中心，谷歌的内部容器编排平台 [Borg](https://oreil.ly/x12HH)。

Kubernetes 从一开始就吸引了大量用户社区，并且贡献者数量增长迅速。今天，Kubernetes 被认为是 GitHub 上最受欢迎的项目之一。可以说，Kubernetes 是最常用和功能最丰富的容器编排平台。Kubernetes 也构成了其他基于它构建的平台的基础。其中最著名的 Platform-as-a-Service 系统之一是 Red Hat OpenShift，为 Kubernetes 提供各种附加功能。这些只是我们选择 Kubernetes 作为本书云原生模式参考平台的一些原因。

这本书假设你已经掌握了一些 Kubernetes 的基础知识。在第一章，我们回顾了核心 Kubernetes 概念，并为后续的模式奠定了基础。

# 设计模式

*设计模式*的概念可以追溯到 20 世纪 70 年代，起源于建筑领域。建筑师和系统理论家克里斯托弗·亚历山大及其团队在 1977 年出版了开创性作品[*一种模式语言*](https://oreil.ly/TKzwz)（牛津大学出版社），描述了用于创建城镇、建筑和其他建设项目的建筑模式。后来，这个想法被新兴的软件行业采纳。在这个领域最著名的书籍是由 Erich Gamma、Richard Helm、Ralph Johnson 和 John Vlissides 合著的[*设计模式——可复用面向对象软件的元素*](https://oreil.ly/k5toF)（Addison-Wesley），被称为四人帮。当我们谈论著名的单例、工厂或委托模式时，正是因为这部定义性作品。自那以后，许多其他优秀的模式书籍已经为不同领域撰写，具有不同的粒度级别，例如由 Gregor Hohpe 和 Bobby Woolf（Addison-Wesley）编写的[*企业集成模式*](https://oreil.ly/5aRjR)或由 Martin Fowler（Addison-Wesley）编写的[*企业应用架构模式*](https://oreil.ly/yOdWA)。

简而言之，*模式*描述了*问题的可重复解决方案*。^(1) 这个定义适用于我们在本书中描述的模式，除了我们的解决方案可能没有那么多的变化。模式不同于配方，因为它不是提供解决问题的逐步说明，而是提供解决整个类似问题的蓝图。例如，亚历山大模式*啤酒厅*描述了应如何建造公共饮酒厅，其中“陌生人和朋友是喝酒的伙伴”，而不是“孤独者的锚”。按照这种模式建造的所有大厅看起来都不同，但共享共同特征，如为四到八人的小组提供的开放式凹室，以及一百人可以聚会享受饮料、音乐和其他活动的地方。

但是，一个模式不仅仅提供了一个解决方案。它还涉及形成一种语言。这本书中的模式形成了一种密集的、以名词为中心的语言，其中每个模式都有一个独特的*名称*。当这种语言确立时，这些名称在人们谈论这些模式时会自动唤起类似的心理表征。例如，当我们谈论一张桌子时，任何讲英语的人都会认为我们在谈论一块有四条腿和一个顶部的木头，你可以放东西的地方。在软件工程中，当讨论“工厂”时也会发生同样的事情。在面向对象编程语言的背景下，我们立刻将“工厂”与产生其他对象的对象联系起来。因为我们立刻知道模式背后的解决方案，我们可以继续解决尚未解决的问题。

模式语言还具有其他特征。例如，模式是相互关联的，可以重叠，从而覆盖大部分问题空间。另外，正如原著《模式语言》中所阐述的，模式具有不同的粒度和范围。更通用的模式涵盖广泛的问题空间，并提供解决问题的粗略指导。细粒度的模式提供非常具体的解决方案建议，但适用范围较窄。本书包含各种模式，许多模式相互引用，甚至可能包含其他模式作为解决方案的一部分。

模式的另一个特征是它们遵循严格的格式。然而，每个作者定义不同的形式；不幸的是，并没有关于如何布置模式的共同标准。Martin Fowler 在《编写软件模式》中对模式语言使用的格式给出了很好的概述。

# 本书结构

我们选择了本书的简单模式格式。我们没有遵循任何特定的模式描述语言。对于每个模式，我们使用以下结构：

名称

每个模式都有一个名称，也是章节的标题。名称是模式语言的核心。

问题

本节提供了更广泛的背景和详细描述模式空间。

解决方案

本节展示了模式如何以 Kubernetes 特定方式解决问题。本节还包含了与其他相关模式的交叉引用，这些模式或者相关，或者作为给定模式的一部分。

讨论

本节包括关于给定上下文中解决方案优缺点的讨论。

更多信息

这一最终部分包含了与模式相关的其他信息来源。

我们按以下方式组织了本书中的模式：

+   第一部分，“基础模式”，涵盖了 Kubernetes 的核心概念。这些是构建基于容器的云原生应用程序的基本原则和实践。

+   第二部分，“行为模式”，描述了建立在基础模式之上的模式，并添加了管理各种类型容器的运行时概念。

+   第三部分，“结构模式”，包含与在 Kubernetes 平台的原子 Pod 中组织容器相关的模式。

+   第四部分，“配置模式”，深入探讨了在 Kubernetes 中处理应用程序配置的各种方式。这些是细粒度的模式，包括了连接应用程序到其配置的具体方法。

+   第五部分，“安全模式”，讨论了将应用程序容器化并部署在 Kubernetes 上时出现的各种安全问题。

+   第六部分，“高级模式”，包含高级概念的集合，例如平台本身如何扩展或如何在集群内直接构建容器镜像。

根据上下文的不同，相同的模式可能适用于多个类别。每个模式章节都是独立完整的；您可以单独阅读章节，且顺序不限。

# 本书适合读者：

本书面向希望设计和开发云原生应用，并将 Kubernetes 作为平台的*开发者*。对于那些对容器和 Kubernetes 概念有基本了解，并希望深入学习的读者来说，本书最为适合。然而，要理解用例和模式，并不需要了解 Kubernetes 的低级细节。架构师、顾问和其他技术人员也会从这里描述的可重复使用模式中受益。

本书基于真实项目的用例和经验教训。它是多年工作积累的最佳实践和模式的结晶。我们希望帮助您理解以 Kubernetes 为先的思维方式，并创建更好的云原生应用，而不是重复造轮子。它以轻松的风格撰写，类似于一系列可独立阅读的文章。

让我们简要看看本书*不包括*的内容：

+   本书不是 Kubernetes 的介绍，也不是参考手册。我们涉及许多 Kubernetes 特性，并对其进行一些详细解释，但我们专注于这些特性背后的概念。第一章，“介绍”，简要回顾 Kubernetes 基础知识。如果您正在寻找一本全面的 Kubernetes 书籍，我们强烈推荐由 Marko Lukša（Manning Publications）撰写的 *Kubernetes in Action*。

+   本书不是关于如何逐步设置 Kubernetes 集群的指南。每个示例假设您已经搭建好 Kubernetes。您可以尝试多种示例。如果您有兴趣学习如何设置 Kubernetes 集群，我们推荐阅读[*Kubernetes: Up and Running*](https://learning.oreilly.com/library/view/kubernetes-up-and/9781098110192)，由 Brendan Burns、Joe Beda、Kelsey Hightower 和 Lachlan Evenson（O’Reilly）共同撰写。

+   本书不涉及为其他团队操作和管理 Kubernetes 集群。我们故意跳过 Kubernetes 的管理和运维方面，并从开发者的视角深入探讨 Kubernetes。本书可以帮助运维团队了解开发者如何使用 Kubernetes，但不足以进行 Kubernetes 集群的管理和自动化。如果您有兴趣学习如何操作 Kubernetes 集群，我们推荐阅读[*Kubernetes 最佳实践*](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/)，由 Brendan Burns、Eddie Villalba、Dave Strebel 和 Lachlan Evenson（O’Reilly）共同撰写。

# 您将学到什么

这本书中有很多发现。一些模式乍一看可能像是 Kubernetes 手册的摘录，但仔细观察后，你会发现这些模式是从概念角度呈现的，这在其他相关主题的书籍中找不到。其他模式则通过详细的步骤解释如何解决具体问题，就像第四部分，“配置模式”中的情况一样。在一些章节中，我们解释了 Kubernetes 的一些特性，这些特性不太适合作为模式定义的一部分。不要过分纠结它是一个模式还是一个特性。在所有章节中，我们从第一原理出发看待所涉及的力量，并专注于用例、经验教训和最佳实践。这才是其中的有价值的部分。

无论模式的粒度如何，你都将学习每个特定模式所提供的 Kubernetes 的全部内容，同时提供大量例子来说明概念。所有这些例子都经过测试，我们告诉你如何获取完整的源代码在“使用代码示例”中。

# 第二版中的新内容

自四年前第一版问世以来，Kubernetes 生态系统一直在持续增长。因此，已经发布了许多 Kubernetes 版本，并且用于使用 Kubernetes 的工具和模式已经成为事实上的标准。

幸运的是，我们书中描述的大部分模式经受住了时间的考验并仍然有效。因此，我们更新了这些模式，增加了适用于 Kubernetes 1.26 版本的新功能，并移除了过时和废弃的部分。大部分情况下，只需要进行小的改动，除了第二十九章，“弹性扩展”，以及第三十章，“镜像构建器”，由于这些领域的新发展，它们经历了重大变化。

另外，我们还包含了五种新的模式，并引入了一个新的类别，第五部分，“安全模式”，这填补了第一版的空白，并为开发人员提供了重要的安全相关模式。

我们的 GitHub [示例](https://oreil.ly/kXGjC) 已经更新并扩展了。最后，我们为读者增加了 50%的内容供他们享受。

# 本书中使用的约定

本书中使用了以下排版约定：

*斜体*

指示新术语、URL、电子邮件地址、文件名和文件扩展名。

`固定宽度`

用于程序清单，以及段落中引用程序元素，如变量或函数名、数据库、数据类型、环境变量、语句和关键字。

正如前文所述，模式形成了一个简单而相互连接的语言。为了强调这种模式的网络，每个模式都是大写并设置为斜体（例如，*Sidecar*）。当一个模式名称同时也是 Kubernetes 的核心概念（比如 *Init Container* 或 *Controller*）时，我们仅在直接引用模式本身时使用这种特定格式。在有意义的情况下，我们还会将模式章节进行相互链接以便于导航。

我们还使用以下约定：

+   你在 Shell 或编辑器中输入的所有内容都将以`constant width font`字体呈现。

+   Kubernetes 资源名称总是以大写字母显示（例如，Pod）。如果资源是像 ConfigMap 这样的组合名称，则保留其原样，以清楚表明它指的是 Kubernetes 概念，而不是更自然的“config map”。

+   有时，Kubernetes 资源名称与“service”或“node”等常见概念完全相同。在这些情况下，我们仅在引用资源本身时使用资源名称格式。

###### 提示

这个元素表示一个提示或建议。

###### 注意

这个元素表示一般性说明。

###### 警告

这个元素表示一个警告或注意事项。

# 使用代码示例

每个模式都配备了完全可执行的示例，你可以在附带的[网页](https://k8spatterns.io)中找到它们。你可以在每章的“更多信息”部分找到每个模式示例的链接。

“更多信息”部分还包含与模式相关的进一步信息的链接。我们会在示例存储库中更新这些列表。

本书中所有示例代码的源代码都可以在[GitHub](https://oreil.ly/bmj-Y)上找到。存储库和网站还提供指针和说明，说明如何获取一个 Kubernetes 集群来尝试这些示例。请查看提供的资源文件，它们包含许多有价值的注释，可以进一步帮助理解示例代码。

许多示例使用一个名为*random-generator*的 REST 服务，在调用时返回随机数。它是专门为本书的示例设计的。你可以在[GitHub](https://oreil.ly/WuYSu)找到其源代码，其容器镜像`k8spatterns/random-generator`托管在[Docker Hub](https://oreil.ly/N36MB)上。

我们使用 JSON 路径表示法来描述资源字段（例如，`.spec.replicas`指向资源的`spec`部分的`replicas`字段）。

如果你在示例代码或文档中发现问题，或者有问题，请不要犹豫在[GitHub 问题跟踪器](https://oreil.ly/hCnmn)上开启一个工单。我们会监视这些 GitHub 问题，并乐意回答任何问题。

所有示例代码都按照[知识共享署名 4.0 国际许可证（CC BY 4.0）](https://oreil.ly/QuiQc)进行分发。该代码可供商业和非商业项目使用、分享和适应。但是，如果您复制或重新分发示例代码，应该将归属归还给本书。

此归属可以是对书籍的引用，包括书名、作者、出版社和 ISBN，例如“*Kubernetes Patterns*，第 2 版，作者 Bilgin Ibryam 和 Roland Huß（O’Reilly）。版权所有 2023 年 Bilgin Ibryam 和 Roland Huß，978-1-098-13168-5。” 或者，附上一个链接到[相关网站](https://k8spatterns.io)，并加上版权声明和许可证链接。

我们也欢迎代码贡献！如果您认为我们可以改进我们的示例，我们很乐意听取您的意见。只需打开一个 GitHub 问题或创建一个拉取请求，让我们开始交流。

# O’Reilly 在线学习

###### 注意

近 40 年来，[*O’Reilly Media*](http://oreilly.com)为企业提供技术和商业培训、知识和见解，帮助公司取得成功。

我们独特的专家和创新者网络通过图书、文章、会议和我们的在线学习平台分享他们的知识和专长。O’Reilly 的在线学习平台为您提供按需访问实时培训课程、深入学习路径、交互式编码环境以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。有关更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将有关本书的评论和问题发送至出版商：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   加利福尼亚州塞巴斯托波尔 95472

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为这本书创建了一个网页，列出勘误、示例和其他信息。您可以访问此页面[*https://oreil.ly/kubernetes_patterns-2e*](https://oreil.ly/kubernetes_patterns-2e)。

发送电子邮件至*bookquestions@oreilly.com*以评论或询问有关本书的技术问题。

有关我们的图书和课程的新闻和信息，请访问[*https://oreilly.com*](https://oreilly.com)。

在 LinkedIn 上找到我们：[*https://linkedin.com/company/oreilly-media*](https://linkedin.com/company/oreilly-media)

在 Twitter 上关注我们：[*https://twitter.com/oreillymedia*](https://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*https://youtube.com/oreillymedia*](https://youtube.com/oreillymedia)

在 Twitter 上关注作者：[*https://twitter.com/bibryam*](https://twitter.com/bibryam)，[*https://twitter.com/ro14nd*](https://twitter.com/ro14nd)

在 Mastodon 上关注作者：[*https://fosstodon.org/@bilgin*](https://fosstodon.org/@bilgin)，[*https://hachyderm.io/@ro14nd*](https://hachyderm.io/@ro14nd)

在 GitHub 上找到作者：[*https://github.com/bibryam*](https://github.com/bibryam)，[*https://github.com/rhuss*](https://github.com/rhuss)

关注他们的博客：[*https://www.ofbizian.com*](https://www.ofbizian.com)，[*https://ro14nd.de*](https://ro14nd.de)

# 致谢

Bilgin 对他美妙的妻子 Ayshe 永远感激，感谢她在他又一本书的创作过程中给予的无尽支持和耐心。他也感谢他可爱的女儿 Selin 和 Esin，她们总是知道如何让他笑容满面。你们对他来说意义非凡。最后，Bilgin 要感谢他的梦幻般的合著者 Roland，使这个项目成为现实。

Roland 深深感谢妻子 Tanja 在写作过程中始终如一的支持和包容，他还感谢儿子 Jakob 的鼓励。此外，Roland 希望特别感谢 Bilgin，因为他出色的见解和写作让这本书得以完成。

创建这本书的两个版本是一段漫长的多年旅程，我们要感谢那些在这条正确道路上一直陪伴我们的审阅者。

对于第一版，特别感谢 Paolo Antinori 和 Andrea Tarocchi 在整个旅程中对我们的帮助。非常感谢 Marko Lukša、Brandon Philips、Michael Hüttermann、Brian Gracely、Andrew Block、Jiri Kremser、Tobias Schneck 和 Rick Wagner，他们用他们的专业知识和建议支持了我们。最后但同样重要的是，要感谢我们的编辑 Virginia Wilson、John Devins、Katherine Tozer、Christina Edwards 以及 O’Reilly 的所有出色人士，他们帮助我们把这本书完成到最后。

完成第二版绝非易事，我们感谢所有在完成中支持我们的人。我们要特别感谢我们的技术审阅者 Ali Ok、Dávid Šimanský、Zbyněk Roubalík、Erkan Yanar、Christoph Stäbler、Andrew Block 和 Adam Kaplan，以及我们的开发编辑 Rita Fernando，在整个过程中她们的耐心和鼓励。我们还要向 O’Reilly 的生产团队表示赞赏，特别是 Beth Kelly、Kim Sandoval 和 Judith McConville，在完成书稿时他们的细致关注。

我们要特别感谢 Abhishek Koserwal 在《第二十六章，“访问控制”》（ch26.html#AccessControl）中不懈而专注的努力。他的贡献正是我们最需要的时候，并产生了深远影响。

^(1) Alexander 和他的团队在建筑学的原始含义中定义如下：“每个模式描述了在我们的环境中反复出现的问题，然后描述了解决该问题的核心方法，使得您可以无数次地使用这个解决方案，而不必完全相同地再做一次。”（*A Pattern Language*，Christopher Alexander 等人，1977 年。）
