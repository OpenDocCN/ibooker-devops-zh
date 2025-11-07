# 前置材料

## 前言

云原生领域已经成熟到我们可以开始构建实用解决方案的程度。大量项目涌现出来，每个项目都专注于解决更大愿景的一部分。我们现在发现自己正在努力将这些不同的项目拼凑成一个端到端的产品。我们如何管理这些工具列表带来的固有复杂性，并构建一个完整的解决方案？

Mauricio Salatino 的《Kubernetes 平台工程》提供了全面的答案，以平台工程的形式回答了这个问题。平台工程的学科定位是通过高效且可靠的软件交付到生产环境，使云原生开发对应用开发者变得可访问。我认为平台工程是至关重要的现代学科，它将驯服复杂性并实现很久以前当 Kubernetes 首次将云原生技术带给大众时所做出的诱人承诺。

本书提供了必要的见解，说明了现代平台如何被构建以有效地整合生态系统中最有用的云原生技术，并为你的平台的应用开发者客户解决实际问题。它通过实际操作练习和示例，有效地提供了实用的指导，以培养构建有意义平台解决方案的实际技能。这些页面中的宝贵信息将使平台团队能够构建一个自助开发者平台作为他们的产品，使开发者能够以前所未有的速度和可靠性将他们的应用程序交付到生产环境中。

在我作为 Cloud Native Computing Foundation 两个不同项目的共同创建者、维护者和指导委员会成员的时间里，我亲自遇到了云原生生态系统中的许多令人惊叹的个体。从我的经验来看，毛里西奥在撰写这本有益的书籍方面处于独特的位置，这本书将引导你将这些项目整合成一个完整的平台，因为他自己一直通过在多次场合将人、社区和技术结合在一起，在生态系统中一直是一个整合者。毛里西奥展现了一种不可思议的能力，能够识别项目愿景中的共同利益，将合适的人聚集在一起，并找到统一努力的共同基础。他一直致力于他罕见的才能，通过我们的协作和协同作用而不是竞争或重复，找到让我们共同变得更好的路径。

就像毛里西奥将人和技术结合在一起一样，他也将许多项目整合成了这些页面中的一个有价值的整体。我期待这本书中学到的经验将成为你在实现平台工程愿景过程中最值得的步骤之一。请享受这段旅程！

——Jared Watts

Upbound 创始工程师

## 前言

我在两年多前开始写这本书。在为云原生社区工作超过四年之后，我学到了许多我想与团队分享的经验教训，以加快他们采用 Kubernetes 的旅程。因为我为几个开源项目做出了贡献（其中大多数包含在这本书中），为书中的想法创建目录表并不困难。另一方面，写一本关于永远在变化中的生态系统的书是具有挑战性的。但正如你阅读本书时会发现的那样，平台工程就是管理不断发展的项目和不同团队的需求的复杂性，这些团队需要正确的工具来完成他们的工作。

这本书让我有机会结识并和来自不同背景和社区的行业中最优秀的人一起工作，他们与我有着共同的激情：开源、云原生和知识分享。我环游世界，在云原生领域的会议上发表演讲，始终从社区成员、开发者和努力跟上每天创建的大量开源项目的团队那里收集反馈。我希望这本书能帮助你和你的团队评估、集成和构建在 Kubernetes 之上的平台。

## 致谢

我要特别感谢所有为本书提供的示例做出贡献的人（包括[`github.com/salaboy/from-monolith-to-k8s/`](https://github.com/salaboy/from-monolith-to-k8s/)上的原始仓库和[`github.com/salaboy/platforms-on-k8s/`](https://github.com/salaboy/platforms-on-k8s/)上的新仓库）。这本书是为并由提到的项目社区所写。

特别感谢我的兄弟 Ezequiel Salatino ([`salatino.me/`](https://salatino.me/))，他设计和构建了前端应用程序，让读者能够体验一个网站而不是一堆 REST 端点。我将永远感激 Matheus Cruz 和 Asare Nkansah，他们帮助我构建了示例的大块内容，而没有期待任何回报。最后，感谢我的朋友 Thomas Vitale，他分享了对多个版本草稿的详细评论；你所有的评论使本书的内容更加准确和专注。

没有曼宁团队提供的所有支持，我无法完成这本书。我要感谢开发编辑 Ian Hough，他为稿件投入了无数小时。采购编辑 Michael Stephens 自始至终坚信这本书的想法，Raphael Villela 作为技术编辑提供了所有技术建议，Werner Dijkerman 作为技术校对员，他的评论和确保所有代码都能良好运行。

致所有审稿人：Alain Lompo、Alexander Schwartz、Andres Sacco、Carlos Panato、Clifford Thurber、Conor Redmond、Ernesto Cárdenas Cangahuala、Evan Anderson、Giuseppe Catalano、Gregory A. Lussier、Harinath Mallepally、John Guthrie、Jonathan Blair、Kent Spillner、Lucian Torje、Michael Bright、Mladen Knezic、Philippe Van Bergen、Prashant Dwivedi、Richard Meinsen、Roman Levchenko、Roman Zhuzha、Sachin Rastogi、Simeon Leyzerzon、Simone Sguazza、Stanley Anozie、Theo Despoudis、Vidhya Vinay、Vivek Krishnan、Werner Dijkerman、WIlliam Jamir、Zoheb Ainapore，你们的建议帮助使这本书变得更好。

项目特定感谢：

+   *Argo Project* ([`argoproj.github.io/`](https://argoproj.github.io/))—我想感谢 Codefresh 的 Dan Garfield 对本书的持续支持以及他对 OpenGitOps ([`opengitops.dev/`](https://opengitops.dev/)) 初始化的贡献。

+   *Crossplane* ([`crossplane.io`](https://crossplane.io))—我想感谢 Jared Watts 不断愿意帮助并推动事物向前发展。同时，我想感谢 Viktor Farcic 和 Stefan Schimanski 对 Crossplane 社区的持续支持。Crossplane 社区教会了我许多宝贵的经验，塑造了我的职业生涯。

+   *Dagger* ([`dagger.io`](https://dagger.io))—我想感谢 Marcos Nils 和 Julian Cruciani 在 Dagger 示例方面的帮助以及他们愿意在为开发者节省时间时改进事物的意愿。

+   *Dapr* ([`dapr.io`](https://dapr.io))—对 Yaron Schneider 和 Mark Fussel 表示衷心的感谢和赞赏，他们不断支持本书的出版，以及对整个 Diagrid ([`diagrid.io`](https://diagrid.io)) 团队，他们正在 Dapr 之上构建令人惊叹的产品。

+   *Keptn* ([`keptn.sh`](https://keptn.sh))—非常感谢 Giovanni Liva 和 Andreas Grabner 的快速响应以及他们在 Keptn 和 OpenFeature 社区中做出的令人惊叹的工作。

+   *Knative* ([`knative.dev`](https://knative.dev))—整个 Knative 社区都很棒，但特别感谢 Lance Ball，他领导了 Knative Functions 工作组，构建了令人惊叹的东西。

+   *Kratix* ([`kratix.io`](https://kratix.io))—特别感谢 Abby Bangser 分享她的平台见解并审阅本书的关键章节。你所有的评论和意见使这本书的价值大大提升。

+   *OpenFeature* ([`openfeature.dev`](https://openfeature.dev))—我想感谢 James Milligan 在使 OpenFeature 和`flagd`示例工作方面的帮助。

+   *Tekton* ([`tekton.dev`](https://tekton.dev))—非常感谢 Andrea Fritolli 在 Tekton 社区中的出色工作，以及他总是及时回复我的 Slack 消息。

+   *Vcluster* ([`vcluster.com`](https://vcluster.com))—Ishan Khare 和 Fabian Kramm 对我在这本书中所做的工作至关重要。他们让事情运转起来的意愿已经超越了极限。对创建和维护`vcluster`、Devspace ([`www.devspace.sh/`](https://www.devspace.sh/))和 DevPod ([`devpod.sh/`](https://devpod.sh/))项目表示衷心的感谢。

## 关于这本书

*Kubernetes 上的平台工程*是为了帮助正在经历 Kubernetes 采用之旅的团队而编写的。本书采用以开发者为中心的方法来涵盖构建、打包和部署云原生应用到 Kubernetes 集群，但并不止于此。一旦你和你团队了解了如何为你的应用使用 Kubernetes，你将面临与管理和扩展 Kubernetes、多租户和多集群设置相关的新挑战。

在 Kubernetes 之上的平台需要集成广泛的各种工具，以使专业团队能够在执行日常任务的同时，防止他们学习所有这些工具的工作原理。平台团队负责学习、整理和集成工具，以使开发团队、数据科学家、运维团队、测试团队、产品团队以及参与你组织软件交付流程的每个人都更容易生活。

大部分内容都集中在 Kubernetes 上，并构建为对用于特定功能的技术栈无关。如果你刚开始使用 Kubernetes，或者你是一名云原生实践者，这本书可以帮助你了解如何将多个项目结合起来构建团队特定的体验，并减少你在日常工作中涉及的认知负荷，无论你和你团队使用的编程语言是什么。

### 本书是如何组织的：路线图

本书分为九章，并使用“行走骨架”的概念构建一个平台，以支持团队构建会议应用。本书的流程如下：

第一章介绍了平台是什么，为什么你需要一个，以及我们将在这本书中涵盖的平台与云提供商提供的平台相比如何。本章介绍了会议应用的商业用例，后续章节将进一步探讨。

第二章评估了在 Kubernetes 上构建云原生和分布式应用的挑战。本章鼓励读者部署会议应用，并通过更改其配置和测试不同场景来探索其设计。通过分析团队在部署和运行 Kubernetes 上的应用时将面临的挑战，并提供一个使用行走骨架进行实验的游乐场，本书旨在使有足够经验的读者能够应对更大的挑战。

第三章专注于构建、打包和分发工件所需的所有额外步骤，以便在不同的云服务提供商中运行我们的应用程序。本章介绍了服务管道的概念，并探讨了两个不同但互补的项目：Tekton 和 Dagger。

当我们的工件准备就绪，可以部署时，第四章围绕环境管道的概念展开。通过定义我们的环境管道并采用 GitOps 方法，团队可以使用声明式方法管理多个环境的配置。本章探讨了 Argo CD 作为配置和管理环境的工具。

应用程序不能独立工作。大多数应用程序需要基础设施组件，例如数据库、消息代理和身份提供者等，才能正常运行。第五章介绍了使用名为 Crossplane 的项目在云服务提供商之间部署应用程序基础设施组件的 Kubernetes 原生方法。

一旦我们处理好了构建、打包和部署我们的应用程序以及应用程序运行所需的其它组件，第六章建议读者在 Kubernetes 之上构建一个平台，使用到目前为止所学的一切，但仅关注一个简单的用例：创建开发环境。

平台不仅仅是创建环境、管理集群和部署应用程序。平台应该为团队提供定制的工作流程以提高生产力。第七章专注于通过应用级 API 使开发团队更高效，平台团队可以决定如何将这些 API 连接到可用资源。本章评估了 Dapr 和 OpenFeature 等工具，以使团队拥有更多资源，并为他们提供一个运行应用程序的地方。

虽然提高开发者的效率可以缩短软件交付时间，但如果新版本被阻塞且没有在客户面前部署，所有努力都将白费。第八章专注于展示可以在完全承诺之前对新版本进行实验的技术，即更精确地说，是发布策略。本章评估了 Knative Serving 和 Argo Rollouts，以实现不同的发布策略，让团队可以以受控的方式实验新功能。

由于平台是软件，我们需要衡量我们在演进它们时的有效性。第九章评估了两种方法来利用我们构建平台时使用的工具，并计算关键指标，这些指标允许平台工程团队评估他们的平台项目。本章探讨了 CloudEvents、CDEvents 和 Keptn 生命周期工具包作为收集事件、存储它们并聚合它们以计算有意义的指标的选择。

到本书结束时，读者将获得一个清晰的图景和实际操作经验，了解如何在 Kubernetes 之上构建平台，平台工程团队的重点是什么，以及为什么学习和跟上云原生领域的发展对于成功如此重要。

### 关于代码

本书包含许多源代码示例，既有编号列表，也有与普通文本并列。在这两种情况下，源代码都使用`固定宽度字体`（如本例所示）格式化，以将其与普通文本区分开来。有时代码也会**`加粗`**，以突出显示与章节中先前步骤相比已更改的代码，例如当新功能添加到现有代码行时。

在许多情况下，原始源代码已被重新格式化；我们添加了换行并重新调整了缩进，以适应书中的可用页面空间。在极少数情况下，即使这样也不够，列表中还包括了行续接标记（➥）。此外，当代码在文本中描述时，源代码中的注释通常也会从列表中移除。许多列表旁边都有代码注释，突出显示重要概念。

您可以从本书的在线版本（liveBook）中获取可执行的代码片段，链接为[`livebook.manning.com/book/platform-engineering-on-kubernetes`](https://livebook.manning.com/book/platform-engineering-on-kubernetes)。书中示例的完整代码可以在 Manning 网站上下载，链接为[`www.manning.com/books/platform-engineering-on-kubernetes`](https://www.manning.com/books/platform-engineering-on-kubernetes)。

每一章都链接到逐步教程，鼓励读者在自己的环境中动手操作工具和项目。您可以在以下 GitHub 仓库中找到所有源代码和逐步教程：[`github.com/salaboy/platforms-on-k8s/`](https://github.com/salaboy/platforms-on-k8s/)。

### liveBook 讨论论坛

购买《Kubernetes 平台工程》包括对 liveBook（Manning 的在线阅读平台）的免费访问。使用 liveBook 的独特讨论功能，您可以在全局或特定章节或段落中附加评论。为自己做笔记、提问和回答技术问题，以及从作者和其他用户那里获得帮助都非常简单。要访问论坛，请访问[`livebook.manning.com/book/platform-engineering-on-kubernetes/discussion`](https://livebook.manning.com/book/platform-engineering-on-kubernetes/discussion)。您还可以在[`livebook.manning.com/discussion`](https://livebook.manning.com/discussion)了解更多关于 Manning 论坛和行为准则的信息。

Manning 对我们读者的承诺是提供一个场所，在那里个人读者之间以及读者与作者之间可以进行有意义的对话。这不是对作者参与特定数量活动的承诺，作者对论坛的贡献仍然是自愿的（且未付费）。我们建议您尝试向作者提出一些挑战性的问题，以免他的兴趣转移！只要这本书还在印刷中，论坛和先前讨论的存档将可通过出版社的网站访问。

## 关于作者

![封面插图](img/Author_photo.png)

Mauricio Salatino 在 Diagrid([`diagrid.io`](https://diagrid.io))担任开源软件工程师。他目前是 Dapr OSS 贡献者和 Knative 指导委员会成员。在 Diagrid 工作之前，Mauricio 在红帽和 VMware 等公司为云原生开发者构建工具，已经过去了 10 年。当他不为开发者编写工具或为云原生领域的开源项目做出贡献时，他通过他的博客[`salaboy.com`](https://salaboy.com)和/或 LearnK8s([`learnk8s.io`](https://learnk8s.io))教授 Kubernetes 和云原生知识。

## 关于封面插图

《Kubernetes 上的平台工程》封面上的图像是“来自 Argentiera 和 Milo 群岛的女性”，或称为“来自 Argentiera 和 Milo 群岛的女性”，取自 Jacques Grasset de Saint-Sauveur 的收藏，1788 年出版。每一幅插图都是手工精心绘制和着色的。

在那些日子里，仅凭人们的服饰就能轻易地识别出他们居住的地方以及他们的职业或社会地位。Manning 通过基于几个世纪前丰富多样的地域文化的书封面，庆祝计算机行业的创新精神和主动性，这些文化通过如这一系列图片的图片得以重现。
