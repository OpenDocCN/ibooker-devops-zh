# 前言

欢迎阅读 *《黑客 Kubernetes》*，一本专为希望安全稳定运行其工作负载的 Kubernetes 实践者而写的书籍。截至撰写时，Kubernetes 已经存在了大约六年。有超过一百种认证的 [Kubernetes 产品](https://oreil.ly/bo2xA) 可供选择，如分发版和托管服务。随着越来越多的组织决定将其工作负载迁移到 Kubernetes 上，我们希望在这本书中分享我们的经验，帮助您的工作负载更安全、更容易部署和操作。感谢您加入我们的旅程，希望您在阅读本书和应用所学知识时同样愉快。

在本前言中，我们将描述我们的目标受众，谈谈我们为什么写这本书，并解释我们认为您应该如何使用它，提供一个快速的内容指南。我们还会介绍一些行政细节，如 Kubernetes 的版本和使用的约定。

# 关于你

为了最大限度地利用本书，我们假设您要么担任 DevOps 角色，要么是 Kubernetes 平台专家，要么是云原生架构师，要么是站点可靠性工程师（SRE），或者从事与首席信息安全官（CISO）相关的工作。此外，我们假设您对实际操作感兴趣——尽管我们在原则上讨论威胁和防御，但我们尽力同时演示它们，并指导您使用能帮助您的工具。

此时，我们还要确保您了解到，您正在阅读的书籍面向高级主题。我们假设您已经对 Kubernetes 以及特别是 Kubernetes 安全主题有所了解，至少是表面了解。换句话说，我们不会详细讨论事物的工作原理，但会在每章节基础上进行总结或回顾重要的概念或机制。

###### 警告

我们撰写本书时考虑了蓝队和红队。不言而喻，我们在这里分享的内容仅供保护您自己的 Kubernetes 集群和工作负载使用。

特别地，我们假设您了解容器的用途以及它们在 Kubernetes 中的运行方式。如果您对这些主题还不熟悉，我们建议您先做一些初步阅读。以下是我们建议参考的书籍：

+   [*Kubernetes: Up and Running*](https://oreil.ly/k9ydo)，作者 Brendan Burns、Kelsey Hightower 和 Joe Beda（O’Reilly）

+   [*管理 Kubernetes*](https://oreil.ly/cli0J)，作者 Brendan Burns 和 Craig Tracey（O’Reilly）

+   [*Kubernetes 安全*](https://oreil.ly/S8jQf)，作者 Liz Rice 和 Michael Hausenblas（O’Reilly）

+   [*容器安全*](https://oreil.ly/Yh8EM)，作者 Liz Rice（O’Reilly）

+   *云原生安全*，作者 Chris Binnie 和 Rory McCune（Wiley）

现在我们已经明确了本书的目标以及我们认为谁将从中受益，让我们转移到另一个话题：作者们。

# 关于我们

基于我们共同超过 10 年的实际经验，设计、运行、攻击和防御基于 Kubernetes 的工作负载和集群，我们作为作者，希望为你作为云原生安全从业者提供成功所需的知识。

安全性通常是通过过去的错误之光来阐明的，我们两个都在学习（和犯错！）Kubernetes 安全性已经有一段时间了。我们希望确认我们对主题的理解是否正确，因此我们写了一本书，通过共同的视角验证我们的怀疑。

我们曾在不同的公司和角色中任职，提供培训课程，发布从工具到博客文章的材料，并在各种公开演讲中分享我们在该主题上的经验教训。我们在这里的动机和使用的例子很大程度上根植于我们日常工作中的经验和/或我们在客户公司观察到的事物。

# 如何使用本书

本书是关于 Kubernetes 安全的基于威胁的指南，使用纯净的 Kubernetes 安装及其（内建的）默认设置作为起点。我们将从一个分布式系统运行任意工作负载的抽象威胁模型开始讨论，并详细评估安全 Kubernetes 系统的每个组件。

在每一章中，我们都会检查组件的架构和潜在的默认设置，并审查高调攻击和历史上的公共漏洞（CVE）。我们还演示攻击并分享最佳实践配置，以示范从多个攻击角度加固集群。

为了帮助您浏览本书，以下是各章节的快速介绍：

+   在第一章，“介绍”中，我们设定了场景，介绍了我们的主要对手以及威胁建模是什么。

+   第二章，“Pod 级资源” 然后专注于 Pod，从配置到攻击到防御。

+   接下来，在第三章，“容器运行时隔离”中，我们转变方向，深入探讨隔离和沙盒技术。

+   接着，在第四章，“应用程序和供应链”中，我们讨论了供应链攻击及其检测和缓解方法。

+   在第五章，“网络”中，我们再次审查网络默认设置以及如何保护集群和工作负载流量。

+   然后，在第六章，“存储”中，我们将焦点转向持久性方面，查看文件系统、卷以及静态信息。

+   第七章，“硬件多租户” 讲述了在集群中运行多租户工作负载的主题，以及可能出现的问题。

+   接下来是 第八章，“策略”，我们在此章节回顾了各种使用的策略类型，讨论了访问控制——特别是基于角色的访问控制（RBAC）——以及通用策略解决方案如开放策略代理（OPA）。

+   在 第九章，“入侵检测” 中，我们讨论了即使已经实施了控制措施，如果有人成功闯入，你可以采取的应对措施。

+   最后一章，第十章，“组织”，有些特殊，不侧重于工具，而是侧重于云端和本地安装环境下的人员方面的讨论。

在 附录 A，“Pod 级攻击” 中，我们将通过一个实际的探索，手把手地介绍了有关 Pod 级攻击的讨论，正如在 第二章 中讨论的那样。最后，在 附录 B，“资源” 中，我们根据每章内容提供了进一步阅读材料，并汇集了在本书上下文中相关的注释 CVE 集合。

您无需按顺序阅读各章节；我们尽力使章节尽可能自包含，并在适当时引用相关内容。

###### 注意

请注意，在撰写本书时，Kubernetes 1.21 是最新稳定版本。这里展示的大多数示例适用于早期版本，并且我们完全意识到，当您阅读本书时，当前版本可能会显著更高。但其核心概念保持不变。

通过这本简短的指南和快速的定位，让我们来看看本书中使用的约定。

# 本书中使用的约定

本书使用以下排版约定：

*斜体*

指示新术语、网址、电子邮件地址、文件名和文件扩展名。

`固定宽度`

用于程序清单。也在段落内用于引用程序元素，如变量或函数名称、数据库、数据类型、环境变量、语句和关键字。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般提示。

###### 警告

此元素表示警告或注意事项。

# 使用代码示例

附加材料位于 [*http://hacking-kubernetes.info*](http://hacking-kubernetes.info)。

如果您有技术问题或在使用代码示例时遇到问题，请发送电子邮件至 *bookquestions@oreilly.com*。

本书旨在帮助您完成工作。通常情况下，如果本书提供示例代码，您可以在您的程序和文档中使用它。除非您要复制代码的大部分内容，否则无需征得我们的许可。例如，编写一个使用本书多个代码片段的程序不需要许可。销售或分发 O’Reilly 书籍的示例需要许可。引用本书回答问题并引用示例代码不需要许可。将本书大量示例代码整合到您产品的文档中需要许可。

我们感谢您的赞赏，但通常不要求署名。署名通常包括标题、作者、出版商和 ISBN。例如：“*Hacking Kubernetes* by Andrew Martin and Michael Hausenblas (O’Reilly)。版权所有 2022 Andrew Martin 和 Michael Hausenblas，978-1-492-08173-9。”

如果您认为您使用的示例代码超出了合理使用范围或以上给出的许可，欢迎随时与我们联系：*permissions@oreilly.com*。

# O’Reilly 在线学习

###### 注意

40 多年来，[*O’Reilly Media*](http://oreilly.com) 提供技术和商业培训、知识和洞察，帮助企业成功。

我们独特的专家和创新者网络通过书籍、文章和我们的在线学习平台分享知识和专业知识。O’Reilly 的在线学习平台提供按需访问的实时培训课程、深度学习路径、交互式编码环境，以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。欲了解更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将对本书的评论和问题寄给出版商：

+   O’Reilly Media, Inc.

+   Gravenstein Highway North 1005 号

+   加利福尼亚州塞巴斯托波尔市 95472

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104 (传真)

我们为本书设立了网页，列出勘误、示例和任何额外信息。您可以访问[*https://oreil.ly/HackingKubernetes*](https://oreil.ly/HackingKubernetes)。

电子邮件*bookquestions@oreilly.com*以评论或提出关于本书的技术问题。

有关我们的图书和课程的新闻和信息，请访问[*http://oreilly.com*](http://oreilly.com)。

在 Facebook 上找到我们：[*http://facebook.com/oreilly*](http://facebook.com/oreilly)

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

观看我们的 YouTube 频道：[*http://youtube.com/oreillymedia*](http://youtube.com/oreillymedia)

# 致谢

感谢我们的审阅者 Roland Huss、Liz Rice、Katie Gamanji、Ihor Dvoretskyi、Mark Manning 和 Michael Gasch。您的评论确实起到了作用，我们感谢您的指导和建议。

Andy 要感谢他的家人和朋友们给予他不断的爱和鼓励，感谢 ControlPlane 团队那些鼓舞人心又锋利的成员给予的深刻洞察和指导，以及那些始终热心和卓越的云原生安全社区。特别感谢 Rowan Baker、Kevin Ward、Lewis Denham-Parry、Nick Simpson、Jack Kelly 和 James Cleverley-Prance。

Michael 要向他那令人感激而又支持无微不至的家人表达最深的谢意：我们的孩子 Saphira、Ranya 和 Iannis；我那位聪明又有趣的妻子 Anneliese，还有我们最棒的狗狗 Snoopy。

我们不得不提及[Hacking Kubernetes Twitter 列表](https://oreil.ly/xr1is)，它是我们灵感和指导的来源，包括按字母顺序排列的知名人物，如[@antitree](https://twitter.com/antitree)，[@bradgeesaman](https://twitter.com/bradgeesaman)，[@brau_ner](https://twitter.com/brau_ner)，[@christianposta](https://twitter.com/christianposta)，[@dinodaizovi](https://twitter.com/dinodaizovi)，[@erchiang](https://twitter.com/erchiang)，[@garethr](https://twitter.com/garethr)，[@IanColdwater](https://twitter.com/IanColdwater)，[@IanMLewis](https://twitter.com/IanMLewis)，[@jessfraz](https://twitter.com/jessfraz)，[@jonpulsifer](https://twitter.com/jonpulsifer)，[@jpetazzo](https://twitter.com/jpetazzo)，[@justincormack](https://twitter.com/justincormack)，[@kelseyhightower](https://twitter.com/kelseyhightower)，[@krisnova](https://twitter.com/krisnova)，[@kubernetesonarm](https://twitter.com/kubernetesonarm)，[@liggitt](https://twitter.com/liggitt)，[@lizrice](https://twitter.com/lizrice)，[@lordcyphar](https://twitter.com/lordcyphar)，[@lorenc_dan](https://twitter.com/lorenc_dan)，[@lumjjb](https://twitter.com/lumjjb)，[@mauilion](https://twitter.com/mauilion)，[@MayaKaczorowski](https://twitter.com/MayaKaczorowski)，[@mikedanese](https://twitter.com/mikedanese)，[@monadic](https://twitter.com/monadic)，[@raesene](https://twitter.com/raesene)，[@swagitda_](https://twitter.com/swagitda_)，

最后但绝非最不重要的，两位作者特别感谢 O’Reilly 团队，尤其是 Angela Rufino，因为她在撰写本书的过程中给予了我们无微不至的指导。
