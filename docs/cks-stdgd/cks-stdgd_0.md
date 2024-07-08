# 序言

Kubernetes 认证计划自 2018 年以来已经存在五年。在这段时间里，安全性变得越来越重要，包括 Kubernetes 领域。最近，增加了[认证 Kubernetes 安全专家 (CKS)](https://www.cncf.io/certification/cks) 角色来满足需求。安全性可以有不同的方面，解决这些问题的方式可以非常多样化。这就是 Kubernetes 生态系统发挥作用的地方。除了 Kubernetes 内置的安全功能外，许多工具已经发展起来，帮助识别和修复安全风险。作为 Kubernetes 管理员，你需要熟悉广泛的概念和工具，以加固你的集群和应用程序。

CKS 认证计划旨在验证安全相关主题的能力，需要通过[认证 Kubernetes 管理员 (CKA)](https://www.cncf.io/certification/cka) 考试后才能注册。如果你对 Kubernetes 认证程序完全陌生，我建议首先了解 CKA 或[认证 Kubernetes 应用开发者 (CKAD)](https://www.cncf.io/certification/ckad) 计划。

在本学习指南中，我将探讨 CKS 考试涵盖的主题，以全面准备你通过认证考试。我们将研究何时以及如何应用 Kubernetes 的核心概念和外部工具来保护集群组件、集群配置以及在 Pod 中运行的应用程序。我还将提供一些提示，帮助你更好地准备考试，并分享我准备考试的个人经验的所有方面。

CKS 不同于其他认证的典型多项选择格式。它完全基于表现，并要求你在巨大的时间压力下展示对手头任务的深入知识。你准备好一次通过考试了吗？

# 适合读者

本书适合已经通过 CKA 考试并希望在安全领域扩展知识的任何人。考虑到注册 CKS 前需要通过 CKA 考试，你应该已经熟悉考试问题的格式和环境。第一章 简要回顾了考试课程的一般方面，但重点强调了适用于 CKS 考试的信息。如果你还没有参加 CKA 考试，我建议你先阅读[*认证 Kubernetes 管理员 (CKA) 学习指南*](https://oreil.ly/cka-study-guide)（O'Reilly）。这本书将为你提供开始学习 CKS 所需的基础。

# 学到什么

本书内容压缩了与 CKS 考试相关的最重要方面。不需要考虑云提供商特定的 Kubernetes 实现，如 AKS 或 GKE。鉴于 Kubernetes 中提供了大量的配置选项，几乎不可能涵盖所有用例和场景而不重复官方文档。建议考生参考[Kubernetes 文档](https://kubernetes.io/docs/home)以获取更广泛的知识。与 CKS 考试相关的外部工具（如 Trivy 或 Falco）仅在高层次上进行了介绍。请参考它们的文档以探索更多功能、功能和配置选项。

# 本书结构

本书的大纲严格遵循 CKS 课程。虽然可能存在更自然的教学结构来学习 Kubernetes 的一般知识，但课程大纲将帮助考生通过专注于特定主题来准备考试。因此，您可能会根据自己的知识水平交叉参考本书的其他章节。

请注意，本书仅涵盖与 CKS 考试相关的概念。不讨论基础的 Kubernetes 概念和原语。如果您希望深入了解，请参考 Kubernetes 文档或其他书籍。

Kubernetes 的实际经验对于通过考试至关重要。每章都包含名为“示例练习”的部分，其中包含练习题。这些问题的解决方案可以在附录中找到。

# 本书使用的约定

本书使用以下排版约定：

*斜体*

表示新术语、URL 和电子邮件地址。

`常量宽度`

用于文件名、文件扩展名和程序清单，以及段落中用于引用程序元素（如变量或函数名称、数据库、数据类型、环境变量、语句和关键字）。

**`常量宽度粗体`**

显示用户应直接输入的命令或其他文本。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般说明。

###### 警告

此元素表示警告或注意事项。

# 使用代码示例

本书中的一些代码片段使用反斜杠字符（`\`）来将单行代码分解为多行，以便适应页面。如果您直接从书中内容复制粘贴到终端或编辑器中，则需要手动整理代码。更好的选择是参考代码书的[GitHub 存储库](https://github.com/bmuschko/cks-study-guide)，那里已经有了正确的格式。

GitHub 存储库根据 Apache License 2.0 分发。代码可以在商业和开源项目中免费使用。如果在源代码中遇到问题或有疑问，请在[GitHub 问题跟踪器](https://oreil.ly/YTRzJ)中提出问题。我很乐意进行交流并解决可能出现的任何问题。

本书旨在帮助您完成工作。一般而言，如果本书提供示例代码，您可以在您的程序和文档中使用它。除非您复制了代码的大部分，否则无需获得我们的许可。例如，编写一个使用本书多个代码块的程序不需要许可。销售或分发 O’Reilly 书籍中的示例代码需要许可。引用本书并引用示例代码来回答问题不需要许可。将本书大量示例代码整合到您产品的文档中需要许可。我们感谢您的署名，但通常不要求署名。署名通常包括标题、作者、出版商和 ISBN。例如：“*Certified Kubernetes Security Specialist (CKS) Study Guide* by Benjamin Muschko (O’Reilly)。版权 2023 年 Automated Ascent, LLC, 978-1-098-13297-2。”

如果您认为您对代码示例的使用超出了公平使用或以上给出的权限，请随时通过*permissions@oreilly.com*与我们联系。

# O’Reilly 在线学习

###### 注意

超过 40 年来，[*O’Reilly Media*](http://oreilly.com)提供技术和商业培训、知识和见解，帮助企业成功。

我们独特的专家和创新者网络通过书籍、文章和我们的在线学习平台分享他们的知识和专业知识。O’Reilly 的在线学习平台为您提供按需访问的实时培训课程、深入的学习路径、交互式编码环境以及来自 O’Reilly 和 200 多家其他出版商的大量文本和视频。有关更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将关于本书的评论和问题发送至出版商：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   CA 95472 Sebastopol

+   800-889-8969（美国或加拿大）

+   707-829-7019（国际或本地）

+   707-829-0104（传真）

+   *support@oreilly.com*

+   [*https://www.oreilly.com/about/contact.xhtml*](https://www.oreilly.com/about/contact.xhtml)

我们有本书的网页，列出勘误、示例和任何额外信息。您可以访问此页面：[*https://oreil.ly/cks-study-guide*](https://oreil.ly/cks-study-guide)。

有关我们的书籍和课程的新闻和信息，请访问[*http://oreilly.com*](http://oreilly.com)。

在 LinkedIn 上找到我们：[*https://linkedin.com/company/oreilly-media*](https://linkedin.com/company/oreilly-media)

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*http://youtube.com/oreillymedia*](http://youtube.com/oreillymedia)

在 Twitter 上关注作者：[*https://twitter.com/bmuschko*](https://twitter.com/bmuschko)

在 GitHub 上关注作者：[*https://github.com/bmuschko*](https://github.com/bmuschko)

关注作者的博客：[*https://bmuschko.com*](https://bmuschko.com)

# 致谢

每本书籍项目都是一段漫长的旅程，没有编辑部和技术审阅人员的帮助是不可能完成的。特别感谢 Robin Smorenburg，Werner Dijkerman，Michael Kehoe 和 Liz Rice 提供的详细技术指导和反馈。我还要感谢 O’Reilly Media 的编辑 John Devins 和 Michele Cronin，他们一直以来的支持和鼓励。
