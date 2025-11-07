# 前置材料

## 前言

我从事计算机安全工作近 40 年，过去 20 年专注于容器技术。大约 10 年前，当 Docker 出现时，它引发了人们在网上分发和运行应用程序方式的革命。在我为 Docker 工作时，我觉得它本可以设计得更好。与运行在 root 权限下的守护进程一起工作，然后添加越来越多的守护进程，感觉这是一种错误的方法。相反，我觉得我们可以使用低级操作系统概念来创建一个工具，以相同的方式运行相同的容器化应用程序，但具有更高的安全性和更少的权限。带着这个想法，我在红帽公司的团队着手构建一系列工具，以帮助开发人员和管理员以最安全的方式运行容器。Podman 就是从这个努力中诞生的。

我在 2000 年代初开始写关于 SELinux 等主题的博客，并且从那时起一直在写文章。多年来，我写了数百篇关于容器和安全的文章，但我希望将想法整合并描述 Podman 技术，以便我可以向用户和客户推荐一本书。

本书介绍了 Podman 及其使用方法。它还深入探讨了我们所利用的 Linux 操作系统的技术和不同部分。由于我是一个安全工程师，我还用几章的篇幅描述了容器安全的工作原理。阅读这本书应该能让你更好地理解容器是什么，它们是如何工作的，以及如何使用 Podman 的不同功能。你甚至还会学到更多关于 Docker 的知识。随着 Podman 越来越受欢迎并渗透到你的基础设施中，这本书将成为一个实用的参考，引导你的道路。

## 致谢

我感谢所有帮助我写这本书的人。这包括 Podman 团队成员，他们写的文章帮助我理解了一些我不完全理解的技术，并帮助构建了一个伟大的产品。感谢 Brent Baude、Matt Heon、Valentin Rothberg、Giuseppe Scrivano、Urvashi Mohnani、Nalin Dahyabhai、Lokesh Mandvekar、Miloslav Trmac、Jason Greene、Jhon Honce、Scott McCarty、Tom Sweeney、Ashley Cui、Ed Santiago、Chris Evich、Aditya Rajan、Paul Holzinger、Preethi Thomas 和 Charlie Doern。我还想感谢那些使 Linux 容器和 Podman 成为可能的无数开源贡献者。

我感谢 Manning 出版社的整个团队，但特别感谢 Toni Arritola。Toni 教会了我如何更好地集中思想，并在这次旅程中成为了一位伟大的合作伙伴。她不得不应对我这样一个老数学专业毕业生，我从未擅长写作，但她帮助使这本书成为可能。

向所有审稿人——Alain Lompo、Alessandro Campeis、Allan Makura、Amanda Debler、Anders Björklund、Andrea Monacchi、Camal Cakar、Clifford Thurber、Conor Redmond、David Paccoud、Deepak Sharma、Federico Kircheis、Frans Oilinki、Gowtham Sadasivam、Ibrahim Akkulak、James Liu、James Nyika、Jeremy Chen、Kent Spillner、Kevin Etienne、Kirill Shirinkin、Kosmas Chatzimichalis、Krzysztof Kamyczek、Larry Cai、Michael Bright、Mladen Knežić、Oliver Korten、Richard Meinsen、Roman Zhuzha、Rui Liu、Satadru Roy、Seung-jin Kim、Simeon Leyzerzon、Simone Sguazza、Syed Ahmed、Thomas Peklak 和 Vivek Veerappan——表示感谢，你们的建议帮助使这本书变得更好。

## 关于本书

《Podman 实战》描述了用户如何构建、管理和运行容器。我写这本书的目标是解释如何轻松地将你在 Docker 中学到的技能转移到 Podman 上，以及如果你之前从未使用过容器引擎，使用 Podman 是多么容易。此外，《Podman 实战》还教你如何使用高级功能，如 pods，并指导你走向构建可在 Kubernetes 边缘或内部运行的应用程序的道路。最后，《Podman 实战》解释了 Linux 内核中用于隔离容器与系统以及与其他容器之间的所有安全功能。

### 谁应该阅读这本书？

《Podman 实战》是为那些希望理解、开发和与容器一起工作的软件开发者以及需要在生产环境中运行容器的系统管理员所写的。阅读这本书将使你对容器是什么有更深入的了解。了解 Linux 进程以及熟悉使用 Linux shell 是充分利用本书的必要条件。

本书应该为每个人在他们的容器使用之旅中提供一些内容。对 Docker 有深入了解的用户将了解 Podman 的 Docker 中不可用的高级功能，并将对 Docker 的工作原理有更深入的理解。新手用户将学习容器和 pods 的基础知识。

### 本书如何组织：路线图

《Podman 实战》分为四部分和六个附录：

+   第一部分，“基础”，包含四章，为读者提供了 Podman 的介绍。第一章解释了 Podman 的功能、为什么它被创建以及为什么它很重要。接下来的两章介绍了命令行界面以及如何在容器中使用卷。最后，第四章介绍了 pods 的概念以及 Podman 如何与它们一起工作。这些章节中应该有适合每个人的内容，但如果你有丰富的 Docker 经验，你应该能够快速浏览第二章的大部分内容。

+   第二部分，“设计”，包含两个章节，其中我深入探讨了 Podman 的设计。您将了解无根容器及其工作原理，并在这些章节结束时对用户命名空间和无根容器的安全性有更深入的理解。您还将学习如何自定义 Podman 环境的配置。

+   第三部分，“高级主题”，包含三个章节，并超越了 Podman 的基础知识。在第七章中，您将了解 Podman 如何通过与其与 systemd 的集成在生产环境中工作。它涵盖了在容器内运行 systemd 以及如何将其用作容器管理器。您将学习如何使用 Podman 容器设置边缘服务器，其中 systemd 管理容器的生命周期。Podman 使您能够轻松生成 systemd 单元文件，以帮助您将容器化应用程序投入生产。在第八章中，您将了解如何使用 Podman 将容器移动到 Kubernetes。Podman 支持使用与 Kubernetes 相同的 YAML 文件启动容器，以及从当前容器生成 Kubernetes YAML 的能力。在第九章中，您将看到 Podman 作为服务运行，允许远程访问 Podman 容器。将 Podman 作为服务使用允许您使用其他编程语言和工具来管理 Podman 容器。您将了解 `docker-compose` 如何与 Podman 容器协同工作。您还将学习如何使用 Python 库（如 podman-py 和 docker-py）与 Podman 服务通信以管理容器。

+   第四部分，“容器安全”，包含两个章节，其中我讨论了重要的安全考虑因素。第十章涵盖了用于确保容器隔离的功能。本章涵盖了 Linux 的安全子系统，如 SELinux、seccomp、Linux 能力、内核文件系统和命名空间。第十一章随后检查了我在尽可能安全地运行容器时认为的最佳实践。

此外，还有六个附录涵盖了与 Podman 相关的主题：

+   附录 A 涵盖了所有与 Podman 相关的工具，包括 Buildah、Skopeo 和 CRI-O。

+   附录 B 深入探讨了 Podman 以及 Docker 可用的不同 OCI 运行时，包括 `runc`、`crun`、Kata 和 gVisor。

+   附录 C 描述了如何将 Podman 安装到您的本地系统，无论该系统是 Linux、Mac 还是 Windows。

+   附录 D 描述了 Podman 开源社区以及如何加入。

+   附录 E 和 F 深入探讨了在 Mac 和 Windows 系统上运行 Podman。

### liveBook 讨论论坛

每购买一本《Podman in Action》都包括免费访问 liveBook，曼宁的在线阅读平台。使用 liveBook 的独特讨论功能，您可以在全球范围内或特定章节或段落中附加评论。为自己做笔记、提问和回答技术问题，以及从作者和其他用户那里获得帮助都非常简单。要访问论坛，请访问[`livebook.manning.com/book/podman-in-action/discussion`](https://livebook.manning.com/book/podman-in-action/discussion)。您还可以在[`livebook.manning.com/discussion`](https://livebook.manning.com/discussion)了解更多关于曼宁论坛和行为准则的信息。

曼宁对我们读者的承诺是提供一个平台，在这里个人读者之间以及读者与作者之间可以进行有意义的对话。这并不是对作者参与特定数量活动的承诺，作者对论坛的贡献仍然是自愿的（且未付费）。我们建议您尝试向他提出一些挑战性的问题，以免他的兴趣转移！只要这本书还在印刷中，论坛和先前讨论的存档将可通过出版社的网站访问。

### 作者在线

您可以在 Twitter 和 GitHub 上关注丹尼尔·沃尔什 @rhatdan。他经常在[`www.redhat.com/sysadmin/users/dwalsh`](https://www.redhat.com/sysadmin/users/dwalsh)以及几个其他网站上写博客。YouTube 上有许多丹尼尔·沃尔什发表的演讲视频。

## 关于作者

![丹尼尔·沃尔什](img/FM-Daniel_Walsh.png)

丹尼尔·沃尔什领导了创建 Podman、Buildah、Skopeo、CRI-O 及其相关工具的团队。丹自 2001 年 8 月加入红帽公司以来，担任高级杰出工程师。他在计算机安全领域工作了 40 多年。丹在红帽公司领导容器团队之前，曾领导开发 SELinux，因此有时被称为 SELinux 先生。丹在圣十字学院获得数学学士学位，在伍斯特理工学院获得计算机科学硕士学位。您可以在 Twitter 和 GitHub 上找到他 @rhatdan。您可以通过 dwalsh@redhat.com 给他发邮件。

## 关于封面插图

《Podman in Action》封面上的插图标题为“La vandale”，或“破坏者”，取自雅克·格拉塞·德·圣索沃尔于 1797 年出版的作品集。每一幅插图都是手工精心绘制和着色的。

在那个时代，人们通过他们的服饰就能轻易地识别出他们的居住地以及他们的职业或社会地位。曼宁通过基于几个世纪前丰富多样的地域文化的书封面来庆祝计算机行业的创新精神和主动性，这些文化通过如这一系列图片般的作品得以重现。
