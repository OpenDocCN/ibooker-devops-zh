# 前置内容

## 前言

十年前，我编写了我的第一个 makefile 来自动化 C++应用程序的测试、构建和部署。三年后，在担任顾问期间，我遇到了 Jenkins 和 Docker，并发现了如何通过 CI/CD 原则将我的自动化技能提升到下一个层次。

CI/CD 的美丽之处在于，它只是记录你正在做的事情的一种严谨方式。它并没有从根本上改变你做事的方式，但它鼓励你记录开发过程中的每个步骤，使你和你团队能够在以后大规模地重现整个工作流程。在接下来的几个月里，我开始写博客文章，做演讲，并为 CI/CD 相关的工具做出贡献。

然而，对我而言，设置 CI/CD 工作流程一直是一个非常手动的过程。它是通过图形界面定义一系列针对各种管道任务的单独作业来完成的。每个作业都是通过网页表单配置的——填写文本框、从下拉列表中选择条目等等。然后，一系列作业被串联起来，每个作业触发下一个，形成一个管道。这使得故障排除体验成为噩梦，在失败的情况下回滚到最后已知配置是一项繁琐的操作。

几年后，**pipeline-as-code**实践作为更大规模的“as code”运动的一部分出现，该运动还包括**基础设施即代码**。我终于可以配置构建、测试和部署，这些都在可追踪的代码中，并存储在集中的 Git 仓库中。所有之前的痛苦都得到了缓解。

当我从软件工程师、技术领导者和高级 DevOps 经理转变为现在作为 CTO 共同领导我的第一个初创公司时，我成为了 pipeline as code 的粉丝和信徒。pipeline as code 成为我参与的每个项目的重要组成部分。

我有机会参与不同类型的架构——从单体到微服务，再到无服务器应用程序——为大规模应用程序构建和维护 CI/CD 管道。在这个过程中，我积累了在“持续一切”的旅程中遵循的技巧和最佳实践。

分享这种经验的想法触发了这本书的诞生。对于许多团队来说，实现 pipeline as code 具有挑战性，因为它们需要使用许多相互协作的工具和流程。学习曲线需要大量的时间和精力，导致人们怀疑这是否值得。这本书是一本手册，介绍了如何从头开始构建 CI/CD 管道，使用最广泛采用的 CI 解决方案：Jenkins。我希望结果能帮助你接受构建 CI/CD 管道的新范式。

## 致谢

首先，我要感谢我的妻子，Mounia。你一直支持我，在我努力完成这项工作时，你总是耐心地倾听，并总是让我相信我能完成它。我爱你。

接下来，我想感谢我的编辑 Karen Miller。感谢您与我合作，感谢您在疫情期间遇到困难时保持耐心。您对本书质量的承诺使它对每一位读者都变得更好。还要感谢所有与我一起参与本书制作和推广的其他人：Deirdre Hiam，我的项目编辑；Sharon Wilkey，我的校对编辑；Keri Hales，我的校对员；以及 Mihaela Batinić，我的审稿编辑。这确实是一个团队的努力。

最后，我想感谢我的家人，包括我的父母和兄弟，感谢他们在每次聚会中倾听我谈论这本书时所展现出的内在力量。

致所有审稿人：Alain Lompo、Alex Koutmos、Andrea Carlo Granata、Andres Damian Sacco、Björn Neuhaus、Clifford Thurber、Conor Redmond、Giridharan Kesavan、Gustavo Filipe Ramos Gomes、Iain Campbell、Jerome Meyer、John Guthrie、Kosmas Chatzimichalis、Maciej Drożdżowski、Matthias Busch、Michal Rutka、Michele Adduci、Miguel Montalvo、Naga Pavan Kumar Tikkisetty、Ryan Huber、Satej Kumar Sahu、Simeon Leyzerzon、Simon Seyag、Steve Atchue、Tahir Awan、Theo Despoudis、Ubaldo Pescatore、Vishal Singh 和 Werner Dijkerman，你们的建议帮助使这本书变得更好。

## 关于本书

*代码即管道* 被设计成通过实际示例进行动手实践。它将教会您 Jenkins 的方方面面，并成为您构建云原生应用的稳固 CI/CD 管道的最佳伴侣。

### 适合阅读本书的人群

*代码即管道* 是为所有希望提高 CI/CD 技能的 DevOps 和云实践者设计的。

### 本书组织结构

本书分为四个部分，共涵盖 14 章。

第一部分将带您了解基本的 CI/CD 原则，并讨论 Jenkins 如何帮助实现这些原则：

+   第一章概述了持续集成、部署和交付实践，并讨论了 Jenkins 如何帮助您拥抱这些 DevOps 实践。

+   第二章介绍了代码化管道方法以及如何使用 Jenkins 实现，还涵盖了声明式和脚本 Jenkins 管道之间的区别。

第二部分涵盖了如何使用基础设施即代码方法在云上部署自愈 Jenkins 集群：

+   第三章深入探讨了 Jenkins 分布式构建架构，并提供了 AWS 上的完整示例。

+   第四章介绍了使用 HashiCorp Packer 的不可变基础设施方法，包括如何烘焙一个包含所有必需依赖项的 Jenkins 机器镜像，以便直接运行 Jenkins 集群。

+   第五章演示了如何使用 HashiCorp Terraform 在 AWS 上部署安全且可扩展的 Jenkins 集群。

+   第六章详细描述了在不同云服务提供商上部署 Jenkins 集群的过程，包括 GCP、Azure 和 DigitalOcean。

第三部分专注于从头开始构建云原生应用的 CI/CD 管道，包括在 Swarm 或 Kubernetes 中运行的 Docker 化微服务和无服务器应用：

+   第七章为构建容器化微服务的 CI 工作流程奠定了基础。它涵盖了如何在 Jenkins 上定义多分支管道以及如何在推送事件上触发管道。

+   第八章演示了如何在 Docker 容器内运行自动化测试。描述了各种测试，包括使用无头 Chrome 的 UI 测试、代码覆盖率、使用 SonarQube 的静态代码分析以及安全分析。

+   第九章涵盖了在 CI 管道内构建 Docker 镜像、管理它们的版本以及扫描安全漏洞。它还讨论了如何使用 Jenkins 自动化 GitHub 拉取请求的审查。

+   第十章介绍了将 Docker 化应用部署到 Docker Swarm 的过程，并使用 Jenkins 进行演示。它展示了如何维护多个运行时环境以及如何实现持续部署和交付。

+   第十一章深入探讨了使用 Jenkins 管道自动化 Kubernetes 上应用程序部署的方法，包括如何打包和版本化 Helm 图表以及运行部署后测试。它还演示了 Jenkins X 的使用方法以及它与 Jenkins 的比较。

+   第十二章介绍了如何为基于无服务器的应用构建 CI/CD 管道，以及如何管理多个 Lambda 部署环境。

第四部分涵盖了轻松维护、扩展和监控在生产中运行的 Jenkins 集群：

+   第十三章探讨了如何使用 Prometheus、Grafana 和 Slack 构建交互式仪表板，以持续监控 Jenkins 的异常和性能问题。它还涵盖了如何将 Jenkins 日志流式传输到基于 ELK 堆栈的集中日志平台。

+   第十四章介绍了如何使用细粒度的 RBAC 机制来保护 Jenkins 作业。它还探讨了如何备份、恢复和迁移 Jenkins 作业和插件。

### 关于代码

这本书提供了一种动手实践的经验，其中包含了许多代码示例。这些代码示例遍布全文，并以单独的代码列表形式出现。代码以`固定宽度字体`的形式呈现，就像这样，所以当你看到它时就会知道。

书中使用的所有源代码均可在 Manning 网站([`www.manning.com/books/pipeline-as-code`](https://www.manning.com/books/pipeline-as-code))或我的 GitHub 仓库([`github.com/mlabouardy/pipeline-as-code-with-jenkins`](https://github.com/mlabouardy/pipeline-as-code-with-jenkins))上找到。这个仓库是我倾注心血的成果，我感谢所有捕捉到错误、进行性能改进和帮助文档工作的人。一切都非常适合贡献！

### liveBook 讨论论坛

购买 *Pipeline as Code* 包括免费访问由 Manning Publications 运营的私人网络论坛，您可以在论坛上对书籍发表评论、提出技术问题，并从作者和其他用户那里获得帮助。要访问论坛，请访问 [`livebook.manning.com/#!/book/pipeline-as-code/discussion`](https://livebook.manning.com/#!/book/pipeline-as-code/discussion)。您还可以在 [`livebook.manning.com/#!/discussion`](https://livebook.manning.com/#!/discussion) 了解更多关于 Manning 的论坛和行为准则。

Manning 对读者的承诺是提供一个场所，让读者之间以及读者与作者之间可以进行有意义的对话。这并不是对作者参与特定数量活动的承诺，作者对论坛的贡献仍然是自愿的（且未付费）。我们建议您尝试向作者提出一些挑战性的问题，以免他的兴趣转移！只要书籍仍在印刷中，论坛和先前讨论的存档将可通过出版商的网站访问。

### 其他在线资源

需要更多帮助？

+   查看我的博客 ([`labouardy.com/`](https://labouardy.com/))，我在那里定期分享关于 Jenkins 的最新消息以及构建 CI/CD 工作流的最佳实践。

+   一份每周 DevOps 新闻通讯 ([`devopsbulletin.com`](https://devopsbulletin.com)) 可以帮助您了解 pipeline-as-code 空间的最新奇事。

+   StackOverflow 上的 Jenkins 标签 ([`stackoverflow.com/questions/tagged/jenkins`](https://stackoverflow.com/questions/tagged/jenkins)) 是一个提问和帮助他人的绝佳地方。

## 关于作者

| ![](img/Labouardy.png) | **Mohamed Labouardy** 是 [Crew.work](http://crew.work) 的 CTO 和联合创始人，同时也是 DevSecOps 传教士。他是 [Komiser.io](http://komiser.io) 的创始人，也是多本关于无服务器和分布式应用的书籍的作者。他喜欢为开源项目做贡献，并经常参加会议演讲。您也可以在 Twitter 上找到他 (@mlabouardy)。 |
| --- | --- |

## 关于封面插图

*Pipeline as Code* 封面上的插图标题为“Bohémien de prague”，或称布拉格的波希米亚人。这幅插图取自雅克·格拉塞·德·圣索沃尔（1757–1810）的作品集，名为 *Costumes de Différents Pays*，于 1797 年在法国出版。每一幅插图都是手工精心绘制和着色的。格拉塞·德·圣索沃尔收藏的丰富多样性生动地提醒我们，200 年前世界的城镇和地区在文化上有多么不同。人们彼此孤立，说着不同的方言和语言。在街道或乡村，仅凭他们的服饰就能轻易识别他们居住的地方以及他们的职业或社会地位。

自那以后，我们的着装方式已经改变，而当时区域间的多样性，如此丰富，现在已经逐渐消失。现在很难区分不同大陆的居民，更不用说不同的城镇、地区或国家了。也许我们用文化多样性换取了更加丰富多彩的个人生活——当然，是更加多样化和快节奏的技术生活。

在难以区分一本计算机书与另一本的时候，曼宁通过基于两百年前丰富多样的区域生活所设计的书封面，庆祝了计算机行业的创新精神和主动性，这些画面由格拉塞·德·圣索沃尔重新赋予生命。
