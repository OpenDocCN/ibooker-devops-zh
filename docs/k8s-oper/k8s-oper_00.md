# 序言

Kubernetes 是一个流行的容器编排器。它将许多计算机汇集在一起，形成一个大型计算资源，并通过 Kubernetes 应用程序编程接口（API）建立了一种访问该资源的方式。Kubernetes 是由 Google 发起的开源软件，在 Cloud Native Computing Foundation (CNCF) 的赞助下，通过一大群合作者在过去五年中开发而成。

运算符扩展 Kubernetes，自动化特定应用程序的整个生命周期管理。运算符作为在 Kubernetes 上分发应用程序的打包机制，它们监视、维护、恢复和升级它们所部署的软件。

# 本书的读者对象

如果您已在 Kubernetes 集群上部署过应用程序，您将熟悉一些塑造运算符模式的挑战和愿望。如果您已经在自己的编排集群之外维护基础服务（如数据库和文件系统），并且希望将它们引入这个领域，那么这本关于 Kubernetes 运算符的指南就是为您准备的。

# 您将学到什么

本书解释了什么是运算符，以及运算符如何扩展 Kubernetes API。它展示了如何部署和使用现有的运算符，以及如何使用 [Red Hat Operator Framework](https://github.com/operator-framework) 为您的应用程序创建和分发运算符。我们介绍了设计、构建和分发运算符的良好实践，并解释了通过可靠性工程（SRE）原则赋予运算符活力的思维方式。

在第一章描述了运算符及其概念之后，我们将建议如何访问 Kubernetes 集群，在这里您可以进行本书其余部分的练习。有了运行中的集群，您将部署一个运算符，并观察其在应用程序失败、扩展或升级到新版本时的行为。

后来，我们将探索运算符 SDK，并向您展示如何使用它将一个示例应用程序建立为 Kubernetes 的一流公民。在此实际基础上，我们将讨论运算符源自的 SRE 思想以及它们所共享的目标：减少操作工作和成本，提高服务可靠性，并通过解放团队免受重复性维护工作来推动创新。

## 运算符框架和 SDK

运算符模式是在 [CoreOS](https://coreos.com) 提出的，作为在 Kubernetes 集群上自动化越来越复杂应用程序的一种方式，包括管理 Kubernetes 本身以及其核心的 [etcd](https://github.com/coreos/etcd) 键值存储。运算符的工作通过被 Red Hat 收购继续进行，导致了开源运算符框架和 SDK 在 2018 年的发布。本书的示例使用了 Red Hat Operator SDK 和加入 Operator Framework 的分发机制。

## 其他运算符工具

已经形成了围绕 Operators 的社区，仅在 Red Hat 的分发渠道中就有超过一百个 Operators，涵盖了各种供应商和项目的应用。还存在其他几个 Operator 构建工具。我们不会详细讨论它们，但在阅读本书后，您将能够将它们与 Operator SDK 和 Framework 进行比较。用于构建 Operators 的其他开源工具包括[Kopf](https://oreil.ly/JCL-S)（用于 Python）、[Kubebuilder](https://oreil.ly/8zdbj)（来自 Kubernetes 项目）和[Java Operator SDK](https://oreil.ly/yXhVg)。

# 本书使用的约定

本书使用以下排版约定：

*Italic*

表示新术语、URL、电子邮件地址、文件名和文件扩展名。

`Constant width`

用于程序列表以及段落内用来指代程序元素（如变量或函数名称、数据库、数据类型、环境变量、语句和关键字）的代码。

**`Constant width bold`**

展示用户应该按字面意思输入的命令或其他文本。

*`Constant width italic`*

展示应该由用户提供的值或由上下文确定的值替换的文本。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般注释。

###### 警告

此元素表示警告或注意事项。

# 使用代码示例

附加材料（代码示例、练习等）可在[*https://github.com/kubernetes-operators-book/*](https://github.com/kubernetes-operators-book/)下载。

如果您有技术问题或使用代码示例时遇到问题，请发送电子邮件至*bookquestions@oreilly.com*。

本书旨在帮助您完成工作。一般来说，如果本书提供了示例代码，您可以在您的程序和文档中使用它。除非您正在复制代码的大部分内容，否则您无需联系我们以获取许可。例如，编写一个使用本书多个代码片段的程序并不需要许可。出售或分发来自 O'Reilly 书籍的示例代码需要许可。通过引用本书并引用示例代码来回答问题不需要许可。将本书大量示例代码纳入产品文档中需要许可。

我们感谢，但通常不需要归属。归属通常包括标题、作者、出版商和 ISBN。例如：“*Kubernetes Operators* by Jason Dobies and Joshua Wood (O’Reilly). Copyright 2020 Red Hat, Inc., 978-1-492-04804-6.”

如果您觉得您使用的代码示例超出了合理使用范围或上述许可，请随时通过*permissions@oreilly.com*与我们联系。

# O’Reilly Online Learning

###### 注意

[*O’Reilly Media*](http://oreilly.com) 在过去 40 多年里，提供技术和商业培训、知识和见解，帮助公司取得成功。

我们独特的专家和创新者网络通过图书、文章、会议和我们的在线学习平台分享他们的知识和专长。O’Reilly 的在线学习平台为您提供按需访问实时培训课程、深入学习路径、交互式编码环境，以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。有关更多信息，请访问 [*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将有关本书的评论和问题发送至出版商：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为这本书建立了一个网页，列出勘误、示例和任何额外信息。您可以访问此页面 [*https://oreil.ly/Kubernetes_Operators*](https://oreil.ly/Kubernetes_Operators)。

发送电子邮件至 *bookquestions@oreilly.com* 进行评论或提出技术问题。

有关我们的图书、课程和会议的更多信息，请访问 [*http://www.oreilly.com*](http://www.oreilly.com)。

在 Facebook 上找到我们：[*http://facebook.com/oreilly*](http://facebook.com/oreilly)

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在 YouTube 上关注我们：[*http://www.youtube.com/oreillymedia*](http://www.youtube.com/oreillymedia)

# 致谢

我们要感谢 Red Hat 公司和 OpenShift 倡导团队的支持，特别感谢 Ryan Jarvinen 的坚定和全方位的帮助。我们还要感谢许多人对这项工作进行了审查、检查、建议，以及其他付出时间使这项工作更加连贯和完整的人，其中包括 Anish Asthana、Evan Cordell、Michael Gasch、Michael Hausenblas、Shawn Hurley 和 Jess Males。
