# 前言

欢迎来到*编程 Kubernetes*，感谢选择花费一些时间与我们在一起。在我们深入讨论之前，让我们迅速澄清一些行政和组织上的事务。我们还将分享撰写本书的动机。

# 适合阅读本书的对象

你可能是一名正在转向云原生的开发者，或者是希望从 Kubernetes 中获得最大利益的 AppOps 或命名空间管理员。原始设置已经不能满足你的需求，你可能已经了解了[扩展点](http://bit.ly/2XmoeKF)。很好，你来对地方了。

# 我们为什么写这本书

我们两个从 2015 年初开始贡献于、撰写关于、教授和使用 Kubernetes。我们开发了一些用于 Kubernetes 的工具和应用，并多次举办了关于在 Kubernetes 上进行开发的研讨会。有一天我们说，“为什么不写一本书呢？”这样更多的人可以异步地按照自己的节奏学习如何编程 Kubernetes。于是我们在这里。希望你阅读本书能和我们写作时一样有趣。

# 生态系统

从更大的视角来看，Kubernetes 生态系统仍处于早期阶段。虽然 Kubernetes 在 2018 年初已经确立自己作为管理容器（及其生命周期）的行业标准，但仍然需要关于如何编写本地应用的良好实践。基本的构建块，如[`client-go`](http://bit.ly/2L5cUMu)，自定义资源和云原生编程语言，已经就绪。然而，大部分知识还是部落性质的，分散在人们的头脑中，并且散布在成千上万个 Slack 频道和 StackOverflow 答案中。

###### 注意事项

在撰写本文时，Kubernetes 1.15 是最新的稳定版本。编译的示例应该能在较旧的版本（至少 1.12）上运行，但我们基于更新版本的库来编写代码，对应的是 1.14 版本。一些更高级的 CRD 功能需要在 1.13 或 1.14 版本的集群上运行，在第九章中甚至需要 1.15 版本。如果你无法访问到足够新的集群，强烈建议在本地工作站上使用[Minikube](http://bit.ly/2WT3k1l)或[kind](https://kind.sigs.k8s.io)。

# 您需要了解的技术

这本中级书籍需要读者对几个开发和系统管理概念有最基本的了解。在深入阅读之前，你可能需要复习一下以下内容：

包管理

本书中的工具通常有多个依赖项，您需要通过安装一些软件包来满足这些依赖项。因此，您需要了解您机器上的软件包管理系统。它可能是 Ubuntu/Debian 系统上的*apt*，CentOS/RHEL 系统上的*yum*，或者 macOS 上的*port*或*brew*。无论哪种情况，请确保您知道如何安装、升级和删除软件包。

Git

Git 已经确立了作为分布式版本控制标准的地位。如果你已经熟悉 CVS 和 SVN 但尚未使用 Git，那么你应该尝试一下。*[使用 Git 进行版本控制](http://shop.oreilly.com/product/0636920022862.do)*，作者 Jon Loeliger 和 Matthew McCullough（O’Reilly）是一个很好的开始。结合 Git，[GitHub 网站](http://github.com)是一个开始使用托管库的绝佳资源。要了解 GitHub，请查看[它们的培训课程](https://services.github.com)以及[相关的交互式教程](http://try.github.io)。

Go

Kubernetes 是用[Go 语言](http://golang.org)编写的。在过去几年中，Go 已经成为许多初创公司和许多系统相关的开源项目的首选新编程语言。本书不是关于教你 Go 语言，而是展示如何使用 Go 编程 Kubernetes。你可以通过各种资源学习 Go 语言，从[Go 官网的在线文档](https://golang.org/doc)到博客文章、演讲和大量的书籍。

# 本书使用的约定

本书中使用的以下印刷体约定：

*斜体*

表示新术语、URL、电子邮件地址、文件名和文件扩展名。

`等宽字体`

用于程序清单，以及段落内引用程序元素，如变量或函数名，数据库，数据类型，环境变量，语句和关键字。同时也用于命令和命令行输出。

**`等宽粗体`**

显示用户应直接输入的命令或其他文本。

*`等宽斜体`*

显示应该用用户提供的值或由上下文确定的值替换的文本。

###### 提示

这个元素表示提示或建议。

###### 注意

这个元素表示一般性的说明。

###### 警告

这个元素表示警告或注意。

# 使用代码示例

这本书的目的是帮助你完成工作。你可以在 GitHub 组织[这本书的页面](https://github.com/programming-kubernetes)找到整本书中使用的代码样例。

通常情况下，如果本书提供了示例代码，你可以在你的程序和文档中使用它们。除非你要复制代码的大部分内容，否则无需联系我们请求许可。例如，编写一个使用本书中几段代码的程序不需要许可。售卖或分发包含 O'Reilly 书籍示例的 CD-ROM 需要许可。引用本书并引用示例代码来回答问题不需要许可。将本书的大量示例代码整合到产品文档中需要许可。

我们感激但不要求署名。署名通常包括标题，作者，出版商和 ISBN。例如：“*使用 Kubernetes 进行编程*，作者 Michael Hausenblas 和 Stefan Schimanski（O’Reilly）。版权所有 2019 Michael Hausenblas 和 Stefan Schimanski。”

如果您认为您对代码示例的使用超出了公平使用或上述授权，请随时通过*permissions@oreilly.com*联系我们。

本书中使用的 Kubernetes 清单、代码示例和其他脚本可以通过[GitHub](https://github.com/programming-kubernetes)获取。您可以克隆这些仓库，转到相关章节和示例，直接使用这些代码。

# O’Reilly 在线学习

###### 注意

近 40 年来，[*O’Reilly Media*](http://oreilly.com)已经为公司的成功提供了技术和商业培训、知识和见解。

我们独特的专家和创新者网络通过图书、文章、会议和我们的在线学习平台分享他们的知识和专业知识。O’Reilly 的在线学习平台为您提供按需访问的实时培训课程、深入学习路径、交互式编码环境，以及来自 O’Reilly 和其他 200 多家出版商的大量文本和视频。欲了解更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将有关本书的评论和问题发送至出版商：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-998-9938（在美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为本书创建了一个网页，列出勘误、示例和任何额外信息。您可以访问此页面：[*https://oreil.ly/pr-kubernetes*](https://oreil.ly/pr-kubernetes)。

电子邮件*bookquestions@oreilly.com*以评论或提问关于本书的技术问题。

欲了解更多关于我们的图书、课程、会议和新闻的信息，请访问我们的网站：[*http://www.oreilly.com*](http://www.oreilly.com)。

在 Facebook 上找到我们：[*http://facebook.com/oreilly*](http://facebook.com/oreilly)

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*http://www.youtube.com/oreillymedia*](http://www.youtube.com/oreillymedia)

# 致谢

衷心感谢 Kubernetes 社区开发了如此出色的软件，并且是一群了不起的人们——开放、友善，总是乐于助人。此外，我们非常感谢我们的技术审阅者：Ahmed Belgana、Michael Gasch、Dimitris Gkanatsios、Mingding Han、Jess Males、Max Neunhöffer、Ewout Prangsma 和 Adrien Trouillaud。你们提供了非常宝贵和可操作的反馈，使这本书更易读和有用。感谢你们的时间和努力！

Michael 衷心感谢他出色且支持他的家人：我聪明而有趣的妻子 Anneliese；我们的孩子 Saphira、Ranya 和 Iannis；以及我们几乎仍然是小狗的 Snoopy。

Stefan 想要感谢他的妻子，Clelia，每当他再次“在写书”的时候，她总是非常支持和鼓励他。没有她，这本书就不会存在。如果你在书中发现错别字，很有可能它们是由两只猫，Nino 和 Kira，自豪地贡献的。

最后但同样重要的是，两位作者感谢 O'Reilly 团队，特别是 Virginia Wilson，她在书写过程中引导我们，确保我们按时交付，并且达到了预期的质量要求。
