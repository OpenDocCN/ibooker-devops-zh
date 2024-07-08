# 前言

欢迎阅读*Kubernetes Cookbook*，感谢您选择了这本书！通过这本书，我们希望能帮助您解决围绕 Kubernetes 的具体问题。我们编写了 100 多个配方，涵盖了诸如设置集群、使用 Kubernetes API 对象管理容器化工作负载、使用存储原语、配置安全性等主题。无论您是初次接触 Kubernetes 还是已经使用了一段时间，我们希望您能在这里找到一些有用的内容，以改善您对 Kubernetes 的体验和使用。

# 适合阅读本书的人群

本书适用于 DevOps 领域的任何人。您可能是一个应用程序开发人员，偶尔需要与 Kubernetes 交互，或者是一个平台工程师，为组织中的其他工程师创建可重用的解决方案，或者介于两者之间。本书将帮助您成功地在 Kubernetes 丛林中导航，从开发到生产。它涵盖了核心 Kubernetes 概念以及来自更广泛生态系统的解决方案，这些解决方案几乎已成为行业中的事实标准。

# 我们为什么写这本书

我们多年来一直是 Kubernetes 社区的一部分，看到初学者甚至更高级用户遇到的许多问题。我们希望分享我们在生产环境中运行 Kubernetes 以及在 Kubernetes 上开发的知识——即，为核心代码库或生态系统做出贡献，并编写在 Kubernetes 上运行的应用程序所积累的经验。考虑到自第一版书籍出版以来，Kubernetes 的采用率一直在增长，因此着手撰写本书的第二版是完全合理的。

# 导读本书

本食谱书包含 15 章。每一章由按照标准 O'Reilly 配方格式编写的配方组成（问题、解决方案、讨论）。您可以从头到尾阅读本书，也可以跳转到特定章节或配方。每个配方都是独立的，当需要理解其他配方中的概念时，会提供适当的参考。索引也是一个非常强大的资源，因为有时一个配方也展示了一个特定的命令，索引突出显示这些联系。

# Kubernetes 版本说明

在撰写本文时，[Kubernetes 1.27](https://oreil.ly/3b2Ta)是最新的稳定版本，于 2023 年 4 月底发布，这也是我们在整本书中使用的基线版本。然而，这里提出的解决方案通常适用于旧版本；如果情况不是这样，我们将明确指出，并提到所需的最低版本。

Kubernetes 按照每年三次发布的节奏进行。发布周期大约为 15 周；例如，1.26 版本在 2022 年 12 月发布，1.27 版本在 2023 年 4 月发布，1.28 版本在 2023 年 8 月发布，本书进入生产阶段时。Kubernetes 的[发布版本指南](https://oreil.ly/9eFLs)表明，您可以预期对最新三个次要版本的功能提供支持。Kubernetes 社区支持大约 14 个月的活跃补丁发布系列。这意味着 1.27 版本中的稳定 API 对象将至少支持到 2024 年 6 月。然而，因为本书中的示例通常仅使用稳定的 API，如果您使用更新的 Kubernetes 版本，这些示例仍应有效。

# 技术你需要理解

这本中级书籍需要您对几个开发和系统管理概念有基本的理解。在深入阅读本书之前，您可能希望复习以下内容：

bash（Unix shell）

这是 Linux 和 macOS 上的默认 Unix shell。熟悉 Unix shell，如编辑文件、设置文件权限和用户权限、在文件系统中移动文件以及进行一些基本的 shell 编程，将是有益的。有关一般介绍，请参考卡梅隆·纽汉姆（Cameron Newham）的*[《学习 bash Shell》](http://shop.oreilly.com/product/9780596009656.do )*，第三版，或卡尔·阿尔宾（Carl Albing）和 JP Vossen 的*[《bash Cookbook》](http://shop.oreilly.com/product/0636920058304.do )*，第二版，均由 O’Reilly 出版。

包管理

本书中的工具通常有多个依赖项，需要通过安装一些软件包来满足。因此，您的机器上需要了解软件包管理系统。例如，在 Ubuntu/Debian 系统上可能是*apt*，在 CentOS/RHEL 系统上可能是*yum*，在 macOS 上可能是*Homebrew*。无论如何，请确保您知道如何安装、升级和删除软件包。

Git

Git 已经成为分布式版本控制的标准。如果您还不熟悉 Git，我们推荐阅读 Prem Kumar Ponuthorai 和 Jon Loeliger 合著的*[《使用 Git 进行版本控制》](https://www.oreilly.com/library/view/version-control-with/9781492091189/)*，第三版（O’Reilly）。除了 Git，[GitHub 网站](http://github.com)是一个创建托管库的绝佳资源。要了解 GitHub，请查看[GitHub 培训套件网站](http://training.github.com)。

Go

Kubernetes 是用 Go 编写的。Go 已经在 Kubernetes 社区及其他领域广泛应用作为一种流行的编程语言。本书不涉及 Go 编程，但介绍了如何编译几个 Go 项目。有一些关于如何设置 Go 工作空间的基本理解将会很有帮助。如果您想了解更多，可以从 O’Reilly 的视频培训课程*[《Go 编程入门》](https://learning.oreilly.com/videos/introduction-to-go/9781491913871)*开始。

# 本书使用的约定

本书使用以下印刷约定：

*斜体*

表示新术语、URL、电子邮件地址、文件名和文件扩展名。

`等宽`

用于程序清单，以及在段落内用来指代程序元素，如变量或函数名称、数据库、数据类型、环境变量、语句和关键字。还用于命令和命令行输出。

**`等宽粗体`**

显示用户需要直接键入的命令或其他文本。

*`等宽斜体`*

显示应由用户提供值或由上下文确定的值替换的文本。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般说明。

###### 警告

此元素表示警告或注意事项。

# 使用代码示例

本书提供下载的补充材料（Kubernetes 清单、代码示例、练习等）可在[*https://github.com/k8s-cookbook/recipes*](https://github.com/k8s-cookbook/recipes)获取。您可以克隆此存储库，转到相关章节和配方，并直接使用代码：

```
$ git clone https://github.com/k8s-cookbook/recipes

```

###### 注意

此存储库中的示例并非旨在表示优化设置以在生产中使用。它们为您提供了运行示例中所述示例的基本最低要求。

如果您在使用代码示例时遇到技术问题或问题，请发送电子邮件至*support@oreilly.com*。

本书旨在帮助您完成工作。通常情况下，如果本书提供示例代码，则可以在您的程序和文档中使用它。除非您重复使用本书中的大部分代码，否则不需要联系我们进行许可。例如，编写一个使用本书中几个代码块的程序不需要许可。出售或分发包含 O’Reilly 书籍示例的 CD-ROM 需要许可。通过引用本书回答问题并引用示例代码不需要许可。将本书中大量示例代码合并到您产品的文档中需要许可。

我们感谢，但不需要署名。署名通常包括标题、作者、出版商和 ISBN。例如：“*Kubernetes Cookbook*，作者 Sameer Naik、Sébastien Goasguen 和 Jonathan Michaux（O’Reilly）。版权所有 2024 年 CloudTank SARL，Sameer Naik 和 Jonathan Michaux，978-1-098-14224-7。”

如果您觉得您使用的代码示例超出了合理使用范围或上述许可，请随时通过*permissions@oreilly.com*与我们联系。

# O’Reilly 在线学习

###### 注意

40 多年来，[*O’Reilly Media*](https://oreilly.com) 提供技术和商业培训、知识和洞察力，帮助公司取得成功。

我们独特的专家和创新者网络通过书籍、文章和我们的在线学习平台分享他们的知识和专业知识。O'Reilly 的在线学习平台为您提供按需访问的实时培训课程、深入学习路径、交互式编码环境，以及来自 O'Reilly 和其他 200 多家出版商的大量文本和视频。欲了解更多信息，请访问[*https://oreilly.com*](https://oreilly.com)。

# 如何联系我们

请将有关本书的评论和问题发送至出版商：

+   O'Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-889-8969（美国或加拿大）

+   707-829-7019（国际或当地）

+   707-829-0104（传真）

+   *support@oreilly.com*

+   [*https://www.oreilly.com/about/contact.html*](https://www.oreilly.com/about/contact.html)

我们为本书设置了一个网页，列出勘误、示例以及其他任何附加信息。您可以访问此页面：[*https://oreil.ly/kubernetes-cookbook-2e*](https://oreil.ly/kubernetes-cookbook-2e)。

有关我们的书籍和课程的新闻和信息，请访问[*https://oreilly.com*](https://oreilly.com)。

在 LinkedIn 上找到我们：[*https://linkedin.com/company/oreilly-media*](https://linkedin.com/company/oreilly-media)。

在 Twitter 上关注我们：[*https://twitter.com/oreillymedia*](https://twitter.com/oreillymedia)。

在 YouTube 上关注我们：[*https://youtube.com/oreillymedia*](https://youtube.com/oreillymedia)。

# 致谢

感谢整个 Kubernetes 社区开发如此出色的软件，并成为一群伟大的人——开放、友善且总是乐于助人。

Sameer 和 Jonathan 很荣幸与 Sébastien 共同合作撰写本书的第二版。我们对 Roland Huß、Jonathan Johnson 和 Benjamin Muschko 提供的审阅表示感谢，他们在改进最终产品方面提供了宝贵的帮助。我们还感谢 O'Reilly 的编辑 John Devins、Jeff Bleiel 和 Ashley Stussy，与他们合作非常愉快。
