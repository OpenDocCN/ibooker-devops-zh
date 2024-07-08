# 第一章：读者对象

本书适合希望准备 CKAD 考试的开发人员。内容涵盖考试大纲的所有方面，尽管需要具备 Kubernetes 架构及其概念的基本知识。

如果你对 Kubernetes 完全陌生，建议你首先阅读由 Brendan Burns、Joe Beda、Kelsey Hightower 和 Lachlan Evenson（O'Reilly 出版社）编写的《Kubernetes: Up and Running》，或者 Marko Lukša（Manning Publications）编写的《Kubernetes in Action》。

# 你将学到什么

这本书的内容总结了 CKAD 考试的最重要部分。考虑到 Kubernetes 中众多的配置选项，要在不重复官方文档的情况下覆盖所有用例和场景是不可能的。鼓励考生参考[Kubernetes 文档](https://kubernetes.io/docs/home)，作为更广泛了解的主要手册。

本书的大纲遵循 CKAD 课程。尽管通常学习 Kubernetes 有更自然的、有教育性的结构，但课程大纲将帮助考生集中精力准备考试，专注于特定主题。因此，根据你的现有知识水平，你将会参考本书的其他章节。请注意，本书仅涵盖与 CKAD 考试相关的概念。如果你希望深入了解，请参考 Kubernetes 文档或其他书籍。

在通过考试中，对 Kubernetes 的实际经验至关重要。每一章都包含一个名为“样例练习”的部分，包含练习问题。这些问题的解决方案可以在附录 A 中找到。

# 第二版的新内容

CNCF 定期更新所有 Kubernetes 认证，以跟上该领域的最新发展。2021 年 9 月，CKAD 课程经历了[重大改革](https://training.linuxfoundation.org/ckad-program-change-2021/)。现有主题的组织方式已经改变，并添加了新主题，使认证更加与 Kubernetes 实践者在真实场景中的应用相关。

与书的第一版相比，大约 80%的内容没有改变。新课程包括以下主题：

部署策略

部署策略的涵盖范围超出了直接由 Deployment 原语支持的范围。你需要理解如何实现和管理蓝/绿部署和金丝雀部署。

Helm

Helm 是一个工具，通过将配置文件组合成一个可重复使用的单一包，自动化捆绑、配置和部署 Kubernetes 应用程序。你需要了解如何使用 Helm 来发现和安装现有的 Helm 图表。

API 废弃

你需要了解 Kubernetes 的发布流程及其对已弃用和移除 API 的使用意义。你将学习如何处理需要切换到更新或替换的 API 版本的情况。

自定义资源定义 (CRDs)

CRD 允许通过创建自定义资源类型来扩展 Kubernetes API。你需要了解如何创建 CRD，以及如何管理基于 CRD 类型的对象。

认证、授权和准入控制

对 Kubernetes API 的每个调用都需要进行身份验证。作为日常使用 `kubectl` 的应用程序开发者，你需要理解如何管理和使用你的凭据。一旦身份验证成功，请求还需要通过授权阶段。你需要大致了解基于角色的访问控制（RBAC）的概念，该概念保护对 Kubernetes 资源的访问。准入控制是 CKA 和 Certified Kubernetes 安全专家 (CKS) 考试涵盖的主题，因此我只会浅尝辄止。

入口

你需要将在 Kubernetes 中运行的面向客户的应用程序暴露给外部消费者。Ingress 原语将 HTTPS 流量路由到多个 Service 后端之一。现在 Ingress 已成为 CKAD 课程的一部分。

此外，我在 附录 B 中包含了一个“考试复习指南”，该指南将课程主题与书籍中对应章节以及 Kubernetes 文档中的覆盖内容进行了映射。

CKAD 课程的早期版本包括两个已移除的主题：“创建和配置基本 Pod”和“了解如何使用标签、选择器和注释”。本书涵盖了这两个主题，尽管它们没有明确提到。Pod 对于在 Kubernetes 中运行工作负载至关重要，因此你绝对需要了解它们。第五章 解释了其详细内容。考虑到标签和标签选择对于理解 Deployment、Service 和 Network Policies 等原语非常重要，我保留了一个专门章节名为 标签和注释。

# 本书使用的约定

本书使用以下排版约定：

*Italic*

指示新术语、URL、电子邮件地址、文件名和文件扩展名。

`Constant width`

用于程序清单，以及段落内引用程序元素，如变量或函数名、数据库、数据类型、环境变量、语句和关键字。

**`Constant width bold`**

显示用户需要按照文字输入的命令或其他文本。

*`Constant width italic`*

显示应被用户提供值或根据上下文确定值的文本。

###### 提示

此元素表示提示或建议。

###### 注意

此元素表示一般说明。

###### 警告

This element indicates a warning or caution.

# Using Code Examples

The source code for all examples and exercises in this book is available on [GitHub](https://github.com/bmuschko/ckad-study-guide). The repository is distributed under the Apache License 2.0\. The code is free to use in commercial and open source projects. If you encounter an issue in the source code or if you have a question, open an issue in the [GitHub issue tracker](https://github.com/bmuschko/ckad-study-guide/issues). I’m happy to have a conversation and fix any issues that might arise.

This book is here to help you get your job done. In general, if example code is offered with this book, you may use it in your programs and documentation. You do not need to contact us for permission unless you’re reproducing a significant portion of the code. For example, writing a program that uses several chunks of code from this book does not require permission. Selling or distributing examples from O’Reilly books does require permission. Answering a question by citing this book and quoting example code does not require permission. Incorporating a significant amount of example code from this book into your product’s documentation does require permission.

We appreciate, but generally do not require, attribution. An attribution usually includes the title, author, publisher, and ISBN. For example: “*Certified Kubernetes Application Developer (CKAD) Study Guide*, by Benjamin Muschko (O’Reilly). Copyright 2024 Automated Ascent, LLC, 978-1-098-16949-7.”

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

+   707-827-0515 (international or local)

+   707-829-0104 (fax)

We have a web page for this book, where we list errata, examples, and any additional information. You can access this page at [*https://oreil.ly/ckad-2ed*](https://oreil.ly/ckad-2ed).

Email *bookquestions@oreilly.com* to comment or ask technical questions about this book.

欲了解我们的书籍和课程的最新消息，请访问[*http://oreilly.com*](http://oreilly.com)。

在 YouTube 上观看我们：[*http://youtube.com/oreillymedia*](http://youtube.com/oreillymedia)

在 LinkedIn 上找到我们：[*https://linkedin.com/company/oreilly-media*](https://linkedin.com/company/oreilly-media)

在 Twitter 上关注作者：[*https://twitter.com/bmuschko*](https://twitter.com/bmuschko)

在 GitHub 上关注作者：[*https://github.com/bmuschko*](https://github.com/bmuschko)

关注作者的博客：[*https://bmuschko.com*](https://bmuschko.com)

# 致谢

每本书项目都是一段漫长的旅程，没有编辑部和技术审阅者的帮助是不可能完成的。特别感谢 Jonathan Johnson、Bilgin Ibryam、Vladislav Bilay、Andrew Martin 和 Michael Levan，感谢他们详细的技术指导和反馈。我还要感谢 O'Reilly Media 的编辑 John Devins 和 Virginia Wilson，他们的持续支持和鼓励。
