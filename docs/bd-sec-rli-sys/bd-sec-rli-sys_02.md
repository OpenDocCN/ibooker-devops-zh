# 前言

> 原文：[Preface](https://google.github.io/building-secure-and-reliable-systems/raw/pr01.html)
> 
> 译者：[飞龙](https://github.com/wizardforcel)
> 
> 协议：[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)


如果一个系统在根本上不安全，它能被认为是可靠的吗？或者如果它不可靠，它能被认为是安全的吗？

成功设计、实现和维护系统需要对整个系统生命周期的承诺。只有当安全性和可靠性是系统架构的核心要素时，这种承诺才是可能的。然而，它们经常被忽视，只有在发生事故后才被考虑，导致昂贵且有时困难的改进。

[安全设计](https://oreil.ly/-46BV)在许多产品连接到互联网并且云技术变得更加普遍的世界中变得越来越重要。我们越来越依赖这些系统，它们就需要更加可靠；我们对它们的安全性的信任越大，它们就需要更加安全。

# 我们为什么写这本书

我们希望写一本书，重点是将安全性和可靠性直接融入软件和系统的生命周期，旨在突出保护系统并保持其可靠的技术和实践，并说明这些实践如何相互作用。本书的目的是提供来自专门从事安全性和可靠性的实践者的系统设计、实现和维护方面的见解。

我们要明确指出，本书推荐的一些策略需要基础设施支持，而这可能在您目前的工作地点并不存在。在可能的情况下，我们建议采用可以适应任何规模组织的方法。然而，我们认为重要的是开始讨论如何我们都可以发展和改进现有的安全性和可靠性实践，因为我们不断增长和技术娴熟的专业人士社区的所有成员都可以相互学习很多。我们希望其他组织也会急于与社区分享他们的成功和战斗经验。随着对安全性和可靠性的理念不断发展，行业可以从多样化的实现示例中受益。安全性和可靠性工程仍然是快速发展的领域。我们不断发现导致我们修改（或在某些情况下替换）先前坚定的信念的条件和案例。

# 这本书适合谁

因为安全性和可靠性是每个人的责任，我们的目标受众是广泛的：设计、实现和维护系统的人员。我们挑战传统专业角色之间的分界线，包括开发人员、架构师、[站点可靠性工程师（SRE）](https://oreil.ly/EVa7K)、系统管理员和安全工程师。虽然我们将深入探讨一些可能更适合有经验的工程师的主题，但我们邀请您——读者——在阅读各章节时尝试不同的角色，想象自己扮演您（目前）没有的角色，并思考如何改进您的系统。

我们认为每个人都应该从开发过程的最开始就考虑可靠性和安全性的基本原则，并在系统生命周期的早期阶段就将这些原则整合进去。这是本书整体的一个关键概念。在行业中有许多活跃的讨论，关于安全工程师变得更像软件开发人员，SRE 和软件开发人员变得更像安全工程师。我们邀请您加入这个讨论。

当我们在书中说“您”时，我们指的是读者，而不是特定的工作或经验水平。本书挑战了工程角色的传统期望，并旨在赋予您对整个产品生命周期的安全性和可靠性负责的能力。您不必担心在特定情况下使用本书中描述的所有实践。相反，我们鼓励您在职业生涯的不同阶段或组织的发展过程中返回本书，考虑一开始似乎不重要的想法可能会变得有意义。

# 关于文化的说明

在本书中建立和采用我们推荐的广泛最佳实践需要一种支持这种变革的文化。我们认为，您需要同时关注组织的文化和您所做的技术选择，以便专注于安全性和可靠性，以便您所做的任何调整都是持久和有弹性的。在我们看来，不重视安全性和可靠性重要性的组织需要改变，而改变组织的文化本身通常需要前期投资。

我们在整本书中融入了技术最佳实践，并用数据支持它们，但是不可能包含有数据支持的文化最佳实践。虽然本书提出了我们认为其他人可以采用或概括的方法，但每个组织都有独特的文化。我们讨论了谷歌如何在其文化中努力工作，但这可能并不直接适用于您的组织。相反，我们鼓励您从本书中包含的高层建议中提取出自己的实际应用。

# 如何阅读本书

虽然本书包含了大量的例子，但它不是一本食谱。它呈现了谷歌和行业故事，并分享了多年来我们所学到的东西。每个人的基础设施都是不同的，因此您可能需要大幅调整我们提出的一些解决方案，有些解决方案可能根本不适用于您的组织。我们试图提出高层原则和实际解决方案，以便您可以以适合您独特环境的方式实现。

我们建议您从 1 和 2 章开始阅读，然后阅读您最感兴趣的章节。大多数章节都以方框内的前言或执行摘要开头，概述以下内容：

+   问题陈述

+   在软件开发生命周期中，您应该应用这些原则和实践

+   要考虑的可靠性和安全性之间的交集和/或权衡

在每一章中，主题通常按从最基础到最复杂的顺序排列。我们还用鳄鱼图标标出深入研究和专业主题。

本书推荐了许多工具或技术，被认为是行业中的良好实践。并非每个想法都适用于您的特定用例，因此您应该评估项目的要求，并设计适应您特定风险环境的解决方案。

虽然本书旨在是自包含的，但您会发现[*Site Reliability Engineering*](https://oreil.ly/SRE-book-toc)和[*The Site Reliability Workbook*](https://oreil.ly/SRE-workbook-TOC)的参考，其中谷歌的专家描述了可靠性对服务设计的基本性质。阅读这些书可能会让您更深入地理解某些概念，但这不是必要条件。

我们希望您喜欢这本书，并且这些页面中的一些信息可以帮助您提高系统的可靠性和安全性。

# 本书使用的约定

本书使用以下排版约定：

*斜体*

指示新术语、URL、电子邮件地址、文件名和文件扩展名。

`等宽字体`

用于程序列表，以及段落内引用程序元素，如变量或函数名称，数据库，数据类型，环境变量，语句和关键字。

`**固定宽度粗体**`

显示用户应直接输入的命令或其他文本。也用于程序列表中的强调。

`*固定宽度斜体*`

显示应由用户提供值或由上下文确定的值的文本。

###### 注意

此元素表示一般说明。

###### 深入了解

此图标表示深入了解。

# O'Reilly 在线学习

###### 注意

40 多年来，[*O'Reilly Media*](http://oreilly.com)提供技术和商业培训，知识和见解，帮助公司取得成功。

我们独特的专家和创新者网络通过书籍，文章，会议和我们的在线学习平台分享他们的知识和专业知识。O'Reilly 的在线学习平台为您提供按需访问实时培训课程，深入学习路径，交互式编码环境以及来自 O'Reilly 和其他 200 多家出版商的大量文本和视频。有关更多信息，请访问[*https://oreilly.com*](https://www.oreilly.com)。

# 如何联系我们

请将有关本书的评论和问题发送至出版商：

+   O'Reilly Media，Inc.

+   1005 Gravenstein Highway North

+   塞巴斯托波尔，CA 95472

+   800-998-9938（在美国或加拿大）

+   707-829-0515（国际或本地）

+   707-829-0104（传真）

我们为这本书创建了一个网页，列出勘误，示例和任何其他信息。您可以在[*https://oreil.ly/buildSecureReliableSystems*](https://oreil.ly/buildSecureReliableSystems)上访问此页面。

发送电子邮件[*bookquestions@oreilly.com*](mailto:bookquestions@oreilly.com)以评论或询问有关本书的技术问题。

有关我们的图书，课程，会议和新闻的更多信息，请访问我们的网站[*https://www.oreilly.com*](https://www.oreilly.com)。

在 Facebook 上找到我们：[*https://facebook.com/oreilly*](https://facebook.com/oreilly)

在 Twitter 上关注我们：[*https://twitter.com/oreillymedia*](https://twitter.com/oreillymedia)

在 YouTube 上观看我们：[*https://www.youtube.com/oreillymedia*](https://www.youtube.com/oreillymedia)

# 致谢

这本书是大约 150 人的热情和慷慨贡献的成果，包括来自工程，法律和营销的作者，技术作家，章节经理和审阅人员。贡献者遍布美洲，欧洲和亚太地区的 18 个时区，以及 10 多个办公室。我们想花点时间感谢每个人，他们已经列在每章的基础上。

作为 Google 安全和 SRE 的领导者，Gordon Chaffee，Royal Hansen，Ben Lutch，Sunil Potti，Dave Rensin，Benjamin Treynor Sloss 和 Michael Wildpaner‎是 Google 内部的执行赞助商。他们对将安全性和可靠性直接整合到软件和系统生命周期的项目的信念对于使这本书成为现实至关重要。

如果没有 Ana Oprea 的努力和奉献精神，这本书将永远不会问世。她认识到这样一本书可能具有的价值，在 Google 发起了这个想法，向 SRE 和安全领导者传道，并组织了必要的大量工作。

我们要感谢那些通过提供深思熟虑的意见，讨论和审查做出贡献的人。按章节顺序，他们是：

+   第一章，*安全性和可靠性的交汇点*：Felipe Cabrera，Perry The Cynic 和 Amanda Walker

+   第二章，*了解对手*：John Asante，Shane Huntley 和 Mike Koivunen

+   第三章，*案例研究：安全代理*：Amaya Booker，Michał Czapiński，Scott Dier 和 Rainer Wolafka

+   第四章，设计权衡：Felipe Cabrera，Douglas Colish，Peter Duff，Cory Hardman，Ana Oprea 和 Sergey Simakov

+   第五章，最小权限设计：Paul Guglielmino 和 Matthew Sachs‎

+   第六章，易懂设计：Douglas Colish，Paul Guglielmino，Cory Hardman，Sergey Simakov 和 Peter Valchev

+   第七章，应对不断变化的景观设计：Adam Bacchus，Brandon Baker，Amanda Burridge，Greg Castle，Piotr Lewandowski，Mark Lodato，Dan Lorenc，Damian Menscher，Ankur Rathi，Daniel Rebolledo Samper，Michee Smith，Sampath Srinivas，Kevin Stadmeyer 和 Amanda Walker

+   第八章，弹性设计：Pierre Bourdon，Perry The Cynic，Jim Higgins，August Huber，Piotr Lewandowski，Ana Oprea，Adam Stubblefield，Seth Vargo 和 Toby Weingartner

+   第九章，恢复设计：Ana Oprea 和 JC van Winkel

+   第十章，减轻拒绝服务攻击：Zoltan Egyed，Piotr Lewandowski 和 Ana Oprea

+   第十一章，案例研究：设计、实现和维护公开可信 CA：Heather Adkins，Betsy Beyer，Ana Oprea 和 Ryan Sleevi

+   第十二章，编写代码：Douglas Colish，Felix Gröbert，Christoph Kern，Max Luebbe，Sergey Simakov 和 Peter Valchev

+   第十三章，测试代码：Douglas Colish，Daniel Fabian，Adrien Kunysz，Sergey Simakov 和 JC van Winkel‎

+   第十四章，部署代码：Brandon Baker，Max Luebbe 和 Federico Scrinzi

+   第十五章，调查系统：‎Oliver Barrett‎，Pierre Bourdon 和 Sandra Raicevic

+   第十六章，灾难规划：Heather Adkins，John Asante，Tim Craig 和 Max Luebbe

+   第十七章，危机管理：Heather Adkins，Johan Berggren，John Lunney，James Nettesheim，Aaron Peterson 和 Sara Smollet

+   第十八章，恢复和后果：Johan Berggren，Matt Linton，Michael Sinno 和 Sara Smollett

+   第十九章，Chrome 安全团队案例研究：Abhishek Arya，Will Harris，Chris Palmer，Carlos Pizano，Adrienne Porter Felt 和 Justin Schuh

+   第二十章，理解角色和责任：Angus Cameron，Daniel Fabian，Vera Haas，Royal Hansen，Jim Higgins，August Huber，Artur Janc，Michael Janosko，Mike Koivunen，Max Luebbe，Ana Oprea，Andrew Pollock，Laura Posey，Sara Smollett，Peter Valchev 和 Eduardo Vela Nava

+   第二十一章，建立安全和可靠文化：David Challoner，Artur Janc，Christoph Kern，Mike Koivunen，Kostya Serebryany 和 Dave Weinstein

我们还要特别感谢 Andrey Silin 在整本书中的指导。

以下审阅者为我们提供了宝贵的见解和反馈，指导我们前进：Heather Adkins，Kristin Berdan，Shaudy Danaye-Armstrong，Michelle Duffy，Jim Higgins，Rob Mann，Robert Morlino，Lee-Anne Mulholland，Dave O’Connor，Charles Proctor，Olivia Puerta，John Reese，Pankaj Rohatgi，Brittany Stagnaro，Adam Stubblefield，Todd Underwood 和 Mia Vu。特别感谢 JC van Winkel 进行了书籍级别的一致性审查。

我们还要感谢以下贡献者，他们提供了重要的专业知识或资源，或者对这项工作产生了一些其他卓越的影响：Ava Katushka，Kent Kawahara，Kevin Mould，Jennifer Petoff，Tom Supple，Salim Virji‎和 Merry Yen。

Eric Grosse 的外部定向审查帮助我们在新颖性和实用建议之间取得了良好的平衡。我们非常感谢他的指导，以及来自整本书的行业审阅者 Blake Bisset，David N. Blank-Edelman，Jennifer Davis 和 Kelly Shortridge 的深思熟虑的反馈。以下人员的深入审查使每一章更加针对外部受众：Kurt Andersen，Andrea Barberio，Akhil Behl，Alex Blewitt，Chris Blow，Josh Branham，Angelo Failla，Tony Godfrey，Marco Guerri，Andrew Hoffman，Steve Huff，Jennifer Janesko，Andrew Kalat，Thomas A. Limoncelli，Allan Liska，John Looney，Niall Richard Murphy，Lukasz Siudut，Jennifer Stevens，Mark van Holsteijn 和 Wietse Venema。

我们要特别感谢 Shylaja Nukala 和 Paul Blankinship，他们慷慨地投入了 SRE 和安全技术写作团队的时间和技能。

最后，我们要感谢以下贡献者，他们在这本书中没有直接出现的内容上工作：Heather Adkins，Amaya Booker，Pierre Bourdon，Alex Bramley，Angus Cameron，David Challoner，Douglas Colish，Scott Dier，Fanuel Greab，Felix Gröbert，Royal Hansen，Jim Higgins，August Huber，Kris Hunt，Artur Janc，Michael Janosko，Hunter King，Mike Koivunen，Susanne Landers，Roxana Loza，Max Luebbe，Thomas Maufer，Shylaja Nukala‎，Ana Oprea，Massimiliano Poletto，Andrew Pollock，Laura Posey，Sandra Raicevic，Fatima Rivera，Steven Roddis，Julie Saracino，David Seidman，Fermin Serna，Sergey Simakov，Sara Smollett，Johan Strumpfer，Peter Valchev，Cyrus Vesuna，Janet Vong，Jakub Warmuz，Andy Warner 和 JC van Winkel。

还要感谢 O'Reilly Media 团队——Virginia Wilson，Kristen Brown，John Devins，Colleen Lobner 和 Nikki McDonald——他们在使这本书成为现实方面提供了帮助和支持。感谢 Rachel Head 带来了一次奇妙的编辑体验！

最后，书籍核心团队也想亲自感谢以下人员：

来自 Heather Adkins

人们经常问我谷歌如何保持安全，我能给出的最简短的答案是，谷歌员工的多样化特质是谷歌自卫能力的关键。这本书反映了这种多样性，我相信在我的一生中，我不会发现比谷歌员工更伟大的互联网捍卫者。我个人特别感谢我的美妙丈夫 Will（+42!!），我的妈妈（Libby），爸爸（Mike）和哥哥（Patrick），以及 Apollo 和 Orion，因为他们插入了所有这些错别字。感谢我的团队和谷歌同事在写这本书期间容忍我的缺席，并在面对巨大对手时表现出的坚韧；感谢 Eric Grosse，Bill Coughran，Urs Hölzle，Royal Hansen，Vitaly Gudanets 和 Sergey Brin 在过去 17 年多的指导，反馈和偶尔的挑眉；感谢我的亲爱的朋友和同事（Merry，Max，Sam，Lee，Siobhan，Penny，Mark，Jess，Ben，Renee，Jak，Rich，James，Alex，Liam，Jane，Tomislav 和 Natalie），特别是 r00t++，感谢你们的鼓励。感谢 John Bernhardt 博士教会我很多；抱歉我没有完成学位！

来自 Betsy Beyer

感谢祖母，Elliott，阿姨 E 和 Joan，你们每一天都激励着我。你们是我的英雄！还有 Duzzie，Hammer，Kiki，Mini 和 Salim，你们的积极性和理智检查让我保持理智！

来自 Paul Blankinship

首先，我要感谢 Erin 和 Miller，我依赖他们的支持，还有 Matt 和 Noah，他们总是让我笑。我要对我的谷歌朋友和同事表示感激，特别是我的技术作家同行，他们在概念和语言上挣扎，需要同时成为专家和对天真用户的倡导者。我非常感激这本书的其他作者们——我钦佩并尊重你们每一个人，能够与你们的名字联系在一起是我的荣幸。

来自 Susanne Landers

对于这本书的所有贡献者，我无法表达我有多荣幸能够成为这个旅程的一部分！没有一些特别的人，我今天就不会在这里：Tom 找到了合适的机会；Cyrill 教会了我今天所知道的一切；Hannes，Michael 和 Piotr 邀请我加入了有史以来最棒的团队（Semper Tuti！）。对于那些带我喝咖啡的人（你们知道自己是谁！），没有你们生活将会无比无聊。对 Verbena，她可能比任何其他人更能塑造我，最重要的是，对我生命中最爱的人的无条件支持，以及我们最美好和美妙的孩子们。我不知道我怎么配得上你们，但我会尽我最大的努力。

来自 Piotr Lewandowski

对于每个人都让世界变得比他们发现时更美好。感谢我的家人无条件的爱。感谢我的伴侣与我分享她的生活，无论是好是坏。感谢我的朋友给我生活带来的快乐。感谢我的同事成为我工作中最好的部分。感谢我的导师持续的信任；没有他们的支持，我就不可能成为这本书的一部分。

来自 Ana Oprea

对于这本书即将印刷时将出生的小家伙。感谢我的丈夫 Fabian，他支持我并使我能够在建立家庭的同时从事这项工作和许多其他事情。我感激我的父母 Ica 和 Ion 理解我离他们很远。这个项目证明了没有开放、建设性的反馈循环就不会有进步。我之所以能够领导这本书，是因为我在过去几年中所获得的经验，这要感谢我的经理 Jan 以及整个开发者基础设施团队，他们信任我将工作重点放在安全、可靠性和开发的交叉点上。最后但同样重要的是，我要对 BSides：慕尼黑和 MUC:SEC 的支持社区表示感激，这些地方一直是我不断学习的重要场所。

来自 Adam Stubblefield

感谢我的妻子，我的家人以及多年来所有的同事和导师。

¹ 例如，参见 Dino Dai Zovi 在 Black Hat USA 2019 的[“Every Security Team Is a Software Team Now” talk](https://oreil.ly/Wap7b)，Open Security Summit 的[DevSecOps track](https://oreil.ly/_PAzE)，以及 Dave Shackleford 的[“A DevSecOps Playbook” SANS Analyst paper](https://oreil.ly/_Wmcx)。
