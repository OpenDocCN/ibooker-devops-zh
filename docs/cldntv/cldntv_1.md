# 第一章：云原生简介

什么是云原生应用程序？使它们如此吸引人的原因是什么，以至于云原生模型现在不仅被认为适用于云端，还适用于边缘？最后，你如何设计和开发云原生应用程序？这些都是本书将在全书中回答的问题。但在我们深入探讨什么、为什么和如何之前，我们想先简要介绍一下云原生世界及其一些构建现代云原生应用程序和环境基础的基本概念和假设。

# 分布式系统

开发人员在首次构建云原生应用程序时面临的最大障碍之一是，他们必须处理不在同一台机器上的服务，并且需要处理考虑到机器之间网络的模式。甚至在不知不觉中，他们已经进入了分布式系统的世界。*分布式系统*是指通过网络连接的个体计算机看起来像单个计算机的系统。能够在一群机器上分配计算能力是实现可伸缩性、可靠性和更好经济性的好方法。例如，大多数云提供商正在使用更便宜的通用硬件，并通过软件解决方案解决高可用性和可靠性等常见问题。

## 分布式系统的谬误

大多数开发人员和架构师在进入分布式系统的世界时会有一些不正确或无根据的假设。彼得·德意志（Peter Deutsch）是 Sun Microsystems 的一位研究员，在 1994 年就已经指出了分布式计算的谬误，那时还没有人考虑到云计算。因为云原生应用程序在其核心上就是分布式系统，这些谬误至今仍然有效。以下是德意志描述的谬误列表，以及它们在云原生应用程序中的含义：

网络是可靠的

即使在云中，你也不能假设网络是可靠的。因为服务通常放置在不同的机器上，你需要以一种能够应对潜在网络故障的方式开发软件，这一点我们稍后在本书中讨论。

延迟是零

延迟和带宽经常被混淆，但理解它们的区别非常重要。延迟是数据接收所需的时间，而带宽则表示在给定时间窗口内可以传输多少数据。由于延迟对用户体验和性能有很大影响，你应该注意以下几点：

+   避免频繁的网络调用和引入网络通信的冗余。

+   设计你的云原生应用程序，使数据通过使用缓存、内容传递网络（CDN）和多区域部署尽可能接近客户端。

+   使用发布/订阅（pub/sub）机制来通知有新数据并将其存储在本地以便立即可用。 第三章 更详细地涵盖了诸如 pub/sub 之类的消息模式。

存在无限带宽

如今，网络带宽似乎不再是一个大问题，但是新技术和边缘计算等领域开辟了需要更多带宽的新场景。例如，预计自动驾驶汽车每天会产生约 50TB 的数据。这种数据量要求您在设计云原生应用时考虑带宽的使用情况。领域驱动设计（DDD）和数据模式，如命令查询责任分离（CQRS），在这种对带宽需求严苛的情况下非常有用。 第四章 和 第六章 更详细地介绍了在云原生应用中处理数据的方法。

网络是安全的

对于开发人员来说，两件事经常被忽视：诊断和安全性。假设网络是安全的这一假设可能是致命的。作为开发人员或架构师，您需要把安全性作为设计的优先考虑因素；例如，采用深度防御的方法。

拓扑结构不会改变

宠物与牲畜之间的对比是一个随着容器的出现而流行起来的梗。它意味着你不再将任何机器视为已知实体（宠物），具有其自身的属性，如静态 IP 等等。相反，你将机器视为没有特殊属性的群体成员。这个概念在云原生应用中非常重要。由于云环境旨在提供弹性，可以根据资源消耗或每秒请求等条件添加和移除机器。

存在一个管理员

在传统软件开发中，一个人负责环境的安装、升级应用程序等是相当普遍的。现代云架构和 DevOps 方法改变了软件构建的方式。现代云原生应用是许多服务的组合，这些服务需要协同工作，并由不同团队开发。这使得一个人几乎不可能全面了解和理解应用程序，更不用说解决问题了。因此，您需要确保有治理机制使故障排除变得容易。在本书的整个过程中，我们向您介绍了诸如*发布管理*、*解耦*和*日志与监控*等重要概念。 第五章 提供了云原生应用常见 DevOps 实践的详细介绍。

运输成本为零

从原生云的角度看，可以从两个方面看待这个谬误。首先，传输发生在网络上，大多数云提供商并不免费。例如，大多数云提供商不收取数据入口费用，但却会收取数据出口费用。另一个看待这个谬误的方式是，将任何有效载荷转换为对象的成本不是免费的。例如，序列化和反序列化通常是非常昂贵的操作，您需要考虑这些操作的延迟，而不仅仅是网络调用的延迟。

网络是同构的

这几乎不值得列出，因为几乎每个开发者和架构师都明白，在构建应用程序时必须考虑不同的协议。

如前所述，尽管这些谬论早在很久以前就有记录，但它们仍然是一个很好的提醒，告诉人们在进入原生云时会做出的错误假设。在本书中，我们教授您考虑到分布式计算的所有谬误的模式和最佳实践。

## CAP 定理（CAP Theorem）

CAP 定理通常与分布式系统一起提到。CAP 定理指出，任何网络共享数据系统最多只能具备以下三种理想属性中的两种：

+   一致性（C）等同于保持数据的单一最新副本

+   高可用性（A）（用于更新）的数据

+   对网络分区的容错性（P）

实际情况是，您总会有网络分区（记住，“网络是可靠的”是分布式计算谬误之一）。这让您只有两个选择——您可以优化一致性，也可以优化高可用性。许多 NoSQL 数据库（如 Cassandra）优化为高可用性，而遵循 ACID 原则（原子性、一致性、隔离性和持久性）的基于 SQL 的系统则优化为一致性。

# 十二要素应用（The Twelve-Factor App）

在基础设施即服务（IaaS）和平台即服务（PaaS）早期，很快就明显地需要一种新的应用程序开发方式。例如，传统的扩展往往通过垂直扩展来实现，即向机器添加更多资源。另一方面，在云中，扩展通常是水平的，即添加更多机器以分配负载。这种类型的扩展需要无状态应用程序，这是*十二要素应用宣言*描述的因素之一。十二要素应用方法可以被认为是原生云应用程序的基础，并由 Heroku 的工程师首次引入，源自云中应用程序开发的最佳实践。自从十二要素宣言引入以来，云开发已经发展，但原则仍然适用。以下是适用于原生云应用程序的 12 个因素及其含义：

1.  代码库（Codebase）

    *一种代码库在版本控制中跟踪；多次部署*。

    每个应用程序只有一个代码库，但可以部署到多个环境，例如开发、测试和生产。在云原生架构中，这直接转化为每个服务或函数一个代码库，每个都有自己的持续集成/持续部署（CI/CD）流程。

1.  依赖关系

    *明确声明和隔离依赖关系*。

    声明和隔离依赖关系是云原生开发的重要方面。许多问题源于缺少依赖关系或依赖关系版本不匹配，这些问题来自本地和云环境之间的环境差异。通常情况下，您应始终为诸如 Maven 或 npm 等语言使用依赖管理器。容器大大减少了基于依赖关系的问题，因为所有依赖关系都打包在容器内，并且应在 Dockerfile 中声明。Chef、Puppet、Ansible 和 Terraform 是管理和安装系统依赖项的优秀工具。

1.  配置

    *将配置存储在环境中*。

    配置应严格与代码分离。这样可以轻松地根据环境应用配置。例如，您可以有一个测试配置文件，其中存储了测试环境中使用的所有连接字符串和其他信息。如果您希望将同一应用程序部署到生产环境中，只需替换配置即可。许多现代平台支持外部配置，无论是 Kubernetes 中的配置映射还是云环境中的托管配置服务。

1.  支持服务

    *将支持服务视为附加资源*。

    支持服务被定义为“应用程序在正常操作中通过网络消耗的任何服务”。在云原生应用程序的情况下，这可能是托管的缓存服务或作为服务的数据库实现。建议是通过存储在外部配置系统中的配置设置访问这些服务，这允许松耦合，这也是云原生应用程序适用的原则之一。

1.  构建、发布、运行

    *严格分离构建和运行阶段*。

    正如您将在第五章中了解到的关于 DevOps 的内容，建议采用 CI/CD 实践实现完全自动化的构建和发布阶段。

1.  进程

    *在一个或多个无状态进程中执行应用程序*。

    正如前面提到的，在云中计算应该是无状态的，这意味着数据只应保存在进程外部。这样做可以实现弹性，这也是云计算的一个承诺之一。

1.  数据隔离

    *每个服务管理其自己的数据*。

    这是<em>微服务架构</em>的关键原则之一，这是云原生应用程序中常见的模式。每个服务管理其自己的数据，只能通过 API 访问，这意味着应用程序中的其他服务不允许直接访问另一个服务的数据。

1.  并发性

    *通过进程模型扩展*。

    Improved scale and resource usage are two of the key benefits of cloud native applications, meaning that you can scale each service or function independently and horizontally; thus, you’ll achieve better resource usage.

1.  Disposability

    *通过快速启动和优雅关闭来最大化健壮性*。

    Containers and functions already satisfy this factor given that both provide fast startup times. One thing that is often neglected is to design for a crash or scale in scenario, meaning that the instance count of a function or a container is decreased, which is also captured in this factor.

1.  Dev/Prod Parity

    *保持开发、演示和生产尽可能相似*。

    Containers allow you to package all of the dependencies of your service, which limits the issues with environment inconsistencies. There are scenarios that are a bit trickier, especially when you use managed services that are not available on-premises in your Dev environment. 第五章 介绍了保持环境尽可能一致的方法和技术。

1.  Logs

    *Treat logs as event streams*.

    Logging is one of the most important tasks in a distributed system. There are so many moving parts and without a good logging strategy, you would be “flying blind” when the application is not behaving as expected. The Twelve-Factor manifesto states that you should treat logs as streams, routed to external systems.

1.  Admin Processes

    *将管理和管理任务作为一次性进程运行*。

    This basically means that you should execute administrative and management tasks as short-lived processes. Both functions and containers are great tools for that.

本书全程你会发现许多这些因素，因为它们对于云原生应用仍然非常重要。

# Availability and Service-Level Agreements

Most of the time, cloud native applications are composite applications that use compute, such as containers and functions, but also managed cloud services such as DbaaS, caching services, and/or identity services. What is not obvious is that your compound Service-Level Agreement (SLA) will never be as high as the highest availability of an individual service. SLAs are typically measured in uptime in a year, more commonly referred to as “number of nines.” 表格 1-1 显示了云服务常见的可用性百分比及其对应的停机时间。

表格 1-1\. 可用性百分比和服务停机时间

| 可用性 % | 每年停机时间 | 每月停机时间 | 每周停机时间 |
| --- | --- | --- | --- |
| 99% | 3.65 天 | 7.20 小时 | 1.68 小时 |
| 99.9% | 8.76 小时 | 43.2 分钟 | 10.1 分钟 |
| 99.99% | 52.56 minutes | 4.32 minutes | 1.01 minutes |
| 99.999% | 5.26 分钟 | 25.9 秒 | 6.05 秒 |
| 99.9999% | 31.5 seconds | 2.59 seconds | 0.605 seconds |

Following is an example of a compound SLA:

+   服务 1（99.95%）+ 服务 2（99.90%）：0.9995 × 0.9990 = 0.9985005

复合 SLA 为 99.85%。

# 总结

许多开发人员在开始为云端开发时感到困难。简而言之，开发人员面临三个主要挑战：首先，他们需要理解分布式系统；其次，他们需要了解诸如容器和函数等新技术；第三，他们需要了解在构建云原生应用程序时使用的模式。具备一些基础知识，如分布式系统的谬论、十二因素宣言和复合 SLA，将使过渡更加容易。本章介绍了云原生的一些基本概念，这使您能够更好地理解本书中讨论的一些架构考虑和模式。
