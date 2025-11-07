# 前置内容

## 前言

当我完成《一个月午餐时间学会 Docker》时，我知道续集必须关于 Kubernetes。对于大多数人来说，那将是他们容器之旅的下一阶段，但学习 Kubernetes 并不容易。这部分的困难在于它是一个非常强大的平台，具有庞大的功能集，这些功能总是在不断演变。但这也因为它需要一个可靠的指南，这个指南以正确的水平进行教学——在技术知识上深入足够，但保持关注平台能为你和你的应用程序做什么。我希望《一个月午餐时间学会 Kubernetes》将是那个指南。

Kubernetes 是一个在容器中运行和管理应用程序的系统。它是生产环境中运行容器的最流行方式，因为它得到了所有主要云平台的支持，并且在数据中心中运行同样出色。这是一个世界级的平台，被 Netflix 和 Apple 等公司使用——你甚至可以在你的笔记本电脑上运行它。你必须投资学习 Kubernetes，但回报是你可以带到任何组织、任何项目的技能，并且有信心能够迅速上手。

你需要投入的是时间。熟悉 Kubernetes 能做什么以及你如何在 Kubernetes 建模语言中表达你的应用程序需要时间。你提供时间，而《一个月午餐时间学会 Kubernetes》将提供其余部分。这里有动手练习和实验室，将使你熟悉所有平台功能，以及 Kubernetes 周围的工作实践和工具生态系统。这是一本实用的书，让你准备好真正使用 Kubernetes。

## 致谢

这是我在 Manning 出版的第二本书，写作过程和第一本书一样愉快。很多人为了这本书的出版和帮助我使其更好而付出了辛勤的努力。我想感谢出版团队提供的所有反馈。

向所有审稿人，Alex Davies-Moore，Anthony Staunton，Brent Honadel，Clark Dorman，Clifford Thurber，Daniel Carl，David Lloyd，Furqan Shaikh，George Onofrei，Iain Campbell，Marc-Anthony Taylor，Marcus Brown，Martin Tidman，Mike Lewis，Nicolantonio Vignola，Ondrej Krajicek，Rui Liu，Sadhana Ganapathiraju，Sai Prasad Vaddepally，Sander Stad，Tobias Getrost，Tony Sweets，Trent Whiteley，Vadim Turkov 和 Yogesh Shetty，你们的建议帮助使这本书变得更好。

我还想感谢所有在审查周期和早期访问计划中尝试所有练习并告诉我事情何时不工作的人。感谢大家抽出时间。

## 关于这本书

## 适合阅读这本书的人

我希望你能从这本书中获得真实的 Kubernetes 体验。当你阅读完所有章节并完成练习后，你应该自信地认为你学到的技能是人们真正使用 Kubernetes 的方式。这本书包含了很多内容，你会发现典型的午餐时间对于许多章节来说是不够的。那是因为我想给每个主题都提供应有的覆盖范围，这样你才能真正全面地理解，并在完成书籍时感觉自己像一位经验丰富的 Kubernetes 专业人士。

你不需要对 Kubernetes 有任何了解就可以开始使用这本书，但你应该熟悉像容器和镜像这样的核心概念。如果你对整个容器领域是新手，你会在我的另一本书《一个月午餐时间学会 Docker》（Manning，2020）的电子书版本中找到一些章节作为附录——它们将帮助你设定场景。

Kubernetes 经验将帮助你在工程或运营领域进一步发展你的职业生涯，而且这本书对您的背景没有任何假设。Kubernetes 是一个高级主题，它建立在几个其他概念之上，但当我谈到这些概念时，我会给出它们的概述。这是一本非常实用的书，为了最大限度地利用它，你应该计划通过实际操作练习来学习。你不需要任何特殊的硬件：一台 Mac 或 Windows 笔记本电脑，或者一台 Linux 台式机，就足够了。

GitHub 是我在这本书中使用的所有样本的真实来源。你将在第一章设置你的实验室时下载这些材料，你应该确保收藏该存储库并关注通知。

## 如何使用这本书

这本书遵循了“一个月午餐时间”的原则：你应该能够在午餐时间完成每一章，并在一个月内完成整本书。这里的“工作”是关键，因为你应该考虑留出时间阅读章节，完成“现在试试”练习，并在最后尝试实际操作实验室。你应该预计会有几次较长的午餐时间，因为章节不会省略任何角落或跳过关键细节。你需要大量的肌肉记忆才能有效地与 Kubernetes 一起工作，并且每天练习将真正巩固你在每个章节中获得的知识。

### 你的学习之旅

Kubernetes 是一个庞大的主题，但我已经多年在培训课程和研讨会中教授它，无论是面对面的还是虚拟的，并建立了一个我知道有效的渐进式学习路径。我们将从核心概念开始，逐渐增加更多细节，将最复杂的话题留到你对 Kubernetes 更熟悉的时候。

第二章至第六章将跳转到在 Kubernetes 上运行应用。你将学习如何在 YAML 清单中定义应用，这些应用作为容器由 Kubernetes 运行。你将了解如何配置容器之间的网络访问以及外部世界流量如何到达你的容器。你将学习你的应用如何从 Kubernetes 读取配置并将数据写入由 Kubernetes 管理的存储单元，以及你如何扩展你的应用。

第七章至第十一章基于基础知识，涉及与实际 Kubernetes 使用相关的主题。你将学习如何运行共享相同环境的容器，以及如何使用容器来运行批处理作业和计划作业。你将学习 Kubernetes 如何支持自动化滚动更新，以便你可以零停机时间发布新的应用程序版本，以及如何使用 Helm 提供一种可配置的方式来部署你的应用。你还将了解使用 Kubernetes 构建应用的实用性，包括不同的开发者工作流程和持续集成/持续交付（CI/CD）管道。

第十二章至第十六章全部关于生产就绪，不仅限于在 Kubernetes 中运行你的应用，而是以足够好的方式运行它们以便上线。你将学习如何配置自我修复的应用程序，收集和集中所有日志，并构建监控仪表板来可视化系统的健康状况。安全也是其中之一，你将学习如何保护对应用的公共访问以及如何保护应用程序本身。

第十七章至第二十一章进入专家领域。在这里，你将学习如何处理大型 Kubernetes 部署，并配置你的应用程序以自动扩展和缩小。你将学习如何实现基于角色的访问控制来保护对 Kubernetes 资源的访问，我们还将介绍一些 Kubernetes 的更多有趣用途：作为无服务器函数的平台，以及作为一个可以运行为 Linux 和 Windows、Intel 和 Arm 构建的应用的多架构集群。

到本书结束时，你应该对将 Kubernetes 引入日常工作充满信心。最后一章提供了关于继续使用 Kubernetes 的指导，包括对书中每个主题的进一步阅读建议以及选择 Kubernetes 提供商的建议。

### 现在就试试练习

每一章都有许多指导练习供你完成。本书的源代码全部在 GitHub 上[`github.com/sixeyed/kiamol`](https://github.com/sixeyed/kiamol)——当你设置实验室环境时，你将克隆它，并使用它来运行所有示例，这将使你在 Kubernetes 中运行越来越复杂的应用程序。

许多章节都是基于书中前面的内容，但你不需要按顺序阅读所有章节，所以你可以跟随自己的学习路径。章节内的练习通常都是相互关联的，所以如果你跳过了练习，可能会在后面发现错误，这有助于磨练你的故障排除技能。所有练习都使用容器镜像，这些镜像在 Docker Hub 上公开可用，你的 Kubernetes 集群将下载它需要的任何镜像。

这本书包含了很多内容。如果你在阅读章节的同时完成示例，你会从中获得最多的收获，并且在使用 Kubernetes 时会感到更加自在。如果你没有时间完成每个练习，跳过一些是可以的；每个练习后面都附有显示你将看到的输出的截图，每个章节都以总结部分结束，以确保你对主题有信心。

### 动手实验室

每一章都以一个动手实验室结束，邀请你超越“现在试试”的练习，更进一步。这些实验室不是指导性的——你将获得一些指导和提示，然后就要靠你自己来完成实验室。所有实验室的样本答案都存放在 sixeyed/kiamol GitHub 仓库中，所以你可以检查你完成了什么——或者如果你没有时间完成某个实验室，可以看看我是如何完成的。

### 其他资源

《Kubernetes 实战》由 Marko Lukša（Manning，2017）所著，是一本很好的书，涵盖了我在这里没有涉及到的许多管理细节。除此之外，进一步阅读的主要资源是官方的 Kubernetes 文档，你可以在两个地方找到它。文档网站([`kubernetes.io/docs/home/`](https://kubernetes.io/docs/home/))涵盖了从集群架构到引导教程以及如何自己为 Kubernetes 做贡献的各个方面。Kubernetes API 参考([`kubernetes.io/docs/reference/generated/kubernetes-api/v1.20`](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20))包含了你可以创建的每种类型的对象的详细规范——这是一个需要书签的网站。

Twitter 是 Kubernetes @kubernetesio 账户的家园，你还可以关注 Kubernetes 项目和社区的一些创始人，如 Brendan Burns (@brendandburns)、Tim Hockin (@thockin)、Joe Beda (@jbeda)和 Kelsey Hightower (@kelseyhightower)。

我也经常谈论这些内容。你可以在 Twitter 上关注我@EltonStoneman；我的博客是[`blog.sixeyed.com`](https://blog.sixeyed.com)；我在[`youtube.com/eltonstoneman`](https://youtube.com/eltonstoneman)上发布 YouTube 视频。

## 关于代码

本书包含许多源代码示例，既有编号列表，也有与普通文本并列。在这两种情况下，源代码都以 `fixed-width` `font like` `this` 这样的 `固定宽度` 字体格式化，以将其与普通文本区分开来。有时代码也会用 `in` `bold` 突出显示，以强调与章节中先前步骤相比有所改变的代码，例如当新功能添加到现有代码行时。

在许多情况下，原始源代码已被重新格式化；我们添加了换行并重新调整了缩进，以适应书籍中的可用页面空间。在极少数情况下，即使这样也不够，列表中还包括了行续接标记（➥）。此外，当代码在文本中描述时，源代码中的注释通常也会从列表中删除。代码注释伴随着许多列表，突出显示重要概念。

本书示例的代码可以从 Manning 网站 [`www.manning.com/books/learn-kubernetes-in-a-month-of-lunches,`](https://www.manning.com/books/learn-kubernetes-in-a-month-of-lunches) 下载，也可以从 GitHub [`github.com/sixeyed/kiamol`](http://github.com/sixeyed/kiamol) 下载。

## liveBook 讨论论坛

购买《在一个月的午餐时间内学习 Kubernetes》包括免费访问由 Manning Publications 运营的私人网络论坛，您可以在论坛上对书籍发表评论、提出技术问题，并从作者和其他用户那里获得帮助。要访问论坛，请访问 [`livebook.manning.com/#!/book/learn-kubernetes-in-a-month-of-lunches/discussion`](https://livebook.manning.com/#!/book/learn-kubernetes-in-a-month-of-lunches/discussion)。您还可以在 [`livebook.manning.com/#!/discussion`](https://livebook.manning.com/#!/discussion) 了解更多关于 Manning 的论坛和行为准则。

Manning 对读者的承诺是提供一个场所，让读者之间以及读者与作者之间可以进行有意义的对话。这并不是对作者参与特定数量活动的承诺，作者对论坛的贡献仍然是自愿的（且未付费）。我们建议您尝试向作者提出一些挑战性的问题，以免他的兴趣转移！只要书籍在印刷中，论坛和先前讨论的存档将可通过出版社的网站访问。

## 关于作者

Elton Stoneman 是一位 Docker Captain，一位多年的 Microsoft MVP，同时也是 Pluralsight 和 Udemy 上数十门在线培训课程的作者。他在微软领域的大部分职业生涯中担任顾问，设计和交付大型企业系统。然后他迷上了容器，加入了 Docker，在那里度过了三年忙碌而充满乐趣的时光。现在，他作为一名自由顾问和培训师，帮助处于容器之旅各个阶段的组织。Elton 在 [`blog.sixeyed.com`](https://blog.sixeyed.com) 和 Twitter [@EltonStoneman](https://twitter.com/EltonStoneman) 上撰写关于 Docker 和 Kubernetes 的文章，并在 [`eltons.show`](https://eltons.show) 上定期进行 YouTube 直播。
