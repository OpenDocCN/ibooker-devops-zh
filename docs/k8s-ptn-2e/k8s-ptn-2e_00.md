# 前言

当 Craig、Joe 和我近八年前启动 Kubernetes 时，我认为我们都意识到它改变了全球软件开发和交付方式的潜力。我不认为我们知道，甚至希望相信，这种转变会来得如此迅速。Kubernetes 现在是开发跨主要公共云、私有云和裸机环境中便携可靠系统的基础。然而，即使 Kubernetes 已经变得无处不在，以至于您可以在云中不到五分钟内启动一个集群，确定创建了该集群后该何去何从仍然不那么明显。我们看到 Kubernetes 的运营化取得了如此重大的进展是非常棒的，但这只是解决方案的一部分。它是构建应用程序的基础，并提供了大量的 API 和工具库来构建这些应用程序，但它对于应用架构师或开发人员如何将这些不同的组件结合成满足其业务需求和目标的完整可靠系统提供的提示或指导甚少。

尽管通过类似系统的过去经验或试错可以获得对如何处理您的 Kubernetes 集群的必要视角和经验，但无论是时间成本还是交付给最终用户的系统质量，这都是昂贵的。当您开始在类似 Kubernetes 这样的系统之上交付关键任务服务时，通过试错学习的方式简直是太费时间，并且会导致实际的停机和中断问题。

这就是为什么 Bilgin 和 Roland 的书如此宝贵。*Kubernetes 模式*使您能够从我们编码到 Kubernetes API 和工具中的先前经验中学习。Kubernetes 是社区在各种不同环境中构建和交付许多不同可靠分布式系统经验的副产品。Kubernetes 中添加的每个对象和功能代表了为软件设计师解决特定需求而设计和定制的基础工具。本书解释了 Kubernetes 中的概念如何解决现实世界的问题，以及如何调整和使用这些概念来构建您今天正在开发的系统。

在开发 Kubernetes 时，我们始终说我们的北极星是将分布式系统开发变成 CS 101 课程。如果我们成功实现了这一目标，像这样的书籍就是这类课程的教科书。Bilgin 和 Roland 捕捉了 Kubernetes 开发人员的基本工具，并将它们分解成易于理解和消化的部分。当您完成本书时，您将意识到不仅可以在 Kubernetes 中使用哪些组件，还可以了解使用这些组件构建系统的“为什么”和“如何”。

Brendan Burns

Kubernetes 联合创始人
