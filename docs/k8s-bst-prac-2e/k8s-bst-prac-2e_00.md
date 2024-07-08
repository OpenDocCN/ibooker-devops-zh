# 前言

# 谁应该阅读本书

Kubernetes 是云原生开发的事实标准。它是一个强大的工具，可以使您的下一个应用程序更易于开发，部署更快速，运行更可靠。然而，要发挥 Kubernetes 的潜力，需要正确使用它。本书适用于那些正在将真实应用程序部署到 Kubernetes 上，并有兴趣了解他们可以应用到基于 Kubernetes 构建的应用程序的模式和实践的任何人。

重要的是，本书不是 Kubernetes 的入门。我们假设您对 Kubernetes API 和工具有基本的了解，并且知道如何创建和与 Kubernetes 集群交互。如果您想学习 Kubernetes，有许多很好的资源可以帮助您，比如 [*Kubernetes: Up and Running*](https://oreil.ly/ziNRK)（O'Reilly），可以为您提供介绍。

相反，本书是一个资源，适用于任何想要深入了解如何在 Kubernetes 上部署特定应用程序和工作负载的人。无论您是准备在 Kubernetes 上部署第一个应用程序，还是已经使用 Kubernetes 多年，本书都应该对您有所帮助。

# 为什么写这本书

我们四人中，都有帮助各种用户将其应用程序部署到 Kubernetes 上的重要经验。通过这些经验，我们看到了人们在哪些地方遇到困难，并帮助他们找到成功的方法。在撰写本书时，我们试图记录这些经验，以便更多的人通过阅读我们从这些实际经验中学到的教训来学习。我们希望通过将我们的经验写下来，能够扩展我们的知识，并使您能够成功地独立部署和管理您的应用程序在 Kubernetes 上。

# 浏览本书

尽管您可能会一口气读完这本书的每一页，但这并不是我们设计您使用它的方式。相反，我们设计本书为独立的章节集合。每一章节都提供了完成特定任务的完整概述，这些任务您可能需要在 Kubernetes 上完成。我们期望人们深入研究本书以了解特定主题或兴趣，然后让本书独自放置，只有在出现新主题时才返回。

尽管这本书采用独立的方法，但一些主题贯穿全书。有几章关于在 Kubernetes 上开发应用程序。第二章涵盖开发者工作流程。第五章讨论持续集成和测试。第十五章涵盖在 Kubernetes 上构建高级平台，而第十六章讨论管理状态和有状态应用程序。除了开发应用程序外，还有几章关于在 Kubernetes 中操作服务。第一章涵盖基本服务的设置，第三章涵盖监控和指标。第四章涵盖配置管理，而第六章涵盖版本控制和发布。第七章涵盖在全球范围内部署您的应用程序。

本书还有几章关于集群管理，包括第八章关于资源管理，第九章关于网络，第十章关于 Pod 安全，第十一章关于策略和治理，第十二章关于多集群管理，以及第十七章关于准入控制和授权。最后，一些章节是真正独立的；这些章节涵盖机器学习（第十四章）和与外部服务集成（第十三章）。

尽管在真实世界中尝试实际主题之前阅读所有章节可能会有所帮助，但我们主要希望您将本书视为实践中这些主题的指南。

# 本版新增内容

我们希望通过增加四个新章节来补充第一版，这些章节涵盖随着 Kubernetes 的成熟而不断发展的工具和模式的最佳实践。这些新章节分别是第十八章关于 GitOps，第十九章关于安全，第二十章关于混沌测试，以及第二十一章关于实施运算符。

# 本书中使用的约定

本书中使用以下排版约定：

*斜体*

表示新术语、URL、电子邮件地址、文件名和文件扩展名。

`常量宽度`

用于程序清单，以及段落内引用程序元素，例如变量或函数名称、数据库、数据类型、环境变量、语句和关键字。

**`常数宽度粗体`**

显示用户应按字面输入的命令或其他文本。

*`常数宽度斜体`*

显示应由用户提供值或由上下文确定值的文本。

###### 小贴士

这个元素表示小贴士或建议。

###### 注意

这个元素表示一般注意。

###### 警告

这个元素表示警告或注意。

# 使用代码示例

补充材料（代码示例、练习等）可在[*https://oreil.ly/KBPsample*](https://oreil.ly/KBPsample)下载。

如果您有技术问题或使用代码示例时遇到问题，请发送电子邮件至*bookquestions@oreilly.com*。

这本书旨在帮助您完成工作。一般情况下，如果本书提供了示例代码，您可以在您的程序和文档中使用它。除非您重复使用了大量代码，否则无需征得我们的许可。例如，编写一个使用本书多个代码片段的程序不需要许可。销售或分发 O’Reilly 书籍中的示例代码需要许可。引用本书并引用示例代码回答问题无需许可。将本书大量示例代码整合到产品文档中需要许可。

我们赞赏，但通常不需要，署名。署名通常包括标题、作者、出版商和 ISBN。例如：“*Kubernetes Best Practices* by Brendan Burns, Eddie Villalba, Dave Strebel, and Lachlan Evenson (O’Reilly). Copyright 2024 Brendan Burns, Eddie Villalba, Dave Strebel, and Lachlan Evenson, 978-1-098-14216-2.”

如果您觉得您对代码示例的使用超出了合理使用或上述许可范围，请随时联系我们，邮箱为*permissions@oreilly.com*。

# O’Reilly 在线学习

###### 注意

超过 40 年来，[*O’Reilly Media*](http://oreilly.com)为公司成功提供技术和商业培训、知识和见解。

我们独特的专家和创新者网络通过书籍、文章、会议和我们的在线学习平台分享他们的知识和专长。O’Reilly 的在线学习平台为您提供按需访问实时培训课程、深入学习路径、交互式编码环境以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。有关更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将关于本书的评论和问题发送给出版商：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   加利福尼亚州塞巴斯托波尔 95472

+   800-889-8969（美国或加拿大）

+   707-829-7019（国际或本地）

+   707-829-0104（传真）

+   *support@oreilly.com*

+   欢迎联系我们：[*https://www.oreilly.com/about/contact.html*](https://www.oreilly.com/about/contact.html)

我们有一个网页专门用来列出这本书的勘误、示例和任何额外信息。您可以访问这个页面：[*https://oreil.ly/kubernetes-best-practices2*](https://oreil.ly/kubernetes-best-practices2)。

想了解关于我们的书籍和课程的最新消息，请访问：[*https://oreilly.com*](https://oreilly.com)。

在 LinkedIn 上找到我们：[*https://linkedin.com/company/oreilly-media*](https://linkedin.com/company/oreilly-media)

关注我们的 Twitter：[*https://twitter.com/oreillymedia*](https://twitter.com/oreillymedia)

观看我们的 YouTube 频道：[*https://youtube.com/oreillymedia*](https://youtube.com/oreillymedia)

# 致谢

Brendan 想感谢他的美好家庭，Robin、Julia 和 Ethan，对他所做的一切的爱与支持；Kubernetes 社区，没有他们，这一切都不可能；以及他那些了不起的合著者，没有他们这本书也不会存在。

Dave 想感谢他美丽的妻子 Jen 和他们的三个孩子 Max、Maddie 和 Mason，对他们的支持。他还想感谢 Kubernetes 社区多年来提供的所有建议和帮助。最后，他感谢他的合著者们让这个冒险成为现实。

Lachlan 想感谢他的妻子和三个孩子，对他的爱与支持。他还想感谢多年来在 Kubernetes 社区中教导他的所有优秀个体。他还想特别感谢 Joseph Sandoval 的指导。最后，他感谢他那些出色的合著者让这本书成为可能。

Eddie 想感谢他的妻子 Sandra，在写作过程中她始终给予的支持、爱和鼓励。他还想感谢他的女儿 Giavanna，因为她给了他留下遗产的动力，这样她可以为她的爸爸感到自豪。最后，他感谢 Kubernetes 社区和他的合著者们，他们一直是他在云原生旅程中的路标。

我们都要感谢 Virginia Wilson 在撰写手稿和帮助我们整合所有想法方面的工作，以及 Jill Leonard 在第二版中的指导。最后，我们要感谢 Bridget Kromhout、Bilgin Ibryam、Roland Huß、Justin Domingus、Jess Males 和 Jonathan Johnson 对完成的关注。
