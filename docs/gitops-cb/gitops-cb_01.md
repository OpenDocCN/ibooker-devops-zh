# 序言

我们为开发者、DevOps 工程师、可靠性工程师（SRE）或处理 Kubernetes 的平台工程师撰写了本书。无论您是哪一类构建者，我们希望分享我们从现场和社区中学到的关于流水线和 CI/CD 工作负载的最新 Kubernetes 自动化见解。本书包含了 Kubernetes 和云原生生态系统中最受欢迎的软件和工具的全面清单，旨在提供一系列可能有助于您日常工作或值得进一步探索的实用配方。在实施 Kubernetes 自动化方面，我们没有特定的技术或项目的依恋，但对于一些选择我们有着自己的看法，以提供简洁的 GitOps 路径。

本书按照 GitOps 原则，从基础到 Kubernetes 生态系统中的高级主题进行了顺序排列的章节。我们希望这些配方对您的项目有价值并激发灵感！

+   第一章介绍了 GitOps 原则以及它们为何在任何新的 IT 项目中持续变得更加普遍和重要。

+   第二章涵盖了在 Kubernetes 集群中运行这些配方所需的安装要求。Git、容器注册表、容器运行时和 Kubernetes 等概念和工具对于这一旅程至关重要。

+   第三章全面介绍了容器及其在今天的应用开发和部署中的重要性。Kubernetes 是一个容器编排平台；然而，它不能直接构建容器。因此，我们将提供一系列使用云原生社区中最流行工具制作容器应用的实用配方清单。

+   第四章概述了 Kustomize，这是一款管理 Kubernetes 资源的流行工具。Kustomize 具有互操作性，在 CI/CD 流水线中经常被使用。

+   第五章探索了 Helm，一款将应用程序打包在 Kubernetes 中的时尚工具。Helm 还是一个模板系统，您可以用来在 CI/CD 工作负载中部署应用程序。

+   第六章详细介绍了针对 Kubernetes 的云原生 CI/CD 系统。它提供了使用 Tekton 进行持续集成的全面配方列表，Tekton 是 Kubernetes 原生的 CI/CD 系统。此外，还涵盖了其他工具，如 Drone 和 GitHub Actions。

+   第七章开启了本书中 GitOps 纯粹部分，专注于使用 Argo CD 进行持续部署，Argo CD 是一款流行的 Kubernetes GitOps 工具。

+   第八章 深入介绍了使用 Argo CD 进行 GitOps 的高级主题，如秘密管理、渐进式应用交付和多集群部署。这总结了您今天和明天可能会使用的最常见的用例和架构，遵循 GitOps 方法。

# 本书中使用的约定

本书使用以下印刷约定：

*斜体*

指示新术语、URL、电子邮件地址、文件名和文件扩展名。

`等宽字体`

用于程序清单，以及在段落内部引用程序元素，如变量或函数名、数据库、数据类型、环境变量、语句和关键字。

**`等宽粗体`**

显示用户应按字面意思输入的命令或其他文本。

*`等宽斜体`*

显示应替换为用户提供的值或上下文确定值的文本。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般注释。

###### 警告

此元素指示警告或注意事项。

# 使用代码示例

本书附带的补充材料（代码示例、练习等）可在[*https://github.com/gitops-cookbook*](https://github.com/gitops-cookbook)下载。

如果您有技术问题或在使用代码示例时遇到问题，请发送电子邮件至*bookquestions@oreilly.com*。

本书旨在帮助您完成工作。一般情况下，如果本书提供示例代码，您可以在您的程序和文档中使用它。除非您复制了代码的大部分，否则您无需联系我们以获得许可。例如，编写使用本书多个代码片段的程序不需要许可。销售或分发 O’Reilly 图书的示例则需要许可。引用本书并引用示例代码来回答问题不需要许可。将本书中大量示例代码整合到产品文档中需要许可。

我们感谢，但通常不要求归属。归属通常包括标题、作者、出版商和 ISBN。例如：“*GitOps Cookbook* 由 Natale Vinto 和 Alex Soto Bueno（O’Reilly）著。版权所有 2023 年 Natale Vinto 和 Alex Soto Bueno，978-1-492-09747-1。”

如果您觉得您对代码示例的使用超出了公平使用范围或以上授予的权限，请随时通过*permissions@oreilly.com*联系我们。

# O’Reilly 在线学习

###### 注意

40 多年来，[*O’Reilly Media*](http://oreilly.com) 提供技术和商业培训、知识和见解，帮助企业成功。

我们独特的专家和创新者网络通过书籍、文章和我们的在线学习平台分享他们的知识和专业知识。O’Reilly 的在线学习平台为您提供按需访问的实时培训课程、深入学习路径、交互式编码环境以及来自 O’Reilly 和其他 200 多个出版商的大量文本和视频。欲了解更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

关于本书的评论和问题，请联系出版商：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-998-9938 (美国或加拿大)

+   707-829-0515 (国际或本地)

+   707-829-0104 (传真)

我们为这本书制作了一个网页，上面列出了勘误、示例和任何额外信息。你可以访问这个页面：[*https://oreil.ly/gitops-cookbook*](https://oreil.ly/gitops-cookbook)。

发送邮件至 *bookquestions@oreilly.com* 评论或提问关于这本书的技术问题。

欲了解有关我们的书籍和课程的新闻和信息，请访问[*https://oreilly.com*](https://oreilly.com)。

在 LinkedIn 上找到我们：[*https://linkedin.com/company/oreilly-media*](https://linkedin.com/company/oreilly-media)。

在 Twitter 上关注我们：[*https://twitter.com/oreillymedia*](https://twitter.com/oreillymedia)。

在 YouTube 上观看我们：[*https://youtube.com/oreillymedia*](https://youtube.com/oreillymedia)。

# 致谢

我们特别感谢我们的技术审阅员彼得·米隆和安迪·布洛克，他们的准确审查帮助我们改进了这本书的阅读体验。同时也感谢 O’Reilly 全程对我们的帮助。非常感谢我们的同事奥布里·穆赫拉克和科琳·洛布纳，他们在出版这本书时给予了极大的支持。还要感谢卡梅什·桑帕斯和所有在早期版本中给予意见和建议的人们——非常感谢你们的贡献！

## Alex Soto

在这个充满挑战的时期，我要感谢圣塔（今年是的），乌里（别停止音乐），圭里（一个骑行者），加维纳，加比（感谢支持），以及埃德加和埃斯特（生活在星期五尤其美好）；我的朋友埃德森、塞比（最好的旅伴），伯尔（我从你身上学到了很多），卡梅什和所有红帽开发团队，我们是最棒的。

Jonathan Vila, Abel Salgado 和 Jordi Sola，感谢你们关于 Java 和 Kubernetes 的精彩交流。

最后但同样重要的是，我要感谢安娜一直在这里；感谢我的父母米利和拉蒙为我购买了第一台电脑；感谢我的女儿阿达和亚历山德拉，“你们是我眼中的宝贝。”

## 纳塔莱·温托

特别感谢阿莱西亚，在我写这本书期间给予我的耐心和激励。还要感谢我的父母为我所做的一切，谢谢你们，你们是最棒的！
