# 序言

OpenShift，作为 Kubernetes 的红帽发行版，正在迅速成为容器化应用程序的首选平台。随着企业试图扩展和运营 OpenShift，必须理解许多实践方法，以管理资源、为应用团队暴露这些资源、治理这些资源，并持续向这些环境交付变更。

基于我们在管理和支持成千上万的裸金属和虚拟机集群方面积累的专业知识，我们将重点介绍如何使用特定技术和示例来使您的组织有效地运营基本的 Kubernetes 和 OpenShift。

# 我们为什么写这本书

尽管有很多关于如何开始使用 Kubernetes 和 OpenShift 的内容，我们的目标是专注于涵盖更高级概念的书籍，例如有效管理集群资源和可应用于确保业务连续性的可用性模型。此外，我们希望深入探讨在生产环境中成功操作 Kubernetes 和 OpenShift 所必需的主题、工具和最佳实践。我们特别关注安全性、高级资源管理、持续交付、多集群管理和高可用性等主题。此外，本书探讨了支持混合云应用程序的最佳实践，这些应用程序集成了来自多个云环境以及传统 IT 环境的最佳功能和功能。在本书中，我们将我们对这些生产级主题的深入知识和经验提供给更广泛的 OpenShift 和 Kubernetes 社区。

# 本书适合对象

本书适合 DevOps 工程师、Kubernetes 和 OpenShift 平台运维工程师、站点可靠性工程师、网络运维工程师、云原生计算应用开发人员和 IT 架构师。此外，本书对那些创建和管理 Kubernetes 或 OpenShift 集群，或者使用这些平台交付应用程序和服务的人尤为感兴趣。

# 本书的组织方式

本书的结构旨在帮助运维人员和开发人员深入理解运行 Kubernetes 和 OpenShift 的高级概念。第一章概述了基本的 Kubernetes 和 OpenShift。然后讨论了 Kubernetes 的基本概念并描述了 Kubernetes 的架构。在第二章中，我们介绍了 OpenShift 和 Kubernetes 之间的关系，并描述了如何启动多种 Kubernetes 和 OpenShift 环境。第三章深入探讨了高级资源管理主题，包括专业调度、资源预留、专用节点类型以及容量规划和管理。第四章涵盖了在单个集群内支持高可用性所需的关键基础知识。在第五章中，我们介绍了跨多个企业集群进行生产级持续交付和代码推广的概述。第六章和 7 章聚焦于多个生产集群的使用。在这些章节中，我们详细描述了几个混合云使用案例，并深入讨论了多集群配置、升级和策略支持等高级主题。这些概念在第八章得到了进一步强化，我们提供了一个多集群应用交付的工作示例。最后，在第九章中，我们讨论了 Kubernetes 和 OpenShift 的未来，并列出了关于在生产 Kubernetes 和 OpenShift 环境中运行应用程序的各种有用主题的额外参考资料。

# 本书中使用的约定

本书使用以下印刷约定：

*斜体*

指示新术语，URL，电子邮件地址，文件名和文件扩展名。

`常量宽度`

用于程序清单，以及段落内引用程序元素，如变量或函数名，数据库，数据类型，环境变量，语句和关键字。

**`常量宽度粗体`**

显示用户应直接输入的命令或其他文本。

*`常量宽度斜体`*

显示应由用户提供值或由上下文确定的值的文本。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般注释。

###### 警告

此元素表示警告或注意事项。

# 使用代码示例

补充材料（代码示例，练习等）可在[*https://github.com/hybrid-cloud-apps-openshift-k8s-book*](https://github.com/hybrid-cloud-apps-openshift-k8s-book)下载。

如果您有技术问题或使用代码示例时遇到问题，请发送电子邮件至*bookquestions@oreilly.com*。

本书旨在帮助您完成工作任务。一般来说，如果本书提供了示例代码，您可以在自己的程序和文档中使用它。除非您复制了代码的大部分内容，否则无需联系我们以获得许可。例如，编写一个使用本书中几个代码片段的程序不需要许可。销售或分发来自 O’Reilly 书籍的示例代码则需要许可。引用本书并引用示例代码来回答问题也不需要许可。将本书中大量示例代码合并到您产品的文档中则需要许可。

我们感谢，但通常不要求署名。署名通常包括标题，作者，出版商和 ISBN。例如：“*使用 OpenShift 和 Kubernetes 的混合云应用*，由 Michael Elder，Jake Kitchener 和 Dr. Brad Topol（O’Reilly）编写。版权所有 2021 Michael Elder，Jake Kitchener，Brad Topol，978-1-492-08381-8。”

如果您认为您使用的示例代码超出了合理使用或上述许可的范围，请随时通过*permissions@oreilly.com*与我们联系。

# O’Reilly 在线学习

###### 注意

40 多年来，[*O’Reilly Media*](http://oreilly.com)已为公司提供技术和业务培训、知识和见解，帮助其取得成功。

我们独特的专家和创新者网络通过书籍、文章和我们的在线学习平台分享他们的知识和专业知识。O’Reilly 的在线学习平台为您提供按需访问的实时培训课程、深度学习路径、交互式编码环境以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。有关更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。  

# 如何联系我们

请将对本书的评论和问题发送至出版商：

+   O’Reilly Media，Inc.

+   1005 Gravenstein Highway North

+   Sebastopol，CA 95472

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为本书设有一个网页，列出勘误表，示例和任何额外信息。您可以访问[*https://oreil.ly/hybrid-cloud*](https://oreil.ly/hybrid-cloud)查看此页面。

发送电子邮件至*bookquestions@oreilly.com*以发表评论或询问有关本书的技术问题。

有关我们的书籍和课程的新闻和信息，请访问[*http://oreilly.com*](http://oreilly.com)。

在 Facebook 上找到我们：[*http://facebook.com/oreilly*](http://facebook.com/oreilly)

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*http://youtube.com/oreillymedia*](http://www.youtube.com/oreillymedia)

# 致谢

我们要感谢整个 Kubernetes 和 OpenShift 社区，他们对这些开源项目的热情、奉献精神和巨大承诺。如果没有多年来为这些项目贡献代码的开发者、代码审阅者、文档作者和运维人员，Kubernetes 和 OpenShift 将不会拥有今天它们享有的丰富功能集、强大的采用率和庞大的生态系统。

我们要向为本文提供宝贵反馈的技术审阅者表示诚挚的感谢：Dan “Pop” Papandrea、Daniel Berg、Joshua Packer、Scott Berens 和 Burr Sutter。

我们要感谢我们的 Kubernetes 同事 Clayton Coleman、Derek Carr、David Eads、Paul Morie、Zach Corleissen、Jim Angel、Tim Bannister、Celeste Horgan、Irvi Aini、Karen Bradshaw、Kaitlyn Barnard、Taylor Dolezal、Jorge Castro、Jason DeTiberus、Stephen Augustus、Guang Ya Liu、Sahdev Zala、Wei Huang、Michael Brown、Jonathan Berkhahn、Martin Hickey、Chris Luciano、Srinivas Brahmaroutu、Morgan Bauer、Qiming Teng、Richard Theis、Tyler Lisowski、Bill Lynch、Jennifer Rondeau、Seth McCombs、Steve Perry 和 Joe Heck，感谢多年来的精彩合作。

我们还要感谢贡献给 Open Cluster Management 项目及其相关项目的贡献者，包括 OpenShift Hive 项目、Open Policy Agent 和 ArgoCD。

我们要感谢许多开源贡献者，他们使得 Kubernetes 以外的项目和示例得以实现，特别是例子 PAC-MAN 应用的作者和发起者，包括 Ivan Font、Mario Vázquez、Jason DeTiberus、Davis Phillips 和 Pep Turró Mauri。我们还要感谢许多示例 Open Cluster Management 策略的作者和贡献者，包括 Yu Cao、Christian Stark 和 Jaya Ramanathan。

我们还要感谢我们的编辑 Angela Rufino，在一个非常动态的年份和全球性流行病期间耐心地参与书写过程。此外，我们要感谢我们的副本编辑 Shannon Turlington，她对我们的工作进行了细致的审查，并提出了大量建议改进。

特别感谢 Willie Tejada、Todd Moore、Bob Lord、Dave Lindquist、Kevin Myers、Jeff Brent、Jason McGee、Chris Ferris、Vince Brunssen、Alex Tarpinian、Jeff Borek、Nimesh Bhatia、Briana Frank 和 Jake Morlock，在此过程中给予我们的所有支持和鼓励。

—Michael、Jake 和 Brad
