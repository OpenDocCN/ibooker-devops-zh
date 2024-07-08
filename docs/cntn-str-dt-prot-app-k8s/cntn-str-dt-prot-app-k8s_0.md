# 前言

本报告适用于 DevOps 工程师、平台架构师、可靠性工程团队以及对管理 Kubernetes 应用和服务存储感兴趣的应用程序所有者。接下来的页面讨论了在规模化分布式应用程序的数据存储方面面临的挑战，如何解决这些挑战，以及考虑迁移到 Kubernetes 的考虑因素。

第一章概述了在企业生产环境中采用 Kubernetes 的复杂原因，重点是传统存储如何无法满足现代云原生应用的需求。随后讨论了 Kubernetes 存储基元，包括容器存储接口以及软件定义存储如何支持 Kubernetes 工作负载的简要解释。在奠定这一基础之后，其余章节列举了企业数据需求及通过 Kubernetes 存储满足这些需求的策略。主题包括规模和效率的拓扑策略、安全性以及数据保护的挑战。

继续阅读，了解 Kubernetes 如何改变存储和数据管理格局，Kubernetes 原生存储解决方案的关键要素，以及在 Kubernetes 上运行有状态企业应用的最佳实践。

# 致谢

我要感谢 Portworx 的以下人员贡献他们的专业知识：Sarvesh Jagannivas、Bhavin Shah、Rajiv Thakkar、Ryan Wallner 和 Andy Gower。
