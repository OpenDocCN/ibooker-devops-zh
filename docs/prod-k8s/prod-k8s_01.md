# 前言

Kubernetes 是一项非常强大的技术，并且在流行度上迅速崛起。它已经成为我们管理软件部署方式上真正进步的基础。在 Kubernetes 出现之时，虽然 API 驱动的软件和分布式系统已经得到了确立，但尚未被广泛采用。它出色地实现了这些原则，这些原则是其成功的基石，但它也带来了一些至关重要的其他东西。在不久的过去，只有拥有最杰出工程团队的大型科技公司才能实现自动收敛于声明的期望状态的软件。如今，由于 Kubernetes 项目，高可用、自我修复、自动扩展的软件部署对每个组织来说都是可实现的。在我们面前有一个未来，软件系统能接受我们的广泛高级指令，并执行它们以提供预期的结果，通过发现条件、应对变化的障碍和自动修复问题，无需我们干预。而且，这些系统比我们手工操作更快速、更可靠。Kubernetes 已经让我们离这个未来更近了一步。然而，这种强大和能力是以一些额外复杂性为代价的。我们决定撰写这本书，是因为我们希望分享我们的经验，帮助他人应对这种复杂性。

如果你想要使用 Kubernetes 构建一个符合生产标准的应用平台，你应该阅读这本书。如果你正在寻找一本帮助你入门 Kubernetes 或者关于 Kubernetes 工作原理的书籍，那么这本书不适合。关于这些主题有大量信息可以在其他书籍、官方文档以及无数的博客文章和源代码中找到。我们建议你在阅读本书的同时进行自己的研究和测试，以验证我们讨论的解决方案，因此我们很少深入进行*逐步*教程样式的例子。我们尽量涵盖必要的理论，并把大部分*实现*留给读者自己来练习。

在这本书中，你将会找到关于选项、工具、模式和实践的指导。重要的是，理解作者如何看待构建应用平台的实践。我们是工程师和架构师，被派往许多财富 500 强公司，帮助他们从构想到生产的平台愿景。自从 Kubernetes 在 2015 年达到`1.0`版本以来，我们就把它作为实现这一目标的基础。我们尽可能地专注于模式和哲学，而不是工具，因为新的工具出现得比我们写书更快！然而，我们不可避免地必须用当下最合适的工具来演示这些模式。

我们已成功引导团队通过云原生之旅，彻底改变他们构建和交付软件的方式。话虽如此，我们也曾经历失败。失败的一个常见原因是组织对 Kubernetes 能解决什么存在误解。这就是为什么我们早期深入探讨这个概念的原因。在此期间，我们发现几个领域对我们的客户尤其有趣。帮助客户在他们通往生产环境的道路上更进一步，甚至帮助他们定义这条道路的对话已成常态。这些对话变得如此普遍，以至于我们决定也许是时候写一本书了！

尽管我们一次又一次地与组织一同走向生产，但其中有一个关键的一致性特征。这就是，无论我们多么希望如此，道路*永远*不会看起来相同。基于此，我们想要设定一个期望：如果你打算通过这本书找到通往生产环境的“5 步计划”或“每个 Kubernetes 用户应该知道的 10 件事”，你将会感到沮丧。我们在这里讨论许多决策点和我们见过的陷阱，并在适当时用具体的例子和轶事加以支持。最佳实践确实存在，但必须始终从实用主义的角度来看待。没有一种适合所有情况的方法，“这取决于情况”是你将不可避免地面对的许多问题的完全有效的答案。

话虽如此，我们强烈建议你*挑战*本书！在与客户合作时，我们总是鼓励他们挑战和完善我们的指导。知识是流动的，我们始终根据新特性、信息和约束更新我们的方法。你应该继续这种趋势；随着云原生空间的不断发展，你肯定会决定采取与我们推荐的不同路径。我们在这里告诉你我们曾经走过的路，这样你就可以权衡我们的观点和你自己的观点。

# 本书中使用的约定

本书中使用以下排版约定：

*Italic*

表示新术语、网址、电子邮件地址、文件名和文件扩展名。

`Constant width`

用于程序清单，以及在段落内引用程序元素，如变量或函数名、数据库、数据类型、环境变量、语句和关键字。

**`Constant width bold`**

显示用户应按字面意义输入的命令或其他文本。

*`Constant width italic`*

显示应由用户提供的值或由上下文确定的值替换的文本。

Kubernetes 的类型名称采用大写，如 Pod、Service 和 StatefulSet。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般提示。

###### 警告

此元素表示警告或注意。

# 使用代码示例

补充材料（代码示例、练习等）可在 [*https://github.com/production-kubernetes*](https://github.com/production-kubernetes) 下载和讨论。

如果您有技术问题或使用代码示例时遇到问题，请发送电子邮件至 *bookquestions@oreilly.com*。

这本书旨在帮助您完成工作。一般情况下，如果书中提供了示例代码，您可以在自己的程序和文档中使用它们，无需事先联系我们获得许可，除非您要复制代码的大部分。例如，编写一个使用本书中多个代码片段的程序不需要许可。销售或分发从 O’Reilly 图书中提取的示例代码需要获得许可。引用本书并引用示例代码来回答问题不需要许可。将本书大量示例代码整合到产品文档中需要获得许可。

我们感谢您的支持，但通常不要求署名。署名通常包括标题、作者、出版商和 ISBN。例如：“*Production Kubernetes* by Josh Rosso, Rich Lander, Alexander Brand, and John Harris (O’Reilly). Copyright 2021 Josh Rosso, Rich Lander, Alexander Brand, and John Harris, 978-1-492-09231-5.”

如果您认为使用代码示例超出了公平使用范围或上述许可，请随时通过 *permissions@oreilly.com* 联系我们。

# O’Reilly 在线学习

###### 注意

超过 40 年来，[*O’Reilly Media*](http://oreilly.com) 一直为公司提供技术和商业培训、知识和洞察力，帮助它们取得成功。

我们独特的专家和创新者网络通过书籍、文章和我们的在线学习平台分享他们的知识和专业知识。O’Reilly 的在线学习平台让您随时访问现场培训课程、深入学习路径、交互式编码环境以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。获取更多信息，请访问：[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

有关此书的评论和问题，请联系出版社：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为这本书创建了一个网页，上面列出了勘误、示例和任何额外信息。您可以访问这个页面：[*https://oreil.ly/production-kubernetes*](https://oreil.ly/production-kubernetes)。

通过电子邮件 *bookquestions@oreilly.com* 发表评论或提出关于本书的技术问题。

有关我们的书籍和课程的新闻和信息，请访问 [*http://oreilly.com*](http://oreilly.com)。

在 Facebook 上找到我们：[*http://facebook.com/oreilly*](http://facebook.com/oreilly)

在 Twitter 关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在 YouTube 观看我们：[*http://youtube.com/oreillymedia*](http://youtube.com/oreillymedia)

# 致谢

作者们感谢 Katie Gamanji、Michael Goodness、Jim Weber、Jed Salazar、Tony Scully、Monica Rodriguez、Kris Dockery、Ralph Bankston、Steve Sloka、Aaron Miller、Tunde Olu-Isa、Alex Withrow、Scott Lowe、Ryan Chapple 和 Kenan Dervisevic 对手稿的评论和反馈。感谢 Paul Lundin 在书籍开发过程中的鼓励，并且在 Heptio 建立了出色的 Field Engineering 团队。团队中的每个人都通过协作和开发了我们接下来 450 页所涵盖的许多想法和经验做出了贡献。最后，感谢 VMware 的 Joe Beda、Scott Buchanan、Danielle Burrow 和 Tim Coventry-Cox 在我们启动和开发此项目时的支持。最后，感谢 O’Reilly 的 John Devins、Jeff Bleiel 和 Christopher Faucher 持续的支持和反馈。

作者们还要特别感谢以下人士：

Josh：我要感谢 Jessica Appelbaum，在我撰写本书期间给予我的大力支持，特别是提供的蓝莓煎饼。我还要感谢我的妈妈 Angela 和爸爸 Joe，在我成长过程中为我提供坚实的支持。

Rich：我要感谢我的妻子 Taylor 和孩子们 Raina、Jasmine、Max 和 John，在我撰写本书期间给予我的支持和理解。我还要感谢我的妈妈 Jenny 和爸爸 Norm，他们是我很好的榜样。

Alexander：我爱你，感谢我的不可思议的妻子 Anais，在我撰写本书期间给予我的极大支持。我还感谢我的家人、朋友和同事，是他们帮助我成为今天的我。

John：我要感谢我美丽的妻子 Christina，在我撰写本书期间给予我的爱和耐心。也感谢我的亲密朋友和家人多年来的支持和鼓励。
