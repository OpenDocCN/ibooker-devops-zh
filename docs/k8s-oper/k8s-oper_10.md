# 第十章：参与

Operator 框架中的所有组件，包括 Operator SDK、Operator Lifecycle Manager 和 Operator Metering，都仍处于生命周期的早期阶段。有多种方式可以参与它们的开发，从简单的提交 bug 报告到成为积极的开发者都可以。

与 Operator Framework 的用户和开发者互动的最简单方式之一是通过其[专业兴趣小组](https://groups.google.com/forum/#!forum/operator-framework)，简称 SIG。该 SIG 使用邮件列表讨论包括即将发布的信息、最佳实践和用户问题在内的各种主题。您可以通过他们的网站免费加入该 SIG。

要进行更直接的互动，[Kubernetes Slack 团队](https://kubernetes.slack.com/)是一个活跃的用户和开发者社区。特别是“kubernetes-operators”频道涵盖了与本书相关的主题。

Operator Framework 的 [GitHub 组织](https://oreil.ly/8iDG1)包含每个组件的项目仓库。还有多种补充仓库，例如 [Operator SDK 示例仓库](https://oreil.ly/CYhac)，进一步帮助 Operator 的开发和使用。

# 功能请求和报告 Bug

参与 Operator Framework 最简单但也非常有价值的方式之一是提交 bug 报告。框架项目团队使用 GitHub 内置的问题跟踪来对未解决的问题进行分类和修复。您可以在 GitHub 项目页面的 Issues 标签下找到每个特定项目的跟踪器。例如，Operator SDK 的问题跟踪器可以在 [Operator Framework GitHub 仓库](https://oreil.ly/l6eUM) 找到。

此外，项目团队使用问题跟踪器来跟踪功能请求。新问题按钮提示提交者在 bug 报告和功能请求之间进行选择，并自动适当地打上标签。提交功能请求提供了广泛的用例，并根据社区需求推动项目方向。

在提交新问题时，有一些一般原则^(1)需要记住：

+   *要具体.* 对于 bug，请尽可能提供关于运行环境的详细信息，包括项目版本和集群详情。如果可能，请提供详细的复现步骤。对于功能请求，请从包括所请求功能解决的用例开始。这有助于功能优先级的确定，并帮助团队决定是否有更好或现有的方式来满足请求。

+   *保持范围限制在一个单一的 bug 上。* 多个报告比单一、多方面的问题报告更容易进行分类和跟踪。

+   *请尽量选择适用的项目。* 例如，如果问题特别适用于与 OLM 一起工作，请在该存储库中创建问题。对于一些错误，不总是能确定问题的来源。在这些情况下，您可以选择最适用的项目存储库，让团队适当地对其进行分类。

+   *如果找到现有问题，请使用该问题。* 使用 GitHub 的问题跟踪器搜索功能，看看是否存在类似的错误或功能请求，然后再创建新报告。此外，请检查已关闭问题的列表，如果可能，请重新打开现有的错误。

# 贡献

当然，如果您习惯于处理代码，欢迎贡献源代码。您可以在[开发者指南](https://oreil.ly/Gi9mA)中找到当前的设置开发环境的指南。在提交任何拉取请求之前，请务必查阅最新的[贡献指南](https://oreil.ly/syVVk)。

作为参考，三个主要 Operator Framework 组件的存储库如下：

+   [*https://github.com/operator-framework/operator-sdk*](https://github.com/operator-framework/operator-sdk)

+   [*https://github.com/operator-framework/operator-lifecycle-manager*](https://github.com/operator-framework/operator-lifecycle-manager)

+   [*https://github.com/operator-framework/operator-metering*](https://github.com/operator-framework/operator-metering)

如果您不擅长编码，仍然可以通过更新和完善项目文档来进行贡献。“kind/documentation”标签用于识别突出的错误和增强请求。

# 共享 Operators

[OperatorHub.io](https://operatorhub.io)是一个社区编写的运营商托管站点。该站点包含来自各种类别的运营商，包括：

+   数据库

+   机器学习

+   监控

+   网络

+   存储

+   安全性

社区为在此站点上展示的运营商提供自动化测试和手动审核。它们已打包为必要的元数据文件，可以通过 OLM 安装和管理（详见第八章了解更多信息）。

您可以通过向[Community Operators repository](https://oreil.ly/j0rlN)提交拉取请求来将运营商提交至 OperatorHub.io。请查看[此 OperatorHub.io 页面](https://operatorhub.io/contribute)，其中包含最新的提交说明，包括打包指南。

此外，OperatorHub.io 提供了一种预览您的 CSV 文件在被接受并托管在网站上后的外观方式。这是确保您输入了正确的元数据字段的好方法。您可以在[Operator 预览页面](https://operatorhub.io/preview)了解更多信息。

[Awesome Operators 仓库](https://oreil.ly/OClO4) 维护了一个更新的运算符列表，这些运算符并未托管在 OperatorHub.io 上。虽然这些运算符未经过与托管在 OperatorHub.io 上相同的审核，但它们都是开源的，并列有其对应的 GitHub 仓库。

# 摘要

作为一个开源项目，Operator Framework 依赖社区的参与而茁壮成长。每一点贡献都很重要，从参与邮件列表的讨论到贡献代码修复错误和新增功能。同时，贡献到 OperatorHub.io 还有助于推广您的运算符，并扩展可用功能的生态系统。

^(1) 查看[这里的示例问题](https://oreil.ly/sU3rW)和[这里](https://oreil.ly/m81qp)。
