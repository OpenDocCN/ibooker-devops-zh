# 序言

作为微服务的运行时和编排环境，Kubernetes 在初创企业和大型企业中广泛使用。随着组织应用数量的增加，管理 Kubernetes 集群成为一项全职工作。这是 Kubernetes 管理员的角色。负责这项工作的人确保每个集群处于运行状态，通过加入节点扩展集群，升级节点的 Kubernetes 版本以包含补丁和新功能，并负责关键集群数据的备份策略。为了帮助求职者和雇主有一种标准手段来展示和评估在 Kubernetes 环境中开发的熟练程度，云原生计算基金会（CNCF）开发了 [认证的 Kubernetes 管理员（CKA）](https://oreil.ly/Ds8nq) 计划。要获得这一认证，你需要通过一项考试。

在 CNCF 网页上可以找到另外两种 Kubernetes 认证。[认证的 Kubernetes 应用开发者（CKAD）](https://oreil.ly/RxHwQ) 专注于开发者中心化的 Kubernetes 应用。[认证的 Kubernetes 安全专家（CKS）](https://oreil.ly/53csH) 是为验证在安全相关主题上的能力而创建的，需要在注册之前成功通过 CKA 考试。CKAD 和 CKS 的通过对于参加 CKA 考试并不是强制的。

在这本学习指南中，我将探讨覆盖 CKA 考试内容的主题，以充分准备您通过认证考试。我们将研究何时以及如何应用 Kubernetes 核心概念来管理应用程序。我们还将审视 Kubernetes 工程师的主要工具 `kubectl` 命令行工具。我还将提供一些提示，帮助您更好地准备考试，并分享我准备考试各个方面的个人经验。

CKA 与其他认证的典型多选题格式不同。它完全基于表现，并要求你在极大的时间压力下展示对手头任务的深刻理解。你准备好一次通过考试了吗？

# 本书适合对象

本书的主要目标群体是想为 CKA 考试做准备的管理员。内容“考试详情和资源”涵盖了考试课程的所有方面，但预期你具备 Kubernetes 架构及其概念的基本知识。如果你完全是 Kubernetes 新手，我建议先阅读 [《Kubernetes 上手指南》](https://oreil.ly/KSPkv)（Brendan Burns、Joe Beda、Kelsey Hightower 和 Lachlan Evenson（O’Reilly））或者 *Kubernetes 实战*（Marko Lukša（Manning Publications））。

# 你将学到什么

本书内容总结了与 CKA 考试相关的最重要方面。像 AKS 或 GKE 这样的云提供商特定的 Kubernetes 实现不需要考虑在内。鉴于 Kubernetes 中提供的大量配置选项，几乎不可能覆盖所有用例和场景而不重复官方文档。鼓励考生参考[Kubernetes 文档](https://oreil.ly/MoLjc)，作为广泛学习的权威手册。

本书的大纲严格遵循 CKA 课程。虽然一般来说，可能有更自然、更教学的学习 Kubernetes 的结构，但课程大纲将帮助考生通过专注于特定主题来准备考试。因此，根据您的现有知识水平，您将发现自己在书的其他章节之间进行交叉参考。

请注意，本书仅涵盖与 CKA 考试相关的概念。某些您可能期望在认证课程中涵盖的基元（例如 API 基元 Ingress）未涉及。如果您希望深入了解，请参考 Kubernetes 文档或其他书籍。

在通过 Kubernetes 进行实际体验对通过考试至关重要。每章都包含名为“示例练习”的部分，提供实践问题。这些问题的解决方案可在附录中找到。

# 本书使用的约定

本书使用以下排印约定：

*斜体*

表示新术语、URL 和电子邮件地址。

`常量宽度`

用于文件名、文件扩展名和程序清单，以及段落内引用程序元素如变量或函数名、数据库、数据类型、环境变量、语句和关键字。

**`常量宽度粗体`**

显示用户应按字面输入的命令或其他文本。

*`常量宽度斜体`*

显示应由用户提供值或由上下文确定值的文本。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般说明。

###### 警告

此元素表示警告或注意事项。

# 使用代码示例

本书中所有示例和练习的源代码都可以在[GitHub](https://github.com/bmuschko/cka-study-guide)上找到。该存储库使用 Apache License 2.0 进行分发，代码可在商业和开源项目中免费使用。如果您在源代码中遇到问题或有疑问，请在[GitHub 问题跟踪器](https://oreil.ly/09Z7p)中提出问题。我很乐意与您交流并解决可能出现的任何问题。

本书旨在帮助您完成工作。一般情况下，如果本书提供示例代码，您可以在您的程序和文档中使用它。除非您复制了大量代码，否则无需联系我们获取许可。例如，编写一个使用本书多个代码块的程序不需要许可。出售或分发 O’Reilly 图书中的示例代码需要许可。引用本书并引用示例代码来回答问题不需要许可。将本书的大量示例代码整合到您产品的文档中需要许可。我们感谢但通常不要求署名。署名通常包括标题、作者、出版商和 ISBN。例如：“*Certified Kubernetes Administrator (CKA) Study Guide* by Benjamin Muschko (O’Reilly)。版权 2022 年 Some Benjamin Muschko，978-1-098-10722-2。”

如果您觉得您使用的代码示例超出了合理使用范围或上述许可，请随时通过*permissions@oreilly.com*与我们联系。

# O’Reilly 互动 Katacoda 实验室

交互式 Katacoda 场景模仿真实的生产环境，让您在浏览器中学习时编写和运行代码。作者开发了一系列 Katacoda 场景，让您通过这本书中概述的工具和实践进行实践。有关我们的交互式内容的更多信息，请访问[*http://oreilly.com*](http://oreilly.com)，查看本标题的电子书格式，还可以了解我们的学习平台的所有内容。

# O’Reilly 在线学习

###### 注意

40 多年来，[*O’Reilly Media*](http://oreilly.com)提供技术和商业培训、知识和见解，帮助公司取得成功。

我们独特的专家和创新者网络通过书籍、文章和我们的在线学习平台分享他们的知识和专业知识。O’Reilly 的在线学习平台为您提供按需访问现场培训课程、深入学习路径、交互式编码环境以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。有关更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请向出版商发送有关本书的评论和问题：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   CA 95472 Sebastopol

+   800-998-9938（在美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为本书创建了一个网页，列出了勘误、示例和任何额外信息。您可以访问[*https://oreil.ly/cka-study-guide*](https://oreil.ly/cka-study-guide)查看。

通过*bookquestions@oreilly.com*来评论或询问有关本书的技术问题。

有关我们的图书和课程的新闻和信息，请访问[*http://oreilly.com*](http://oreilly.com)。

在 LinkedIn 上找到我们：[*https://linkedin.com/company/oreilly-media*](https://linkedin.com/company/oreilly-media)

关注我们的 Twitter：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*http://youtube.com/oreillymedia*](http://youtube.com/oreillymedia)

关注作者的 Twitter：[*https://twitter.com/bmuschko*](https://twitter.com/bmuschko)

关注作者的 GitHub：[*https://github.com/bmuschko*](https://github.com/bmuschko)

关注作者的博客：[*https://bmuschko.com*](https://bmuschko.com)

# 致谢

每本书籍项目都是漫长的旅程，没有编辑团队和技术审阅者的帮助是不可能完成的。特别感谢 Jonathon Johnson、Kaslin Fields 和 Werner Dijkerman 提供的详细技术指导和反馈。我还要感谢 O’Reilly Media 的编辑 John Devins 和 Michele Cronin，他们的持续支持和鼓励。
