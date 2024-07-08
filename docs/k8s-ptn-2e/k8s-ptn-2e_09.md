# 第二部分：行为模式

该类别中的模式专注于 Pod 之间的通信和交互与管理平台之间的交互。根据使用的管理控制器类型，Pod 可能会一直运行直至完成，或者被调度定期运行。它可以作为守护进程运行，或确保其副本的唯一性保证。在 Kubernetes 上运行 Pod 的不同方式，选择正确的 Pod 管理原语需要理解它们的行为。在接下来的章节中，我们探讨以下模式：

+   第七章，“批处理作业”，描述了如何隔离一个原子工作单元并运行直至完成。

+   第八章，“周期性作业”，允许根据时间事件触发执行工作单元。

+   第九章，“守护进程服务”，允许您在应用程序 Pod 放置之前，在特定节点上运行以基础设施为焦点的 Pod。

+   第十章，“单例服务”，确保同一时间只有一个服务实例处于活动状态，同时保持高可用性。

+   第十一章，“无状态服务”，描述了管理相同应用程序实例所使用的基本组件。

+   第十二章，“有状态服务”，讲述了如何使用 Kubernetes 创建和管理分布式有状态应用程序。

+   第十三章，“服务发现”，解释了客户端服务如何发现和消费提供服务的实例。

+   第十四章，“自我感知”，描述了向应用程序注入自省和元数据的机制。