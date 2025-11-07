# 第三部分\. 高级主题

在本书的第三部分，您将了解如何使用 Podman 的高级方法。这部分讨论了将 Podman 集成到您的系统中，以及 Podman 如何与其他工具和编排器协同工作。

在第七章中，我介绍了 systemd 集成。Podman 是为了完全集成到系统中而开发的，并利用了初始化系统：systemd。systemd 可以轻松地在 Podman 容器中运行，本章将向您展示如何做到这一点。同样，Podman 也可以在 systemd 服务中运行，并提供命令，允许您自动创建服务配置文件以实现这一功能。

第八章向您展示了 Podman 如何与 Kubernetes 协同工作。Podman 不是 Kubernetes 下的容器引擎，但它可以与 Kubernetes YAML 文件协同工作。因为 Kubernetes YAML 文件用于定义在 Kubernetes 中运行的应用程序，所以 Podman 使得将应用程序从完全编排的环境移动到单个节点，或者从单个节点移动到完全编排的环境变得容易。这个特性使得您更容易开发最终在 Kubernetes 下运行的应用程序，或者通过在您的笔记本电脑上本地运行这些应用程序来调试在 Kubernetes 下发生的问题。当在单个节点上运行多个容器时，Kubernetes YAML 是`docker-compose` YAML 的一个很好的替代品。

第九章介绍了 Podman 作为服务的概念，它允许编写用于使用 RESTful API 的工具生成和管理 Podman 的 pods 和容器。像`docker-compose`和其他基于 docker-py 构建的 Python 工具可以与 Podman 服务接口，从而完全不需要 Docker。Podman 服务甚至允许在远程系统（如 Windows、macOS 和 Linux）上运行的 Podman 与 Linux Podman 容器协同工作。
