# 前言

## 前言

我是在 2017 年开始我的 Kubernetes 之旅的，当时我在德克萨斯州奥斯汀的一家小型金融科技公司工作。我被要求将服务器从 Amazon EC2（亚马逊弹性云计算）迁移到 AKS（Azure Kubernetes 服务）。我之前从未听说过 Kubernetes，但管理层的决定是坚定的，所以我被推到了火里。我疯狂地浏览文档网站、博客、书籍以及我能找到的任何东西，试图吸收这项新技术。我记得我害怕参加站立会议，因为我只是让自己更加困惑，并没有取得任何实质性的进展。我陷入了困境，需要一些帮助。我需要的是比我稍微领先一点的人——一个对 Kubernetes 足够了解，能帮我摆脱困境的人。

在这次经历不久之后，我将帮助他人成功学习 Kubernetes 作为我的目标。恰好这和第一次认证 Kubernetes 管理员（CKA）考试的时间相吻合，这次考试为正确管理 Kubernetes 必须具备的技能提供了一个蓝图。我在 2018 年创建了第一个 CKA 准备课程，并从那时起一直在分享我的 Kubernetes 知识。今天，我很高兴地说，我已经通过我的课程和内容帮助了成千上万的人。这本书是我兴奋地分享我的知识和经验的一种新媒介。

这本书是关于通过 CKA 考试的完整指南，包含了场景、练习和课程，帮助你练习并彻底吸收内容。CKA 考试与其他很多考试不同，你将在第一章中发现这一点。通过持续的练习和努力，你将为 CKA 考试做好充分的准备。

我自 2018 年以来一直持有 CKA 认证，并在过去一年内再次参加考试以获得再认证，并将最新的信息传递给你，这样你就有最好的机会获得你的证书。我祝愿你在考试中一切顺利，但你可以确信，在阅读这本书后，你已经做好了充分的准备。你一定能做到！

## 致谢

与 Manning 合作撰写书籍是一次非常广泛且令人大开眼界的经历。我对 Manning 对每本书所采取的勤奋过程深感钦佩和尊重，这无疑为这本书增添了巨大的价值。没有我的开发编辑 Connor O’Brien，我无法完成这项工作。通过他的仔细检查和深思熟虑的审查，我学到了很多。

我要感谢 Curtis Bates，他是这本书的技术编辑，拥有 25 年在分布式计算、云和 HPC 领域担任软件架构师、系统工程师和软件开发者的经验。

致所有审稿人——Alessandro Campeis、Amit Lamba、Bradford Hysmith、Dale Francis、Dan Sheikh、David Moravec、Dylan Scott、Emanuele Piccinelli、Ernesto Cárdenas Cangahuala、Frankie Thomas-Hockey、Ganesh Swaminathan、Giampiero Granatella、Giang Châu、Ioannis Polyzos、John Harbin、Joseph Perenia、Kamesh Ganesan、Michael Bright、Michele Adduci、Morteza Kiadi、Roman Levchenko、Shawn Bolan、Simeon Leyzerzon、Simon Tschöke、Stanley Anozie、Swapneelkumar Deshpande 和 Tim Sina——谢谢你们，你们的建议帮助使这本书变得更好。

我还想感谢我的妻子，Georgianne，因为她鼓励我写作，并在许多夜晚和周末照顾孩子，而我则在最近的咖啡馆里敲击键盘。

## 关于这本书

### 适合阅读这本书的人

这本书是为那些寻求获得认证的人准备的，当然，但它也适合那些希望遵循经过验证和认可的 Kubernetes 学习方法的人。通过遵循考试蓝图，你会很高兴地发现这本书包含了成为 Kubernetes 管理员所需的所有必要组件。话虽如此，这不是一本 Kubernetes 入门书籍，它将要求你了解如何导航 Linux 操作系统，并具备对 Kubernetes 及其旨在解决的问题的基本理解。

Kubernetes 经验将帮助你进一步发展你的职业生涯，因为它越来越多地被许多财富 500 强公司采用。这个高级技能集需求很高，在你的简历上拥有这个认证将为你的简历增添巨大的价值，这大大增加了增加你薪资的可能性。

### 这本书是如何组织的：路线图

这本书非常实用，就像考试一样。你应该计划完成实际操作练习，对于这些练习，你不需要任何特殊硬件——一个 Mac、Windows 或 Linux 桌面就足够了。我们将使用 kind Kubernetes 设置本地集群；这些设置说明可以在附录 A 中找到。当你完成练习时，你将在附录 D 中找到如何解决它们的额外信息。

本书非常紧密地遵循云原生计算基金会（CNCF）的考试标准，因为你会希望确保所有领域都被涵盖，并且你为考试日做好了充分的准备。本书从对考试的介绍开始，介绍考试是什么，以及如何准备考试，然后第二章从集群架构、安装和配置的 CKA 考试能力开始。接着，本书继续介绍工作负载和调度，然后是服务和网络、存储，最后是故障排除。

第二章和第三章直接进入部署 Kubernetes 集群的基础设施配置，使用 kubeadm 在 Kubernetes 集群上进行版本升级，实施 etcd 备份和恢复，管理基于角色的访问控制，以及管理高可用性 Kubernetes 集群，以全面覆盖集群架构、安装和配置考试标准。

第四章和第五章在前面章节的基础上，通过配置运行在 Kubernetes 之上的应用程序，使用 ConfigMaps 和 Secrets，并继续解释资源限制如何影响 Pod 调度。你将了解如何使用清单管理和常见的模板工具，以及用于在 Kubernetes 中创建自愈应用程序的原语。然后，你将了解如何扩展在 Kubernetes 中运行的应用程序，以及更新 Deployments 如何与滚动更新和回滚协同工作。为了完善你的知识，你将了解如何创建、更新和管理容器化应用程序，这将完成考试的工作负载和调度领域。

第六章帮助你理解 Kubernetes 中的网络工作方式，包括集群节点上的网络配置和 Pod 之间的连接性。你将了解 Kubernetes 中服务类型，包括 ClusterIP、NodePort 和 LoadBalancer。你将了解 Ingress 控制器和 Ingress 资源在 Kubernetes 中的工作方式，以及如何配置和使用 CoreDNS。为了完善本章，你将能够选择合适的容器网络接口插件，这将完成服务和网络考试标准。

第七章探讨了卷和存储在 Kubernetes 中的工作方式，包括存储类、持久卷和持久卷声明。你将了解卷的模式、访问模式和回收策略。你还将学习如何配置具有持久存储的应用程序，这将完善考试中的存储领域。

第八章转向解决 Kubernetes 集群中的问题。你将学习如何评估集群和节点日志，以及如何监控在 Kubernetes 中运行的应用程序。你还将学习如何管理容器标准输出（标准输出）和标准错误（标准错误）日志，以及如何解决应用程序故障、集群组件故障和网络问题。这将完全满足考试标准的故障排除部分。

第九章是逐章复习，旨在为你提供考试前一晚的复习材料，以防你需要最后时刻快速回顾书中的任何内容。这种复习还可以帮助你建立信心，因为你可以在检查完第九章的每一节后感觉准备得更好。不要忘记每个章节中的练习！到本书结束时，你应该对自己的 Kubernetes 集群管理充满信心，并准备好在不久之后参加考试。

### 关于练习

在每一章中，你将看到各种练习，这些练习将测试你对上一节所读内容的理解。我鼓励你独立完成这些练习，在你的本地集群上，或者如果你想通过浏览器访问免费的集群，请访问[`killercoda.com`](https://killercoda.com)。如果你需要帮助，请参阅附录 D，了解如何解决所有练习。在终端中练习这些练习，除了主要示例场景外，对于准备考试日也将至关重要。

### 关于代码

在某些情况下，我可能会引用 YAML 文件或配置文件，所有这些文件都在以下 GitHub 仓库中：[`github.com/chadmcrowell/acing-the-cka-exam`](https://github.com/chadmcrowell/acing-the-cka-exam)。这本书包含许多源代码示例，这些代码以`固定宽度字体`的形式呈现，以区别于普通文本。在许多情况下，原始源代码已被重新格式化；我们添加了换行符并重新调整了缩进，以适应书籍中的可用页面空间，并且一些列表中包含行续接标记（➥）。

你可以从这本书的 liveBook（在线）版本中获取可执行的代码片段，网址为[`livebook.manning.com/book/acing-the-certified-kubernetes-administrator-exam`](https://livebook.manning.com/book/acing-the-certified-kubernetes-administrator-exam)。书中示例的完整代码可以从 Manning 网站[`www.manning.com/books/acing-the-certified-kubernetes-administrator-exam`](https://www.manning.com/books/acing-the-certified-kubernetes-administrator-exam)和 GitHub[`github.com/chadmcrowell/acing-the-cka-exam`](https://github.com/chadmcrowell/acing-the-cka-exam)下载。

### liveBook 讨论论坛

购买 *通过认证的 Kubernetes 管理员考试* 包括对 liveBook（在线阅读平台）的免费访问。使用 liveBook 的独家讨论功能，你可以在全球范围内或针对特定章节或段落附加评论。为自己做笔记、提问和回答技术问题，以及从作者和其他用户那里获得帮助都非常简单。要访问论坛，请访问[`livebook.manning.com/book/acing-the-certified-kubernetes-administrator-exam/discussion`](https://livebook.manning.com/book/acing-the-certified-kubernetes-administrator-exam/discussion)。你还可以在[`livebook.manning.com/discussion`](https://livebook.manning.com/discussion)了解更多关于 Manning 论坛和行为准则的信息。

Manning 对读者的承诺是提供一个平台，让读者之间以及读者与作者之间可以进行有意义的对话。这并不是对作者参与特定数量活动的承诺，作者对论坛的贡献仍然是自愿的（且未付费）。我们建议你尝试向作者提出一些挑战性的问题，以免他们的兴趣转移！只要这本书还在印刷中，论坛和以前讨论的存档将可通过出版商的网站访问。

### 其他在线资源

在第一章中包含了你可以在考试期间打开的资源，这些资源可以补充考试准备。你将能够在虚拟机内的浏览器中访问以下文档及其子域名：[`kubernetes.io/docs/`](https://kubernetes.io/docs/) 和 [`kubernetes.io/blog/`](https://kubernetes.io/blog/)。这包括这些页面的所有可用语言翻译（例如，[`kubernetes.io/zh/docs/`](https://kubernetes.io/zh/docs/)）。所有这些资源都将帮助你准备考试，并且熟悉这些有用的工具对于考试当天将大有裨益。例如，Kubernetes 文档网站有一个搜索功能，你可以使用某些关键词快速导航到资源，这在第九章中有更深入的介绍。

## 关于作者

![Chad Crowell 照片](img/chad-crowell-photo.png)

Chad M. Crowell 是 Raft 公司的 DevSecOps 工程师，同时也是微软认证培训师。Chad 已经与 A Cloud Guru 和 INE 等公司合作发布了八门关于 DevOps 和 Kubernetes 的课程。目前，他领导着一个名为 KubeSkills 的社区，通过辅导和小组学习的方式帮助人们学习 Kubernetes 和容器。你可以在 [`community.kubeskills.com`](https://community.kubeskills.com/) 的 KubeSkills 社区中找到 Chad 的帖子。他还在 YouTube 上有账号 [`YouTube.com/@kubeskills`](https://YouTube.com/@kubeskills)，并在 Twitter 上以 @chadmcrowell 为名。

## 关于封面插图

封面上的图片是来自 *《通过认证 Kubernetes 管理员考试》* 的“Tehinguise ou Danseuse Turcque”，或称为“土耳其舞者”，这幅画取自雅克·格拉塞·德·圣索沃尔（Jacques Grasset de Saint-Sauveur）的作品集，该作品集于 1788 年出版。每一幅插图都是手工精心绘制和着色的。

在那些日子里，人们通过他们的服饰很容易就能识别出他们住在哪里，以及他们的职业或社会地位。Manning 通过基于几个世纪前丰富多样的地区文化，并由像这样的作品集中的图片重新呈现的封面，来庆祝计算机行业的创新精神和主动性。
