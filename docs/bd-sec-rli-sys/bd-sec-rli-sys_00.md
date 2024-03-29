# 罗亚尔·汉森的前言

> 原文：[Foreword by Royal Hansen](https://google.github.io/building-secure-and-reliable-systems/raw/foreword01.html)
> 
> 译者：[飞龙](https://github.com/wizardforcel)
> 
> 协议：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)

多年来，我一直希望有人能写一本像这样的书。自从它们出版以来，我经常钦佩并推荐谷歌的可靠性工程（SRE）书籍，所以当我来到谷歌时，我很高兴地发现一本关注安全和可靠性的书已经在进行中，我也很乐意以一种小小的方式参与其中。自从我开始在技术行业工作以来，跨越各种规模的组织，我看到人们在如何组织安全方面一直在挣扎：它应该是集中的还是联邦的？独立的还是嵌入式的？操作性的还是咨询性的？技术性的还是管理性的？等等……

当 SRE 模型和类似 SRE 的 DevOps 版本变得流行时，我注意到 SRE 所处理的问题领域表现出与安全问题类似的动态。一些组织已经将这两个学科结合起来，形成了一种称为“DevSecOps”的方法。

SRE 和安全都对经典软件工程团队有很强的依赖性。然而，它们与经典软件工程团队在根本上有所不同：

+   可靠性工程师（SRE）和安全工程师往往是破坏和修复，也是构建者。

+   他们的工作涵盖了运营，而不仅仅是开发。

+   SRE 和安全工程师是专家，而不是经典的软件工程师。

+   它们经常被视为障碍，而不是促进因素。

+   它们经常是分立的，而不是在产品团队中整合。

SRE 创建了一个特定技能集的角色和责任，我们可以将其视为安全工程师的角色。SRE 还创建了一个连接团队的实现模型，这似乎是安全社区需要采取的下一步。多年来，我和我的同事们一直主张安全应该成为软件的一流和嵌入式质量。我相信采用受 SRE 启发的方法是朝着这个方向迈出的一个合乎逻辑的步骤。

自从来到谷歌以来，我更多地了解了 SRE 模型是如何在这里建立的，SRE 如何实现 DevOps 理念，以及 SRE 和 DevOps 是如何发展的。与此同时，我一直在将我在金融服务行业的 IT 安全经验转化为谷歌的技术和项目安全能力。这两个领域并不无关，但每个领域都有其值得理解的历史。与此同时，企业正处于一个关键时刻，云计算、各种形式的机器学习以及复杂的网络安全格局共同决定着一个日益数字化的世界将走向何方，以及它将以多快的速度到达，以及涉及哪些风险。

随着我对安全和 SRE 交叉领域的理解加深，我越来越确信更彻底地将安全实践整合到软件和数据服务的整个生命周期中是非常重要的。现代混合云的性质——其中大部分基于提供互连数据和微服务的开源软件框架——使得紧密集成的安全和弹性能力变得更加重要。

在过去 20 年里，大型企业的安全运营和组织方法差异巨大。最突出的实例包括完全集中的首席信息安全官和涵盖防火墙、目录服务、代理等核心基础设施运营团队，这些团队已经发展到数百或数千名员工。在另一端，联邦业务信息安全团队拥有支持或管理一系列功能或业务运营所需的业务线或技术专业知识。在中间某处，委员会、指标和监管要求可能管理安全政策，嵌入式安全冠军可能扮演关系管理角色或跟踪指定组织单位的问题。最近，我看到团队在 SRE 模型上进行改进，将嵌入式角色演变成类似站点安全工程师的角色，或者成为专门安全团队的敏捷 Scrum 角色。

出于充分的理由，企业安全团队主要关注保密性。然而，组织通常认识到数据完整性和可用性同样重要，并通过不同的团队和不同的控制措施来解决这些问题。SRE 功能是可靠性的最佳实践方法。然而，它也在实时检测和响应技术问题方面发挥作用，包括对特权访问或敏感数据的安全相关攻击。最终，尽管工程团队在组织上根据专业技能集分开，但它们有一个共同的目标：确保系统或应用的质量和安全。

在一个每年对技术依赖性越来越强的世界中，一本关于安全和可靠性方法的书籍，从谷歌和整个行业的经验中汲取，可以对软件开发、系统管理和数据保护的演变做出重要贡献。随着威胁形势的演变，动态和综合的防御方法现在已经成为基本必需品。在我之前的角色中，我寻求对这些问题更正式的探讨；我希望安全组织内外的各种团队在方法和工具演变时会发现这次讨论有用。这个项目加强了我对它所涵盖的主题值得在行业中讨论和推广的信念，特别是在越来越多的组织采用 DevOps、DevSecOps、SRE 和混合云架构以及相关的运营模式的情况下。至少，这本书是在日益数字化的世界中系统和数据安全演变和增强的又一步。

皇家汉森，安全工程副总裁
