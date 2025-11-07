# 前置内容

## 前言

写《Google Anthos 实战》的想法是在与数百位对在任何地方管理应用程序、更快交付软件和保护应用程序及软件供应链感兴趣的客户讨论后产生的。客户们希望更好地了解 Anthos 如何帮助他们管理在传统本地设置、边缘、云原生和多云环境中的应用程序部署。他们感兴趣的是实现容器、无服务器、基础设施即代码和服务网格的好处，以提高生产力和速度。他们希望了解如何通过自动化和透明的策略管理来保证并提高应用程序生命周期每个阶段的保障和安全。

《Google Anthos 实战》汇集了那些热衷于 Kubernetes、无服务器和 Anthos 的谷歌工程师们的集体专业知识，以及谷歌云认证高级专家，这是一群在设计和实施企业解决方案方面具有专业知识的精英级云架构师和技术领导者。

## 致谢

没有无数同行者的工作，《Google Anthos 实战》是无法实现的（[`en.wikipedia.org/wiki/Fellow_traveller`](https://en.wikipedia.org/wiki/Fellow_traveller)）。

主要作者们想要感谢其他作者们的贡献；按字母顺序排列，我们感谢 Ameer Abbas、Amita Kapoor、Aparna Sinha、Eric Brewer、Giovanni Galloro、Jarosław Gajewski、Jason Quek、Kaslin Fields、Konrad Cłapa、Kyle Bassett、Melika Golkaram、Onofrio Petragallo、Patricia Florissi、Phand Phil Taylor。其中一些作者被选入了在 2021 年 Google Cloud Next 上发布的书籍预览版。在这本完整版出版物中，所有作者都包括在这本书的 17 章以及电子书和在线的额外 6 章中。

作者们感谢所有审稿人提供的深思熟虑的意见、讨论和审阅。按字母顺序，我们感谢 Ady Degany、Alex Mattson、Alon Pildus、Amina Mansur、Amr Abdelrazik、Anil Dhawan、Ankur Jain、Anna Beremberg、Antoine Larmanjat、Ashwin Perti、Barbara Stanley、Ben Good、Bhagvan Kommadi、Brian Grant、Brian Kaufman、Chen Goldberg、Christoph Bussler、Clifford Thurber、Conor Redmond、Eric Johnson、Fabrizio Pezzella、Gabriele Di Piazza、Ganesh Swaminathan、Gil Fidel、Glen Yu、Guy Ndjeng、Harish Yakumar、Haroon Chaudhry、Hugo Figueiredo、Issy Ben-Shaul、Jamie Duncan、Jason Polites、Jeff Reed、Jeffrey Chu、Jennifer Lin、Jerome Simms、John Abel、Jonathan Donaldson、Jose San Leandro、Kamesh Ganesan、Karthikeyarajan Rajendran、Kavitha Radhakrishnan、Kevin Shatzkamer、Krzysztof Kamyczek、Laura Cellerini、Leonid Vasetsky、Louis Ryan、Luke Kupka、Maluin Patel、Manu Batra、Marco Ferrari、Marcus Johansonn、Massimo Mascaro、Maulin Patel、Micah Baker、Michael Abd-El-Malek、Michael Bright、Michelle Au、Miguel de Luna、Mike Columbus、Mike Ensor、Nima Badiey、Nina Kozinska、Norman Johnson、Purvi Desai、Quan To、Raghu Nandan、Raja Jadeja、Rambabu Posa、Rich Rose、Roman Zhuzha、Ron Avnur、Scott Penberthy、Simone Sguazza、Sri Thuraisamy、Stanley Anozie、Stephen Muss、Steren Giannini、Sudeep Batra、Tariq Islam、Tim Hockin、Tony Savor、Vanna Stano、Vinay Anand、Yoav Reich、Zach Casper 和 Zach Seils。

没有作者、审稿人、编辑和营销团队的巨大合作，这本书是不可能完成的。我们特别感谢来自谷歌的 Arun Ananthampalayam、J. P. Schaengold、Maria Bledsoe、Richard Seroter、Eyal Manor 和 Yash Kamath；以及来自 Manning 的 Doug Rudder、Aleksandar Dragosavljević和 Gloria Lukos。感谢你们的持续支持和灵感。

特别感谢 Will Grannis，谷歌云首席技术官办公室创始人兼总经理，作为一位服务型领导者，始终激励他人。此外，特别感谢埃里克·布赖尔，加州大学伯克利分校计算机科学名誉教授和谷歌基础设施副总裁。没有他的支持和鼓励，这本书是无法写成的。

所有作者的版税将捐赠给慈善机构。

### 作者

+   **阿米尔·阿巴斯**，谷歌高级产品经理，专注于现代应用程序和平台

+   **阿米塔·卡普尔**，德里大学前副教授，现在是 NePeur 的创始人，热衷于利用 AI 做好事

+   **安东尼奥·古利**，谷歌工程总监，一生致力于搜索和云服务，是三个天使的父亲

+   **阿帕尔纳·辛哈**，产品管理和 DevRel 高级总监，建立了 Kubernetes 并领导了 PM 团队，使利润和损失增长了 100 倍

+   **埃里克·布赖尔**，加州大学伯克利分校计算机科学名誉教授和谷歌基础设施副总裁

+   **乔瓦尼·加拉罗，** 谷歌客户工程师，专注于 Kubernetes、云原生工具和开发者生产力

+   **雅罗斯瓦夫·盖耶夫斯基，** 全球首席架构师和 Atos 杰出专家，谷歌云认证研究员，热衷于云、Kubernetes 和整个 CNCF 框架

+   **杰森·奎克，** Devoteam 全球 CTO，G Cloud，最初是一名程序员，现在在谷歌云上构建，热衷于 Kubernetes 和 Anthos

+   **卡斯林·菲尔德斯，** 谷歌云 GKE 和开源 Kubernetes 开发者倡导者，CNCF 大使

+   **康拉德·克拉帕，** 谷歌云认证研究员#5，负责 Atos 托管 GCP 产品设计的首席云架构师

+   **凯尔·贝塞特，** 云原生社区成员和开源倡导者，与谷歌产品和工程团队合作，领导了 Anthos 的原始设计合作伙伴关系

+   **梅丽卡·戈尔卡姆（谷歌员工），** 谷歌云解决方案架构师，专注于 Kubernetes、Anthos 和谷歌分布式边缘云

+   **迈克尔·马迪森，** World Wide Technology 的云架构师，背景在软件开发和 IaC

+   **奥诺弗里奥·佩特拉加洛（谷歌员工），** 谷歌云客户工程师，专注于数据分析和人工智能

+   **帕特里夏·弗洛里斯（谷歌员工），** 技术总监，谷歌云 CTO 办公室，过去 10 年专注于联邦计算，这是联邦分析和联邦学习的一个超集

+   **菲尔·泰勒，** CDW Digital Velocity 的 CTO，13 岁开始编程，是一位不懈的企业家，有使用公共云和 Kubernetes 将产品推向市场的记录

+   **斯科特·苏罗维奇，** HSBC 银行全球容器工程负责人，谷歌研究员，Kubernetes 倡导者，以及《Kubernetes：企业指南》的合著者

## 关于这本书

Anthos ([`cloud.google.com/anthos`](https://cloud.google.com/anthos)) 是一个多云容器化产品，在本地、多个公共云平台、私有云和边缘工作。它也是一个托管应用程序平台，将谷歌云服务和工程实践扩展到许多环境，以便您可以更快地现代化应用程序并确保它们之间的操作一致性。

### 谁应该阅读这本书？

读者应该对分布式应用程序架构有一个一般性的了解，并对云技术有一个基础的了解。他们还应该对 Kubernetes 有一个基本的了解，包括常用资源、如何创建清单以及如何使用 kubectl CLI。

本书是为任何对深化其 Anthos 和 Kubernetes 知识感兴趣的人设计的。阅读本书后，读者对 GCP 和多云平台上的 Anthos 将有更深入的了解。

### 本书是如何组织的：路线图

+   *第一章*—介绍 Anthos 和现代应用如何帮助企业推动多个行业的转型，以及云原生微服务架构如何提供可扩展性和模块化，为企业在当今世界提供基础和竞争优势。

+   *第二章*—大多数组织可以轻松管理少量集群，但随着环境的扩展，通常会遇到支持问题，使得管理变得困难。在本章中，您将了解 Anthos 如何为运行在不同云提供商和本地集群上的 Kubernetes 集群提供单一视图。

+   *第三章*—Kubernetes 正成为“数据中心 API”，是 Anthos 背后的主要组件，为我们提供所需的计算环境，以支持便携式、云原生应用，以及在适当的使用案例中，单体应用。本章介绍了 Kubernetes 的组件、声明式和命令式部署模型之间的差异以及高级调度概念，以保持工作负载在基础设施的部分部分出现故障时仍然可用。

+   *第四章*—Anthos 提供完全支持版本的 Istio，这是一个开源的服务网格，为在 Anthos 集群中运行和在外部服务器（如虚拟机）上运行的工作负载提供多个功能。了解 ASM 的组件以及每个组件如何在网格中提供功能，以及如何使用相互 TLS 来保护流量，提供高级发布周期，如 A/B 测试或金丝雀测试，并使用 GCP 控制台提供网格流量的可见性。

+   *第五章*—深入了解使用 GCP 控制台管理集群和工作负载。了解不同的日志记录和监控考虑因素，如何使用 CLI 管理集群和工作负载，以及如何在混合环境中进行扩展和设计运营管理。

+   *第六章*—利用前几章的知识，了解 Anthos 组件，这些组件为开发者提供创建应用程序的工具，包括 IntelliJ、Visual Studio Code 和 Google 的 Cloud Shell 的 Cloud Code 插件，以及使用版本控制和 Cloud Build 来部署应用程序。

+   *第七章*—Anthos 允许组织在 Kubernetes 上实现标准化，提供统一的模式来开发、部署、扩展和确保便携性和高可用性。可以使用工作负载身份来保护工作负载，这为混合和多云环境中的多个集群提供了增强的安全性。了解如何使用负载均衡器将流量路由到集群，并使用 Google 的 Traffic Director 在多个集群之间路由流量，以及如何使用 VPC 服务控制来保护您的集群。

+   *第八章*—从电信示例中了解更多关于边缘 Anthos 的信息，以及他们如何实现 5G 来增强质量检查、自动驾驶汽车和库存跟踪。

+   *第九章*—无服务器消除了 Kubernetes 对开发人员的复杂性。在本章中，您将了解基于 Knative 的 Cloud Run，以及其组件如何用于解决不同的用例，包括事件、版本管理和流量管理。

+   *第十章*—Anthos 网络具有多层和多种选项。在本章中，您将了解云网络和混合连接，包括专用互连、Cloud VPC 以及使用标准公共互联网连接。深入了解 Anthos 网络选项，了解您如何连接运行 Anthos 或任何兼容 Kubernetes 版本的集群，无论是来自其他云服务提供商还是本地。

+   *第十一章*—随着组织的发展，管理和扩展多个集群的复杂性也随之增加。Anthos Config Management (ACM)通过门卫策略提供安全性，使用 Git 等标准工具进行配置管理，并使用分层命名空间控制器提供额外的命名空间控制。

+   *第十二章*—持续集成和持续交付是成为敏捷组织的主要组成部分之一。为了实现您的 CI/CD 目标，您将学习如何使用 Skaffold、Cloud Code、Cloud Source Repositories、Artifact Registry 等工具，使您的组织真正实现敏捷。

+   *第十三章*—在 Anthos Config Management 的基础上构建，以保护您的集群免受恶意或意外事件的影响。要了解如何保护一个系统，您需要了解它可能被破坏的方式，在本章中，您将学习一个人如何部署提升的 Pod 来接管主机或整个集群。然后，使用 ACM，学习如何通过攻击或错误（如镜像中的漏洞库）来保护各种组件。

+   *第十四章*—您可以在 Anthos 上运行数百万个镜像和产品，并且您的组织可能维护其自己的产品版本。Google 使您更容易使用由 Google 或其他行业领导者（如 NetApp、IBM、Red Hat 和 Microsoft）精选的工作负载集合。在本章中，您将了解 Google Marketplace 以及如何使用它轻松地为您的用户提供解决方案。

+   *第十五章*—说服开发人员或企业从运行在虚拟服务上的遗产应用程序迁移可能很困难且耗时。他们可能没有足够的员工或主题专家来协助工作，并倾向于保持现状。Anthos 包括一个实用工具来帮助这个过程，从识别迁移候选工作负载到实际将这些工作负载从虚拟机迁移到容器。

+   *第十六章*—要将工作负载从任何遗产技术迁移到容器，您需要了解最佳方法和迁移到微服务的益处。本章将通过实际案例和要避免的反模式，教您如何使用 Anthos 通过现代化您的应用程序。

+   *第十七章*——将更高级的工作负载迁移到 Kubernetes 变得越来越普遍，包括可能需要 GPU、PCI 卡或外部硬件组件的工作负载。尽管你可以在虚拟环境中完成这项任务，但这样做有其局限性，并且存在几个复杂性。在本章中，你将学习如何在裸金属上部署 Anthos，以提供一个平台来满足你可能在 VMware 上遇到限制的需求。

以下附加附录可在本书的 ePub 和 Kindle 版本中找到，你可以在 liveBook 上在线阅读：

+   *附录 A 云是新的计算堆栈*

菲尔·泰勒

+   *附录 B 来自现场的经验教训*

凯尔·巴塞特

+   *附录 C 在 VMware 上运行的计算环境*

约阿希姆·加耶夫斯基

+   *附录 D 数据和分析*

帕特里夏·弗洛里斯

+   *附录 E ML 应用的端到端示例*

阿米塔·卡波尔

+   *附录 F 在 Windows 上运行的计算环境*

卡斯林·菲尔德斯

### liveBook 讨论论坛

购买 *Google Anthos in Action* 包括免费访问 liveBook，曼宁的在线阅读平台。使用 liveBook 的独特讨论功能，你可以在全球范围内或针对特定章节或段落附加评论。为自己做笔记、提出和回答技术问题，以及从作者和其他用户那里获得帮助都非常简单。要访问论坛，请访问 [`livebook.manning.com/book/google-anthos-in-action/discussion`](https://livebook.manning.com/book/google-anthos-in-action/discussion)。你还可以在 [`livebook.manning.com/discussion`](https://livebook.manning.com/discussion) 上了解更多关于曼宁论坛和行为准则的信息。

曼宁对读者的承诺是提供一个场所，让读者之间以及读者与作者之间可以进行有意义的对话。这不是对作者参与特定数量活动的承诺，作者对论坛的贡献仍然是自愿的（且未付费）。我们建议你尝试向他们提出一些挑战性的问题，以免他们的兴趣偏离！只要本书有售，论坛和以前讨论的存档将可通过出版社的网站访问。

## 关于主要作者

安东尼奥·古利对建立和管理全球技术人才以推动创新和执行充满热情。他的核心专长在于云计算、深度学习和搜索引擎。目前，他担任谷歌云首席技术官办公室的工程总监。此前，他曾担任谷歌华沙站点负责人，将工程站点的规模扩大了一倍。

到目前为止，Antonio 在欧洲四个国家获得了专业经验，并在欧洲、中东、亚洲和美国六个国家管理过团队；在阿姆斯特丹，作为 Elsevier（一家领先的科学出版商）的副总裁；在伦敦，作为微软 Bing 的工程站点负责人；在意大利和英国担任 CTO；在欧洲和英国担任 Ask.com 的 CTO；以及在几个共同创立的初创公司中，包括欧洲最早的网页搜索公司之一。

Antonio 共同发明了搜索、智能能源和 AI 领域的几项技术，拥有 20 多项已授权/申请的专利，他还撰写了关于编码和机器学习的几本书，这些书也被翻译成了日语、俄语、韩语和中国语。Antonio 会说西班牙语、英语和意大利语，目前正在学习波兰语和法语。Antonio 是两个男孩 Lorenzo（22 岁）和 Leonardo（17 岁）以及一个小公主 Aurora（13 岁）的骄傲的父亲，他们都有发明创造的激情。

Scott Surovich 在过去 20 年里一直是世界上最大银行之一 HSBC 的工程师。在那里，他担任过各种工程角色，包括与思杰、Windows、Linux 和虚拟化合作。在过去三年中，他一直是混合集成平台团队的负责人和 Kubernetes/Anthos 的产品负责人。

Scott 一直热衷于为任何愿意学习的人培训和撰写关于技术的文章。他多年来一直是一名认证培训师，为包括微软、思杰和 CompTIA 在内的多个供应商教授认证课程。2019 年，他共同撰写的第一本书《Kubernetes 和 Docker：企业指南》出版。该书受到了好评，在第一版成功之后，更新后的第二版于 2021 年 12 月 19 日发布，并在发布的第一周成为了畅销书。

他也是一个狂热的 3D 打印爱好者（几乎到了上瘾的地步），微控制器爱好者，以及热衷的曲棍球运动员。当 Scott 有空闲时间时，他更喜欢和妻子 Kim 以及他的狗 Belle 一起度过。

Scott 还想要感谢谷歌给他加入初始 Google Fellow 试点小组的机会，并信任他参与这本书的创建。

Michael Madison 喜欢探索新的云计算技术，并寻找利用计算进步来简化公司运营和为客户创造新价值的方法。他目前在 World Wide Technology 担任云平台架构师，这使他能够帮助公司和组织开始或继续他们的云之旅。

尽管迈克尔作为一名 IT 专业人士已有超过 15 年的经验，但他最初是在娱乐行业起步，为主题公园和邮轮公司工作。最终，他对编程的爱好变成了他的主要职业，并将他的领域扩展到基础设施和云计算。当机会来临时，他全身心投入到云计算项目中，将他在软件开发方面的十年经验应用于云计算和混合部署的挑战中。

迈克尔原籍德克萨斯州，他在乔治亚州、阿拉斯加和德克萨斯州生活和学习。他最终在密苏里州找到了工作，目前住在圣路易斯郊外。迈克尔和他的妻子拥有一辆房车，并计划在几年后带着他们的狗 Shenzi 环游全国。

## 关于封面插图

《Google Anthos in Action》封面上的插图标题为“Frascati 居民”，或“Frascati 居民”，摘自雅克·格拉塞·德·圣索沃尔（Jacques Grasset de Saint-Sauveur）于 1797 年出版的作品集。每一幅插图都是手工精心绘制和着色的。

在那些日子里，人们通过他们的服饰就能轻易地识别出他们居住的地方以及他们的职业或社会地位。曼宁通过基于几个世纪前丰富多样的地域文化的书封面来庆祝计算机行业的创新精神和主动性，这些文化通过如这一系列图片的图片被重新带回生活。
