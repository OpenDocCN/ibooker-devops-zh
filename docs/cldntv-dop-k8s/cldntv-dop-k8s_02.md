# 序言

在 IT 运维领域，DevOps 的关键原则已被广泛理解和采纳，但现在景观正在发生变化。一个名为 Kubernetes 的新应用平台迅速被全球各地各行各业的公司广泛采纳。随着越来越多的应用程序和业务从传统服务器迁移到 Kubernetes 环境，人们开始思考如何在这个新世界中实施 DevOps。

本书解释了在 Kubernetes 成为标准平台的云原生世界中，DevOps 的含义。它将帮助您从 Kubernetes 生态系统中选择最佳工具和框架。它还将呈现一种连贯的方式来使用这些工具和框架，提供在生产环境中实际运行的经过实战检验的解决方案。

## 我会学到什么？

你将了解 Kubernetes 是什么，它的起源是什么，以及它对软件开发和运维的未来意味着什么。你将学习容器的工作原理，如何构建和管理它们，以及如何设计云原生服务和基础设施。

你将理解自行构建和托管 Kubernetes 集群之间的权衡，以及使用托管服务的区别。你将了解到流行的 Kubernetes 安装工具如 kops 和 kubeadm 的能力、限制以及优缺点。你将详细了解来自亚马逊、谷歌和微软等公司的主要托管 Kubernetes 解决方案。

你将获得实际动手写和部署 Kubernetes 应用程序的经验，配置和操作 Kubernetes 集群，并使用 Helm 等工具自动化云基础设施和部署。您将学习 Kubernetes 对安全性、认证和权限的支持，包括基于角色的访问控制（RBAC）以及在生产环境中保护容器和 Kubernetes 的最佳实践。

你将学习如何使用 Kubernetes 设置持续集成和部署；了解 GitOps 的一些基本概念；如何进行数据备份和恢复；如何测试集群的符合性和可靠性；如何监控、追踪、记录和聚合指标；以及如何使您的 Kubernetes 基础设施具备可扩展性、弹性和成本效益。

为了演示我们所讨论的所有内容，我们将它们应用于一个非常简单的演示应用程序。您可以使用我们 Git 仓库中的代码跟随我们的所有示例。

## 这本书适合谁？

本书最直接适用于负责服务器、应用程序和服务的 IT 运维人员，以及负责构建新的云原生服务或将现有应用迁移到 Kubernetes 和云平台的开发人员。我们假设您对 Kubernetes 或容器没有任何先验知识 —— 不用担心，我们会逐步引导您。

即使是经验丰富的 Kubernetes 用户，在本书中仍然会找到许多有价值的内容：它涵盖了诸如 RBAC、持续部署、秘密管理和可观察性等高级主题。无论您的专业水平如何，我们希望您在这些页面中找到一些有用的东西。

## 这本书回答了哪些问题？

在计划和撰写本书时，我们与数百人讨论了云原生和 Kubernetes，包括行业领袖、专家和完全初学者。以下是他们表示希望本书解答的一些问题：

+   “我想知道为什么我应该投入时间学习这项技术。它将帮助我和我的团队解决什么问题？”

+   “Kubernetes 看起来很棒，但学习曲线相当陡峭。快速设置演示很容易，但是操作和故障排除似乎令人望而生畏。我们希望得到关于在现实世界中运行 Kubernetes 集群以及我们可能遇到的问题的坚实指导。”

+   “有主观建议会很有用。Kubernetes 生态系统有太多选项供初学团队选择。当存在多种完成同一任务的方法时，哪一种最好？我们如何选择？”

或许最重要的问题是：

+   “我如何在不破坏公司的情况下使用 Kubernetes？”

在撰写本书时，我们牢记了这些问题，以及许多其他问题，并尽力回答了它们。我们的表现如何？继续阅读以了解答案。

# 本书中使用的约定

本书使用以下排版约定：

*斜体*

表示新术语、网址、电子邮件地址、文件名和文件扩展名。

`等宽`

用于程序清单，以及在段落内用于引用程序元素，例如变量或函数名称、数据库、数据类型、环境变量、语句和关键字。

**`等宽粗体`**

显示用户应按照字面输入的命令或其他文本。

*`等宽斜体`*

显示应由用户提供值或由上下文确定值替换的文本。

###### 提示

这个元素表示一个提示或建议。

###### 注

这个元素表示一般注释。

###### 警告

这个元素表示警告或注意事项。

# 使用代码示例

附加材料（代码示例、练习等）可在[*https://github.com/cloudnativedevops/demo*](https://github.com/cloudnativedevops/demo)下载。

如果您有技术问题或在使用代码示例时遇到问题，请发送电子邮件至*bookquestions@oreilly.com*。

This book is here to help you get your job done. In general, if example code is offered with this book, you may use it in your programs and documentation. You do not need to contact us for permission unless you’re reproducing a significant portion of the code. For example, writing a program that uses several chunks of code from this book does not require permission. Selling or distributing examples from O’Reilly books does require permission. Answering a question by citing this book and quoting example code does not require permission. Incorporating a significant amount of example code from this book into your product’s documentation does require permission.

We appreciate, but do not require, attribution. An attribution usually includes the title, author, publisher, and ISBN. For example: “*Cloud Native DevOps with Kubernetes* by Justin Domingus and John Arundel (O’Reilly). Copyright 2022 John Arundel and Justin Domingus, 978-1-492-04076-7.”

If you feel your use of code examples falls outside fair use or the permission given above, feel free to contact us at *permissions@oreilly.com*.

# O’Reilly Online Learning

###### Note

For more than 40 years, [*O’Reilly Media*](http://oreilly.com) has provided technology and business training, knowledge, and insight to help companies succeed.

Our unique network of experts and innovators share their knowledge and expertise through books, articles, and our online learning platform. O’Reilly’s online learning platform gives you on-demand access to live training courses, in-depth learning paths, interactive coding environments, and a vast collection of text and video from O’Reilly and 200+ other publishers. For more information, visit [*http://oreilly.com*](http://oreilly.com).

# How to Contact Us

Please address comments and questions concerning this book to the publisher:

+   O’Reilly Media, Inc.

+   1005 Gravenstein Highway North

+   Sebastopol, CA 95472

+   800-998-9938 (in the United States or Canada)

+   707-829-0515 (international or local)

+   707-829-0104 (fax)

We have a web page for this book, where we list errata, examples, and any additional information. You can access this page at [*http://bit.ly/cloud-nat-dev-ops*](http://bit.ly/cloud-nat-dev-ops).

Email *bookquestions@oreilly.com* to comment or ask technical questions about this book.

For news and information about our books and courses, visit [*http://oreilly.com*](http://oreilly.com).

Find us on Facebook: [*http://facebook.com/oreilly*](http://facebook.com/oreilly)

Follow us on Twitter: [*http://twitter.com/oreillymedia*](http://twitter.com/oreillymedia)

Watch us on YouTube: [*http://youtube.com/oreillymedia*](http://youtube.com/oreillymedia)

## Acknowledgments

我们衷心感谢许多人，在这本书的初稿阶段阅读并给予我们宝贵的反馈和建议，或以其他方式帮助我们，包括（但不限于）Abby Bangser，Adam J. McPartlan，Adrienne Domingus，Alexis Richardson，Aron Trauring，Camilla Montonen，Gabe Medrash，Gabriell Nascimento，Hannah Klemme，Hans Findel，Ian Crosby，Ian Shaw，Ihor Dvoretskyi，Ike Devolder，Jeremy Yates，Jérôme Petazzoni，Jessica Deen，John Harris，Jon Barber，Kitty Karate，Marco Lancini，Mark Ellens，Matt North，Michel Blanc，Mitchell Kent，Nicolas Steinmetz，Nigel Brown，Patrik Duditš，Paul van der Linden，Philippe Ensarguet，Pietro Mamberti，Richard Harper，Rick Highness，Sathyajith Bhat，Suresh Vishnoi，Thomas Liakos，Tim McGinnis，Toby Sullivan，Tom Hall，Vincent De Smet 和 Will Thames.
