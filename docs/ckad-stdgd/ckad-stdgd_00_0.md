# 序言

微服务架构是当今应用开发的热门领域之一，特别是用于基于云的企业规模应用程序。使用小型、单一用途的服务构建应用程序的好处已被充分记录。但是管理可能会有大量容器化服务的任务并不容易，需要添加一个“编排器”来保持所有服务的整体协调。Kubernetes 是最流行和广泛使用的工具之一，因此能够作为应用程序开发人员使用、调试和监控 Kubernetes 的能力需求很高。为了为求职者和雇主提供一种标准手段来展示和评估在 Kubernetes 环境中开发的能力，Cloud Native Computing Foundation（CNCF）制定了[Certified Kubernetes Application Developer (CKAD)](https://www.cncf.io/training/certification/ckad/) 计划。要获得这一认证，您需要通过一项考试。

CKAD 与[Certified Kubernetes Administrator (CKA)](https://www.cncf.io/training/certification/cka/)不容混淆。虽然有一些主题重叠，但 CKA 主要专注于 Kubernetes 集群管理任务，而不是开发在集群中运行的应用程序。

在这份学习指南中，我将探讨 CKAD 考试涵盖的主题，以充分准备您通过认证考试。我们将探讨何时以及如何应用 Kubernetes 的核心概念来管理应用程序。我们还将深入研究`kubectl`命令行工具，这是 Kubernetes 工程师的主要工具。我还将提供一些技巧，帮助您更好地准备考试，并分享我个人为准备考试的全部方面的经验。

CKAD 与其他认证的典型多选题格式不同。它完全基于绩效，并要求您在巨大的时间压力下展示对手头任务的深入了解。您准备好在第一次尝试中通过考试了吗？
