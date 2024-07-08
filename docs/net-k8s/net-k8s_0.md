# 前言

# 只是另一个封包

自从第一台两台计算机通过电缆连接在一起后，网络已经成为我们基础设施中至关重要的一部分。现在的网络具有多层复杂性来支持多种用例，而像 Mesosphere 和 Kubernetes 这样的项目和容器的出现并没有改变这一点。尽管 Kubernetes 的贡献者们试图为开发者们抽象出这些网络复杂性，但计算机科学就是这样，抽象层叠加抽象。Kubernetes 及其网络 API 是另一个抽象，使得部署应用程序变得更加简单和快速。那么，对于那些需要管理 Kubernetes 的管理员呢？本书旨在揭开 Kubernetes 引入的抽象背后的神秘面纱，指导管理员穿越复杂的层级，并帮助你意识到 Kubernetes 不仅仅是另一个封包。

# 本书的受众

根据 [451 Research](https://oreil.ly/2SlsD)，全球应用容器市场预计从 2019 年的 21 亿美元增长到 2022 年的 42 亿美元。容器市场的这种爆炸性增长凸显了 IT 专业人员在部署、管理和故障排除容器方面的需求。

本书旨在供新的网络、Linux 或集群管理员从头到尾阅读，同时也可供更有经验的 DevOps 工程师跳转到需要提升技能的特定主题使用。网络、Linux 和集群管理员需要熟悉如何在规模上操作 Kubernetes。

在本书中，读者将找到导航运行 Kubernetes 网络所需复杂层级的信息。本书将揭示 Kubernetes 引入的抽象，以便开发者在本地、云端和托管服务上的部署中有类似的体验。负责生产集群操作和网络运行时间的工程师可以使用本书来弥补他们在这些抽象知识方面的知识差距。

# What You Will Learn

通过本书的最后，读者将了解以下内容：

+   Kubernetes 网络模型

+   容器网络接口（CNI）项目及如何为其集群选择 CNI 项目

+   驱动 Kubernetes 的网络和 Linux 基元

+   驱动 Kubernetes 群集的抽象之间的关系

此外，读者还将能够做到以下几点：

+   部署和管理适用于 Kubernetes 集群的生产规模网络

+   在 Kubernetes 集群内部解决与网络相关的应用问题

# 本书中使用的约定

本书使用以下排版惯例：

*斜体*

指示新术语、网址、电子邮件地址、文件名和文件扩展名。

`Constant width`

用于程序列表，以及段落中引用程序元素，如变量或函数名，数据库，数据类型，环境变量，语句和关键字。

**`等宽粗体`**

显示用户应该按字面输入的命令或其他文本。

*`等宽斜体`*

显示应替换为用户提供的值或由上下文确定的值的文本。

###### 小贴士

这个元素表示提示或建议。

###### 注意

这个元素表示一般的注释。

###### 警告

这个元素表示警告或注意事项。

# 使用代码示例

附加材料（代码示例、练习等）可在[*https://github.com/strongjz/Networking-and-Kubernetes*](https://github.com/strongjz/Networking-and-Kubernetes)下载。

如果您有技术问题或在使用代码示例时遇到问题，请发送电子邮件至*bookquestions@oreilly.com*。

本书旨在帮助您完成工作。一般来说，如果本书提供了示例代码，您可以在您的程序和文档中使用它。除非您要复制大量代码，否则无需征得我们的许可。例如，编写一个使用本书中几个代码块的程序不需要许可。销售或分发 O’Reilly 图书的示例需要许可。引用本书并引用示例代码来回答问题不需要许可。将本书的大量示例代码整合到您产品的文档中需要许可。

我们感谢，但通常不需要署名。署名通常包括标题，作者，出版社和 ISBN。例如：“*Networking and Kubernetes* by James Strong and Vallery Lancey (O’Reilly). Copyright 2021 Strongjz tech and Vallery Lancey, 978-1-492-08165-4.”

如果您认为您对代码示例的使用超出了合理使用范围或上述许可，请随时通过*permissions@oreilly.com*与我们联系。

# O’Reilly 在线学习

###### 注意

超过 40 年来，[*O’Reilly Media*](http://oreilly.com)为公司的成功提供技术和商业培训、知识和见解。

我们独特的专家和创新者网络通过书籍、文章和我们的在线学习平台分享他们的知识和专业知识。O’Reilly 的在线学习平台为您提供按需访问的实时培训课程、深入学习路径、交互式编码环境以及 O’Reilly 和 200 多家其他出版商的大量文本和视频。有关更多信息，请访问[*http://oreilly.com*](http://oreilly.com)。

# 如何联系我们

请将有关本书的评论和问题发送给出版商：

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-998-9938（美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为这本书制作了一个网页，列出勘误、示例和任何额外信息。您可以访问这个页面[*https://oreil.ly/NetKubernetes*](https://oreil.ly/NetKubernetes)。

发送电子邮件至*bookquestions@oreilly.com*，评论或询问关于本书的技术问题。

关于我们的书籍和课程的新闻和信息，请访问[*http://oreilly.com*](http://oreilly.com)。

在 Facebook 上找到我们：[*http://facebook.com/oreilly*](http://facebook.com/oreilly)。

在 Twitter 上关注我们：[*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)。

在 YouTube 上关注我们：[*http://youtube.com/oreillymedia*](http://youtube.com/oreillymedia)。

# 致谢

作者们要感谢 O’Reilly Media 团队在帮助他们完成他们的第一本书的过程中的支持。Melissa Potter 在推动这一项目进展中功不可没。我们也要感谢 Thomas Behnken 在他的 Azure 专业知识方面给予的帮助。

**詹姆斯**: 卡伦，感谢你对我的信任，感谢你在他不相信自己时帮助他相信自己。温克，你是我进入这个领域工作的原因，我会永远感激你。安妮，我在学习英语时走了很长一段路，应该用大写字母来感谢你。詹姆斯还要感谢他生活中所有支持他的其他老师和教练。

**瓦莱瑞**: 我要感谢 SIG-Network 中的友好面孔，帮助我开始进入上游 Kubernetes。

最后，作者们要感谢 Kubernetes 社区；没有他们，这本书就不会存在。我们希望这能帮助所有希望采用 Kubernetes 的工程师进一步扩展知识。
