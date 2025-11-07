# 9 探索 FlixTube

本章涵盖

+   回顾你已学到的工具

+   理解 FlixTube 的布局、结构和主要代码路径

+   在开发环境中构建、运行和测试 FlixTube

+   使用 Terraform 将 FlixTube 部署到生产环境

+   构建 FlixTube 的持续交付管道

通过第九章的旅程是一条漫长的道路。在这个过程中，我们使用了众多工具来构建微服务，测试它们并将它们部署到生产环境中。在本章中，我们将看到我们辛勤工作的成果，即 FlixTube 示例应用的完整版本。

通过本章，我们将了解 FlixTube 作为一个整体是如何工作的，并遇到一些新的微服务。我们将复习和巩固我们的技能，并在一个完整但相对简单的微服务应用中展示这些技能。

我们将首先在开发环境中构建和运行 FlixTube。接下来，我们将运行第八章中的测试。最终，我们将 FlixTube 部署到我们的生产 Kubernetes 集群，并为其创建一个持续交付（CD）管道。

## 9.1 没有新工具！

恭喜！你已经学会了所有开始构建微服务应用所需的主要工具。当然，还有更深层次的知识需要掌握。还有许多其他有用的工具你可以学习，未来也会有新的工具出现。

但是，就本书的目的而言，我们已经学到了构建基于微服务产品的最小工具集。随着你深入到持续的开发中，你会发现一些特定于你项目的难题，你需要更深入地研究这些工具。你需要学习更深入的 Docker、Kubernetes 和 Terraform 知识。然而，目前我们工具箱中的工具已经足够我们完成 FlixTube 的第一个版本。所以，让我们开始吧。

## 9.2 获取代码

要跟随本章，你需要下载代码或克隆仓库。

+   从这里下载代码的 zip 文件：

    [`github.com/bootstrapping-microservices/chapter-9`](https://github.com/bootstrapping-microservices/chapter-9)

+   你可以使用 Git 克隆代码，如下所示：

    `git clone https://github.com/bootstrapping-microservices/chapter-9.git`

如需安装和使用 Git 的帮助，请参阅第二章。如果你在代码上遇到问题，请在 GitHub 的仓库中记录一个问题。

## 9.3 回顾基本技能

在我们完成整个 FlixTube 示例的过程中，我们将练习我们学到的构建、运行、测试和部署微服务的基本技能。当你看到这样的列表时，你会意识到我们已经覆盖了多少内容！

+   使用 Node.js 运行微服务（来自第二章）

+   使用 Docker 打包和发布我们的微服务（来自第三章和第六章）

+   使用 Docker Compose 在开发环境中构建和运行我们的应用（来自第四章和第五章）

+   使用数据库存储和检索数据（来自第四章）

+   使用外部文件存储存储和检索文件（来自第四章）

+   使用 HTTP 请求和 RabbitMQ 消息在微服务之间进行通信（来自第五章）

+   使用 Jest 测试单个微服务（来自第八章）

+   使用 Cypress 测试整个应用程序（来自第八章）

+   使用 Terraform 将应用程序部署到 Kubernetes 集群（来自第六章和第七章）

+   使用 Bitbucket Pipelines 创建 CD 管道（第七章）

图 9.1 展示了我们将回顾的技能及其在事物方案中的上下文。为了充分利用本章内容，请跟随示例。您应该自己运行 FlixTube，以便研究它并理解其工作原理。为了测试和加深您的理解，您应该尝试进行自己的更改。实践是巩固这些技能的最佳方式。

![](img/CH09_F01_Davis4.png)

图 9.1 本章回顾的基本技能

## 9.4 FlixTube 概述

本章的代码仅包含一个示例：完整的 FlixTube 项目。您可以在第九章代码仓库的 example-1 子目录中找到它。让我们先从其结构的一个鸟瞰图开始。图 9.2 展示了 FlixTube 的最新版本。

![](img/CH09_F02_Davis4.png)

图 9.2 完成的 FlixTube 示例应用程序概述

### 9.4.1 FlixTube 微服务

您已经了解图 9.2 中显示的一些微服务。例如

+   视频流（首次在第二章中遇到）

+   视频存储（来自第四章）

+   历史（来自第五章）

+   元数据（来自第八章）

还有一些您尚未见过的新的微服务：网关和视频上传。表 9.1 列出了这些微服务的每个目的。

表 9.1 FlixTube 微服务

| 微服务 | 目的 |
| --- | --- |
| 网关 | 应用程序的入口点。服务于前端并提供 REST API。 |
| 视频流 | 从存储流式传输视频供用户观看。 |
| 历史 | 记录用户的观看历史。 |
| 元数据 | 记录每个视频的详细信息及其元数据。 |
| 视频上传 | 协调视频上传到存储。 |
| 视频存储 | 负责从外部云存储中存储和检索视频。 |

### 9.4.2 微服务项目结构

在我们查看整个应用程序的项目结构之前，让我们首先回顾单个 Node.js 微服务的结构。打开第九章代码仓库中 example-1 目录下的元数据目录，以跟随示例。

以元数据微服务为例，图 9.3 描述了其项目的布局。这是一个典型的 Node.js 项目，FlixTube 的所有微服务几乎都有这种相同的结构。

![](img/CH09_F03_Davis4.png)

图 9.3 Node.js 微服务项目结构（元数据微服务）

### 9.4.3 FlixTube 项目结构

现在，让我们看看整个 FlixTube 项目的结构。图 9.4 展示了其布局，包括每个微服务的子目录。打开第九章代码仓库中的 example-1 目录，亲自查看一下。

为了简单起见，FlixTube 是在一个代码仓库中构建的。使用单个仓库是学习微服务开发的绝佳方式（它使你的任务变得更简单），而且一般来说，即使你成为微服务的专家，使用单个仓库也是启动新的微服务应用程序的一种简单方便的方式。当然，使用单个仓库也是我与你分享示例代码的一种方便方式。

说了这么多，生产环境中的微服务通常永远不会包含在单个代码仓库中。使用单个仓库会消除使用微服务的最大优势：它们可以独立部署。将所有微服务放在单个仓库中意味着它们将一起部署（除非你有非常聪明的 CD 流水线）。

![](img/CH09_F04_Davis4.png)

图 9.4 整个 FlixTube 项目的结构

在现实情况下，微服务几乎总是被分割成独立的仓库，通常每个微服务都有一个独立的仓库。为了简单和方便起见，我们暂时将 FlixTube 放在一个仓库中。在第十一章中，我们将讨论单仓库与多仓库的对比，以及如何通过将 FlixTube 分割到独立的代码仓库中来推进它。

## 9.5 在开发中运行 FlixTube

我们的第一步是将 FlixTube 运行在我们的开发工作站（或个人电脑）上。图 9.5 展示了它在开发中的样子。请注意，我们已经用它的模拟版本替换了视频存储微服务。我们很快会讨论为什么以及如何这样做。

![](img/CH09_F05_Davis4.png)

图 9.5 FlixTube 在开发中的样子

### 9.5.1 启动微服务

在启动整个应用程序之前，回顾一下我们如何启动单个微服务是有意义的。当开发新的微服务或专注于现有微服务时，我们经常需要在该微服务之外的应用程序上下文中单独运行该微服务。

我们使用 Node.js 作为我们的微服务，这意味着我们的微服务将直接在我们的开发工作站上的 Node.js 下运行。如果你在第二章和第八章中一直跟随，那么你已经有 Node.js 安装了。如果没有，请回到第二章 2.5.4 节获取说明。在运行 Node.js 项目之前，你必须首先安装依赖项，如下所示：

```
npm install
```

运行 Node.js 项目时，请使用 npm start 脚本约定：

```
npm start
```

这将调用在项目的 package.json 文件中指定的命令行。FlixTube 中的所有微服务都遵循这个常见的 Node.js 约定。这意味着你知道如何启动 FlixTube 中的任何微服务，以独立运行在 *生产模式* 下。

在持续开发期间，更合适的是以*开发模式*运行微服务。这启用了*实时重新加载*（首次在第 2.6.8 节中介绍），这样我们就可以编辑我们的代码，并且微服务会自动重新启动以包含更改。我们使用 start:dev 脚本来以开发模式运行 FlixTube 的任何微服务（我的个人约定）：

```
npm run start:dev
```

(您可以在生产模式、开发模式和实时重新加载中进行进一步修改。要了解更多信息，请返回第二章的 2.6.7 和 2.6.8 节。)

您可能已经注意到，FlixTube 的大多数微服务现在都有依赖项，这使得它们单独启动变得更加困难。其中大部分需要数据库或 RabbitMQ 服务器。其中一些需要两者。我们可以通过以下任何一种方式来处理这个问题：

+   *在您的开发工作站上安装 MongoDB 和 RabbitMQ。* 这在短期内可能很烦人，但从长远来看非常有用。

+   *使用 Docker 或 Docker Compose 实例化 MongoDB 和 RabbitMQ 服务器容器。* 这是一种方便、有效且简单的方法。

+   *模拟 MongoDB、RabbitMQ 和其他依赖项的库。* 这与我们在第八章中做的是类似的。您可能想为您的自动化测试做这件事。

### 9.5.2 启动应用程序

现在，让我们使用 Docker Compose 启动整个 FlixTube 应用程序，这是我们首次在第四章中遇到的有用工具，并且自那时起一直在使用。在日常产品开发中，我们经常构建和重新启动我们的应用程序，而 Docker Compose 使这变得更加简单。我们经常抽出时间专注于单个微服务，但我们在微服务组件演变的同时，仍然经常想要测试我们的更大应用程序。

如果您跟随了第四章、第五章和第八章的内容，您已经安装了 Docker Compose。如果没有，请参考第 4.3.2 节并安装它，以便您可以继续操作。现在您应该打开 VS Code 中的第九章的 example-1 子目录，以查看代码。

列表 9.1 让我们回忆起 Docker Compose 文件（docker-compose.yaml）的样子。FlixTube 版本的此文件是本书中最大的，因此列表 9.1 为了简洁而进行了缩写。此文件中的大多数条目都是相似的，因此可以安全地省略。如果您看过一个，基本上就看过它们了。

列表 9.1 启动 FlixTube 开发环境的 Docker Compose 文件（从第九章的 chapter-9/example-1/docker-compose.yaml 缩写）

```
version: '3'
services:
  db:                                    ①
    image: mongo:4.2.0
    container_name: db
    # ... code omitted for brevity ...

  rabbit:                               ②
    image: rabbitmq:3.8.1-management
    container_name: rabbit
    # ... code omitted ...
  db-fixture-rest-api:                  ③
    image: db-fixture-rest-api
    build: 
      context: ./db-fixture-rest-api
      dockerfile: Dockerfile
    container_name: db-fixture-rest-api
    # ... code omitted ...
  video-streaming:                      ④
    image: video-streaming
    build: 
      context: ./video-streaming
      dockerfile: Dockerfile-dev
    container_name: video-streaming
    # ... code omitted ...
  # ... other microservices omitted     ⑤
```

① 启动 MongoDB 数据库的容器

② 启动 RabbitMQ 服务器的容器

③ 启动用于加载数据库固定数据的 REST API

④ 构建并启动视频流微服务

⑤ 其他所有 FlixTube 微服务都在这里。

大多数 FlixTube 微服务在列表 9.1 中被省略了，但你可以看到的是我们熟悉的老朋友，视频流微服务。还包括我们数据库的设置（在第四章中介绍），RabbitMQ（在第五章中介绍），以及我们将用于自动化测试的数据库固定 REST API（在第八章中介绍）。现在使用 Docker Compose 构建并启动 FlixTube：

```
cd example-1
docker-compose up --build
```

构建和启动需要一些时间，尤其是如果你之前没有这样做过。Docker 需要下载和缓存基础镜像。

现在，随着 FlixTube 应用程序的运行，打开你的浏览器并导航到 http://localhost:4000 以查看 FlixTube 的主页。你会发现 FlixTube 有一个闪亮的新用户界面（UI）！我们很快会更多地讨论这一点。现在，花些时间探索 FlixTube 的 UI：

1.  导航到上传页面。

1.  上传视频。

1.  导航回主页以查看列表中的上传视频。

1.  点击视频播放它。

当你完成开发后，别忘了关闭 FlixTube，以免它在你的开发工作站上继续消耗资源。你可以通过在 Docker Compose 运行的终端中按 Ctrl-C 来做到这一点，然后调用

```
docker-compose down
```

## 9.6 开发中测试 FlixTube

测试对于开发实践至关重要。我们可以也应该进行手动测试，但自动化测试在效率、可靠性和可重复性方面无与伦比。

在第八章中，我们探讨了使用 Jest 和 Cypress 的多种测试方法。我们在这里将再次回顾这些方法。我们在那一章中查看的各种测试在本章 9 的代码库中重复出现。我们现在将对完成的 FlixTube 示例运行这些测试。

当然，任何真正的应用程序都将比我们这里运行的几个测试有更多的测试。这只是一个演示，我并没有追求接近完整的测试覆盖率。在接下来的章节中，尝试自己运行这些测试。

### 9.6.1 使用 Jest 测试微服务

FlixTube 中的元数据微服务包括第八章中的 Jest 单元测试。在运行测试之前，你需要安装依赖项：

```
cd chapter-9/example-1/metadata
npm install
```

现在按照以下标准 npm `test`脚本来运行测试：

```
npm test
```

这将在第八章中配置的元数据微服务的 package.json 文件中执行相关的命令行。图 9.6 显示了成功测试运行的结果。

![](img/CH09_F06_Davis4.png)

图 9.6 使用 Jest 对元数据微服务进行自动测试的成功运行

你也可以在*实时重新加载*模式下运行测试，这意味着你可以编辑你的代码，测试将自动重新启动。我们使用另一个名为`test:watch`的 npm 脚本来做到这一点（我的个人约定）：

```
npm run test:watch
```

要更详细地复习 Jest，请回到第 8.5 节。要回顾 npm 和实时重新加载的 Jest 设置，请参阅第 8.5.8 节。

### 9.6.2 使用 Cypress 测试应用程序

我们也可以从第八章开始运行 Cypress 端到端测试，针对 FlixTube 应用程序。在第八章中，我们针对 FlixTube 的简化版本运行了这个测试。然而，这次我们将针对完整的应用程序运行它。要运行这个测试，你需要为 FlixTube 项目安装依赖项：

```
cd chapter-9/example-1
npm install
```

如果您还没有这样做，请确保实际启动应用程序：

```
docker-compose up --build
```

现在，运行常规的 npm test 脚本，在这个例子中，它被配置为调用 Cypress：

```
npm test
```

这将在终端中以无头模式运行 Cypress。在开发过程中，我们希望启动 Cypress UI，如图 9.7 所示。在这种情况下，我们将使用我们配置的 `test:watch` 脚本来启动 Cypress UI：

```
npm run test:watch
```

![](img/CH09_F07_Davis4.png)

图 9.7 使用 Cypress 对 FlixTube UI 的自动化测试成功运行

在 Cypress UI 运行的情况下，我们可以对影响我们前端代码的代码进行更改，并以非常直观的方式看到结果。要更详细地修订 Cypress，请返回到第 8.8 节。要回顾 Cypress 的 npm 设置，请转到第 8.8.7 节。

## 9.7 深入了解 FlixTube

到目前为止，你应该从高层次上理解了 FlixTube。你知道微服务以及每个微服务的目的。你知道如何在你的开发工作站上构建、运行和测试应用程序。在我们将 FlixTube 部署到生产之前，让我们首先了解一些其更深入细节。在本节中，我们将查看 FlixTube 的各个方面：

+   数据库固定值

+   模拟存储微服务

+   网关

+   FlixTube UI

+   视频流

+   视频上传

### 9.7.1 数据库固定值

我们在第八章中首先讨论了数据库固定值，在那里我们使用这些固定值在运行自动化测试之前将真实数据集加载到数据库中。我们看到了数据库固定值在自动化测试中的应用，但它们对手动测试甚至产品演示也非常有用。能够启动应用程序并展示包含真实数据的完整应用程序是非常有用的！

当使用 Jest 进行单元测试时，我们不需要任何数据，因为我们模拟了 MongoDB 数据库库，并能够通过数据库库的模拟版本提供的假数据替换真实数据。当使用 Jest 进行集成测试时，我们能够在测试代码中直接使用 MongoDB 库与我们的 MongoDB 数据库进行交互。这意味着我们可以在测试代码中内联测试数据，但不必为它创建单独的数据文件。

当使用 Cypress 进行端到端测试时，我们必须找到不同的解决方案。因为 Cypress 测试在浏览器中运行（Cypress 是基于 Electron 构建的，而 Electron 是基于 Chrome 的），所以我们无法访问 MongoDB 库（它只在 Node.js 下运行）。在这种情况下，我们无法直接操作我们的 MongoDB 数据库。

为了解决这个问题，我创建了 *数据库固定配置* *REST API*。这是一个看起来与其他你在本书中看到的任何其他微服务相似的 REST API。我们不会直接查看其代码，但如果你想自己查看，你会发现它已经相当熟悉了。REST API 的代码包含在第八章代码存储库中，并复制到第九章代码存储库，以便我们在对 FlixTube 运行测试时可以使用它。此外，你可以在 GitHub 上找到它的原始源代码：[`github.com/ashleydavis/db-fixture-rest-api`](https://github.com/ashleydavis/db-fixture-rest-api)。你可以在列表 9.1 中找到 REST API 容器的设置。

要了解数据库固定配置的外观，请参阅列表 9.2。一般来说，我们的数据库固定配置存储在 chapter-9/example-1 的 fixtures 子目录下。

FlixTube 只有一个数据库固定配置文件在 videos.js 文件中（如列表 9.2 所示）。文件名表示数据将要存储的数据库集合。从这个固定配置文件中获取的数据将被加载到视频集合中。

包含该文件的目录表示固定配置的名称。在这种情况下，目录的名称是 two-videos，因此数据库固定配置的名称是 two-videos。我之所以给固定配置起这个名字，是因为它的目的是将两个视频的元数据加载到我们的数据库中。一般来说，我们应该给我们的数据库固定配置起有意义的名字，这样我们就可以轻松记住它们的目的。

每个数据库固定配置可以由多个文件组成。尽管在这里我们只为我们的双视频固定配置文件只有一个文件，但它可以有更多这样的文件来设置数据库中其他集合的内容。

列表 9.2 FlixTube 的一个示例数据库固定配置（chapter-9/example-1/fixtures/two-videos/videos.js）

```
const mongodb = require("mongodb");                       ①
module.exports = [                                        ②
    {
        _id: 
        ➥ mongodb.ObjectId("5ea234a1c34230004592eb32"),  ③
        name: "SampleVideo_1280x720_1mb.mp4"              ④
    },
    {
        _id: 
        ➥ mongodb.ObjectId("5ea234a5c34230004592eb33"),  ③
        name: "Another video.mp4"                         ④
    }
];
```

① 导入 MongoDB 库以便我们可以创建数据库 ID

② 将插入到元数据库视频集合中的数据导出

③ 为新记录创建数据库 ID

④ 设置视频的文件名

如果你之前在 9.6.2 节中运行了 Cypress 测试，那么你已经使用了这个数据库固定配置！请注意，列表 9.2 中显示的固定配置实际上是一个 JavaScript 文件。我们可以使用 JSON 格式或 JavaScript 来创建这些数据库固定配置。JSON 适用于静态数据，但 JavaScript 是生成动态数据的一个很好的选择。这为我们生成测试数据提供了很大的灵活性。在列表 9.2 中，看看我们如何使用 MongoDB 库为我们的测试数据生成数据库 ID。

### 9.7.2 模拟存储

为了开发期间的方便，我们将视频存储微服务的 Azure 版本替换为模拟版本。这与我们在第 8.5.10 节中使用的模拟类似。不同之处在于，我们不是用模拟版本替换函数、对象和库，而是现在用一个假版本替换整个微服务。图 9.8 显示了当 Azure 存储被模拟存储微服务替换时 FlixTube 的样子。

我们的模拟存储微服务并不是完全的虚构！它仍然执行存储的任务，但不是使用云存储，而是在本地文件系统中存储视频。我们这样做的主要原因不仅仅是测试；它还为了方便和性能，能够将整个应用程序限制在我们的开发工作站上。

![图片](img/CH09_F08_Davis4.png)

图 9.8 在开发期间用模拟微服务替换云存储，以实现更方便和高效的使用

在开发期间运行时，我们更喜欢消除外部依赖，如连接到云存储。在这种情况下，将存储限制在本地文件系统使开发设置更加简单。性能得到提升，因为视频是本地存储的，而不是发送到云端。除了这个变化之外，FlixTube 仍然按正常工作，其他微服务并不知道 Azure 存储微服务已被踢出并替换为模拟版本。

能够用更简单的模拟版本替换复杂微服务不仅方便，而且在未来的某个时刻可能也是必要的。目前，FlixTube 是一个小型应用程序，但你可以想象，随着它成长为注定要成为世界主导的流媒体服务，它将变得太大，无法在单台计算机上运行。

到那时，我们需要使出浑身解数来使其适应。这包括删除我们不需要的微服务；例如，如果你不需要测试它，你可以从 Docker Compose 文件中移除历史微服务。

注意：移除或替换大型复杂微服务——甚至可能是整个微服务组——是减少我们应用程序大小的重要技术，以便它可以在单台计算机上运行并在开发期间运行。

列表 9.3 展示了 FlixTube 的 Docker Compose 文件中我们的模拟存储微服务的配置。它看起来与 Azure 存储微服务的配置相似。不同的一点是，存储子目录是在主机操作系统和容器之间共享的。这是上传视频存储的目录。以这种方式共享意味着我们可以在主机操作系统上自行检查上传的视频，以测试微服务是否正常运行。

列表 9.3 Docker Compose 文件中的模拟存储微服务设置（摘自第九章/示例-1/docker-compose.yaml）

```
  video-storage:                        ①
    image: mock-storage
    build: 
      context: ./mock-storage           ②
      dockerfile: Dockerfile-dev
    container_name: video-storage
    volumes:
      - /tmp/mock-storage/npm-cache:/root/.npm:z
      - ./mock-storage/src:/usr/src/app/src:z
      - ./mock-storage/storage:
      ➥ /usr/src/app/storage:z         ③
    ports:
     - "4005:80"
    environment:
      - PORT=80
    restart: "no"
```

① 将 DNS 名称设置为 video-storage。（其他微服务不知道 Azure 存储微服务已被模拟版本替换。）

② 我们不是从 azure-storage 子目录构建容器，而是从 mock-storage 子目录构建模拟版本。

③ 在主机操作系统和容器之间共享存储目录，并将视频存储在此目录中。您可以从主机检查它们，以确保模拟存储微服务正常工作。

能够用模拟替换微服务是开发中的一个很好的选项。它可以帮助简化开发，但有时我们需要关注微服务的实际版本；我们需要测试它而不是模拟版本。在这些时候，我们可以在 Docker Compose 文件中简单地用实际版本替换模拟版本。如果您愿意，您可以亲自尝试一下。

列表 9.4 显示了实际存储微服务的已注释配置。只需取消注释此配置，然后注释掉模拟版本的配置。现在重新构建并重启您的应用程序。您现在可以在开发中测试实际存储微服务了！

列表 9.4 实际存储微服务已注释（摘自第九章示例 1 的 docker-compose.yaml）

```
  # video-storage:                     ①
  #   image: azure-storage             ① 
  #   build:                           ①
  #     context: ./azure-storage       ①
  #     dockerfile: Dockerfile-dev     ①
  #   container_name: video-storage    ①
  #   ... code omitted for brevity ... ①
```

① 取消注释此行以在开发期间将 Azure 存储微服务包含在应用程序中。为了使此操作生效，您必须先注释掉模拟存储微服务（如列表 9.3 所示），实际上用实际版本替换它。

列表 9.5 显示了模拟存储微服务的代码。模拟版本用使用本地文件系统的版本替换了实际存储微服务的 /video 和 /upload 路由。模拟微服务是一个即插即用的替代品，因为它的 REST API 符合实际微服务的接口。

列表 9.5 模拟存储微服务（摘自第九章示例 1 的 mock-storage/src/index.js）

```
const express = require("express");
const fs = require("fs");
const path = require("path");
const app = express();
const storagePath = 
➥ path.join(__dirname, "../storage");                      ①
app.get("/video", (req, res) => {                           ②
    const videoId = req.query.id;
    const localFilePath = path.join(storagePath, videoId);
    res.sendFile(localFilePath);                            ③
});
app.post("/upload", (req, res) => {                         ④
    const videoId = req.headers.id;
    const localFilePath = path.join(storagePath, videoId);
    const fileWriteStream = 
    ➥ fs.createWriteStream(localFilePath);
    req.pipe(fileWriteStream)                               ⑤
        .on("error", err => {
            console.error("Upload failed.");
            console.error(err && err.stack || err);
        })
        .on("finish", () => {
            res.sendStatus(200);
        });                                                 ⑤
});
const port = process.env.PORT && parseInt(process.env.PORT) || 3000;
app.listen(port, () => {
    console.log(`Microservice online`);
});
```

① 设置在本地文件系统中存储视频的路径

② HTTP GET 路由处理程序，从存储中流式传输视频

③ 将本地文件直接作为响应发送给 HTTP 请求

④ HTTP POST 路由处理程序，将视频上传到存储

⑤ 将传入的 HTTP 请求体（上传的文件）流式传输到本地文件

### 9.7.3 网关

FlixTube 有一个单独的网关微服务。它被称为 *网关*，因为它充当用户进入应用程序的网关。对于 FlixTube 的当前版本，这是整个应用程序的唯一入口点。网关提供了前端 UI，使用户能够通过他们的网络浏览器与 FlixTube 交互。它还提供了一个 REST API，以便前端可以与后端交互。

FlixTube 目前还不支持任何类型的身份验证，但未来我们可能会升级网关以验证我们的用户。FlixTube 用户必须在网关允许他们与后端交互之前进行登录。

图 9.9 展示了 FlixTube 多个网关的潜在未来。这说明了被称为“前端的后端”的已知模式。每个前端都有自己的网关。有一个网关供网络浏览器访问；另一个网关供移动应用访问；还有一个网关供 FlixTube 管理门户使用。

![图片](img/CH09_F09_Davis4.png)

图 9.9 多个网关的 FlixTube 将看起来如何

如果可能的话，我们希望保持简单，并只支持单个网关。在多个类型的客户端之间共享网关是完全可行的。但如果我们发现前端有不同的需求（例如，网络和移动之间的不同认证形式或网络和管理门户之间的不同安全考虑），那么“前端的后端”模式可以提供帮助。

如果我们扩展到拥有多个网关，那么我们就会希望使用单独的主机名或子域名来访问它们。例如，浏览器的主要网关可以使用 flixtube.com，移动网关使用 mobile.flixtube.com，而管理门户使用 admin.flixtube.com。为了将域名分配给您的应用程序，您需要使用 DNS 提供商购买域名，并将每个域名配置为指向特定网关微服务的 IP 地址。

将 HTTP 请求转发到集群是网关微服务的主要任务之一。我们将在接下来的章节中看到这方面的代码示例。更高级的网关（FlixTube 目前还没有这么高级）将具有 REST API 路由，这些路由会向多个内部微服务发出请求。然后它将多个响应整合成一个返回给前端的单一响应。

例如，想象一个 REST API，它检索单个用户的历史记录。这可能需要向用户账户微服务（FlixTube 目前还没有这个）和历史微服务发送 HTTP 请求，然后在整合响应并发送到前端之前。在这个理论示例中，网关合并了这两个 HTTP 请求的响应。

### 9.7.4 用户界面（UI）

如果您还没有机会探索 FlixTube 的用户界面，现在就去做吧。按照 9.5.2 节中的讨论构建并启动应用程序，然后使用您的网络浏览器导航到 http://localhost:4000。

图 9.10 展示了 FlixTube（视频列表）在已上传一些视频后的主页。我们可以点击列表中的任何视频来观看它。我们可以在顶部的导航栏中点击“视频”、“上传”和“历史”，在主要页面之间切换。

FlixTube 是作为一个传统的服务器端渲染网页实现的，而不是作为在浏览器中渲染的现代单页应用程序（SPA）。如果 FlixTube 是一个真正的商业应用程序，它很可能会使用 React、Angular 或 Vue 编码为一个 SPA。

FlixTube 使用 Express 和 Handlebars 模板引擎通过服务器端渲染，前端使用纯 JavaScript。FlixTube 前端是普通的 HTML、CSS 和 JavaScript，没有任何花哨的现代框架。

为什么不使用流行的现代 SPA 框架之一呢？嗯，简单的原因是这超出了本书的范围。这本书不是关于 UI 的，这就是为什么前端尽可能简单。（此外，我不想选择任何一方，挑起 SPA 框架信徒之间的战争，但所有酷炫的孩子都在用 React，对吧？）。

![图片](img/CH09_F10_Davis4.png)

图 9.10 FlixTube UI 的主页显示了已上传的视频列表。

列表 9.6 是网关微服务主代码文件的摘录。它显示了渲染主页的 HTTP GET 路由。主页显示了上传的视频列表。此路由处理程序首先从元数据微服务请求数据。然后我们使用视频列表模板渲染网页，并将视频列表作为模板的数据输入。

列表 9.6 渲染视频列表网页的网关代码（摘自第九章/示例-1/网关/src/index.js）

```
app.get("/", (req, res) => {                                 ①
    http.request(                                            ②
        {
            host: `metadata`,
            path: `/videos`,
            method: `GET`,
        },
        (response) => {
            let data = "";
            response.on("data", chunk => {
                data += chunk;
            });
            response.on("end", () => {
                res.render("video-list", { 
                 ➥ videos: JSON.parse(data).videos });       ③
            });
            response.on("error", err => {
                console.error("Failed to get video list.");
                console.error(err || `Status code: 
                ➥ {response.statusCode}`);
                res.sendStatus(500);
            });
        }
    ).end();                                                 ②
});
```

① 声明一个 HTTP GET 路由处理程序，用于检索主网页并显示上传的视频列表

② 向元数据服务发送 HTTP 请求以获取视频列表

③ 使用视频列表模板（列表 9.8 显示了模板）渲染网页。我们将视频数组作为渲染模板的数据传递。

列表 9.6 中制作 HTTP 请求的代码使用的是内置的 Node.js `http.request`函数，因此相当冗长。在看到第八章中的 Axios 之后，你可能会想知道为什么我没有使用它。

Axios 是一个出色的现代库，我强烈推荐它！它使用简单，相当灵活，并且与 Java-Script 中的新`async`和`await`关键字配合得很好。它没有在本章中使用的原因是，作为 Node.js 流控制内置库更容易，但我们还没有使用它，但很快你将看到这方面的例子。

我没有为 FlixTube 使用 JavaScript 框架，但我确实使用了 CSS 框架（Tailwind CSS）。这样我就可以制作一个漂亮的 UI，而无需与 CSS 的细节纠缠。

列表 9.7 显示了 FlixTube 的主页。这是一个包含在 Handlebars 模板中的 HTML 文档。Handlebars 是一个简单而强大的模板库，我们可以用它根据数据生成网页。如果你回顾列表 9.6，你会看到视频列表作为模板数据传递。现在在列表 9.7 中，你可以看到我们正在从这个模板数据生成一系列 HTML `div`元素。

当网络浏览器请求主页时，网关微服务会从元数据微服务请求数据。从这些数据中，它将渲染 HTML 以在用户的网络浏览器中显示。

列表 9.7 视频列表网页的 Handlebars 模板（chapter-9/example-1/gateway/src/views/video-list.hbs）

```
<!doctype html>                                                   ①
<html lang="en">
    <head>
        <meta charset="utf-8">
        <title>FlixTube: Home</title>
        <link rel="stylesheet" 
        ➥ href="css/tailwind.min.css">                           ②
        <link rel="stylesheet" href="css/app.css">                ③
    </head>
    <body>
        <div class="flex flex-col">
            <div class="border-b-2 bg-gray-100">                  ④
                <div class="nav flex flex-row items-center mt-1 p-2">
                    <div class="text-xl font-bold">
                        FlixTube
                    </div>
                    <div class="ml-16 border-b-2 border-blue-600">
                        <a href="/">Videos</a>
                    </div>
                    <div class="ml-4">
                        <a href="/upload">Upload</a>
                    </div>
                    <div class="ml-4">
                        <a href="/history">History</a>
                    </div>
                </div>
            </div>                                                ④
            <div class="m-4">                                     ⑤
                <h1>Videos</h1>
                <div id="video-list" class="m-4">                 ⑥
                    {{#if videos}}                                ⑦
                        {{#each videos}}                          ⑦
                            <div class="mt-1">                    ⑧
                                <a href="/video?id={{this._id}}">
                                ➥ {{this.name}}</a>              ⑨
                            </div>                                ⑧
                        {{/each}} 
                    {{else}} 
                        No videos uploaded yet.                   ⑩
                    {{/if}} 
                </div> 
            </div>
        </div>
    </body>
</html>
```

① 一个 HTML5 网页

② 包含 Tailwind CSS。使用 CSS 框架使 CSS 处理起来容易得多！

③ 包含 FlixTube 特定的 CSS

④ 在网页顶部渲染导航栏

⑤ 网页的主要内容

⑥ 视频列表的容器

⑦ 从数据渲染模板的 Handlebars 语法

⑧ 该元素为每个视频重复渲染。

⑨ 从模板数据渲染视频链接

⑩ 在上传视频前显示消息

### 9.7.5 视频流

FlixTube 的核心是视频流。我们早在第二章就讨论了这一点，并且它贯穿了整本书。现在，是时候看看在完成的 FlixTube 示例应用程序中视频流是如何工作的了。其中一些内容将是复习，但重要的是现在我们已经有了网关微服务和 UI，要看到它在更大的上下文中是如何工作的。

图 9.11 说明了流媒体视频的路径，从左侧的外部云存储开始，到右侧在网页浏览器中向用户显示结束。流媒体视频在其到达用户的过程中通过了三个微服务。现在，让我们通过代码跟随这段旅程。

![图 9.11 Davis4](img/CH09_F11_Davis4.png)

图 9.11 FlixTube 中流媒体视频的路径

列表 9.8 是一个摘录，展示了在 Azure 版本的视频存储微服务中流媒体视频旅程的起始点。HTTP GET /video 路由从 Azure 存储检索视频并将其流式传输到 HTTP 响应。目前如何工作的细节并不重要，但如果你想了解，请参阅第 4.4.1 节。

列表 9.8 从 Azure 存储流式传输视频（第九章/example-1/azure-storage/src/index.js 的摘录）

```
app.get("/video", (req, res) => {                      ①
    const videoId = req.query.id;                      ②

    const blobService = createBlobService();
    streamVideoFromAzure(blobService, videoId, res)    ③
        .catch(err => {                                ④
            // ... error reporting omitted ...
            res.sendStatus(500);
        });                                            ④
});
```

① HTTP GET 路由处理程序从视频存储微服务检索流媒体视频。

② 输入要检索的视频 ID 作为 HTTP 查询参数

③ 从 Azure 存储将视频流式传输到 HTTP 响应

④ 处理可能发生的任何错误

继续我们的视频流微服务之旅，列表 9.9 是一个摘录，展示了如何使用 Node.js 流将 HTTP GET /video 路由的流媒体视频从视频存储管道传输到其自己的 HTTP 响应。

视频流微服务还有另一个任务。它向应用程序中的其他微服务广播“已观看视频”消息。这种*事件驱动*编程意味着我们可以在以后决定让其他微服务响应事件，而无需更新视频流微服务的代码。

如您从第五章第 5.8 节所知，是历史微服务接收到这条消息并使用它来记录用户的观看历史。这种间接消息的使用方式使得视频流和历史微服务之间保持良好的解耦。这也突出了微服务应用程序之所以如此灵活和可扩展的原因之一。

列表 9.9 通过视频流微服务转发流式视频（摘自第九章/示例-1/video-streaming/src/index.js）

```
app.get("/video", (req, res) => {                          ①
    const videoId = req.query.id;
    const forwardRequest = http.request(                   ②
        {
            host: `video-storage`,
            path: `/video?id=${videoId}`,
            method: 'GET',
            headers: req.headers,
        }, 
        forwardResponse => {
            res.writeHeader(forwardResponse.statusCode, 
            ➥ forwardResponse.headers);
            forwardResponse.pipe(res);                     ③
        }
    );

    req.pipe(forwardRequest);                              ②
    broadcastViewedMessage(messageChannel, videoId);       ④
});
```

① 定义一个 HTTP GET 路由处理程序，从视频流微服务检索流式视频

② 将 HTTP GET 请求转发到视频存储微服务

③ 将视频存储微服务的响应（使用 Node.js 流）管道传输到该请求的响应中

④ 向其他微服务广播观看视频的消息，以便它们知道用户正在观看视频

我们的视频流之旅继续到网关微服务，这是 UI 之前的最后一站。列表 9.10 中的 HTTP GET /video 路由将流式视频从视频流微服务管道传输到其自己的 HTTP 响应。这是视频离开集群的地方，因此将视频传送到前端。

列表 9.10 通过网关微服务转发流式视频（摘自第九章/示例-1/gateway/src/index.js）

```
app.get("/api/video", (req, res) => {             ①

    const forwardRequest = http.request(          ②
        {
            host: `video-streaming`,
            path: `/video?id=${req.query.id}`,
            method: 'GET',
        }, 
        forwardResponse => {
            res.writeHeader(forwardResponse.statusCode, 
 ➥ forwardResponse.headers);
            forwardResponse.pipe(res);            ③
        }
    );

    req.pipe(forwardRequest);                     ②
});
```

① 定义一个 HTTP GET 路由处理程序，从网关微服务检索流式视频

② 将 HTTP GET 请求转发到视频流微服务

③ 将视频流微服务的响应（使用 Node.js 流）管道传输到该请求的响应中

我们的视频流之旅在 UI 中结束。您可以在列表 9.11 中看到 HTML `video`元素。`source`元素及其`src`字段触发对网关的 HTTP GET 请求，这触发了对视频流的请求，进而触发了对视频存储的请求。然后，流式视频通过视频存储、视频流、网关，最终通过用户浏览器中的`video`元素显示给用户。

列表 9.11 使用 HTML 视频元素在前端播放视频（摘自第九章/示例-1/gateway/src/views/play-video.hbs）

```
<video controls autoplay muted>                       ①
    <source src={{video.url}} type="video/mp4">       ②
    Your browser does not support the video tag.
</video>
```

① 使用 HTML 视频元素在前端显示流式视频

② 在网关微服务中链接到/api/video 路由以检索用于在视频元素中显示的流式视频

### 9.7.6 视频上传

视频流只是 FlixTube 方程的一侧。另一侧是视频上传，这是我们最初如何将视频添加到 FlixTube 的方式。视频上传在书中尚未介绍，尽管它与视频流的工作方式相似，所以您不会遇到任何麻烦。

图 9.12 展示了通过应用程序上传视频的路径。用户通过 FlixTube 前端选择视频文件并上传。上传的视频到达网关微服务的集群，然后通过视频上传微服务转发到视频存储微服务。在那里，它被安全地存储在外部云存储中。我们再次将这段旅程通过代码来追踪。

![](img/CH09_F12_Davis4.png)

图 9.12 通过 FlixTube 上传视频的路径。

图 9.13 是 FlixTube 上传网页的截图。如果你在 9.5.2 节中跟随了操作，你将已经看到这个并尝试上传一个视频。用户点击选择文件并选择要上传的文件。一旦上传完成，UI 将更新（如图 9.13 所示），以提供上传无错误完成的反馈。如果发生错误，将显示错误。

![](img/CH09_F13_Davis4.png)

图 9.13 FlixTube 上传视频的用户界面

列表 9.12 是前端代码片段，用于将视频上传到后端。这是使用 `fetch` 函数通过 HTTP POST 请求上传视频。此时，你可能正确地想到为什么我们还要使用另一个 HTTP 请求库？

嗯，通常情况下，我们会在前端使用像 Axios 这样的库。然而，这是一个没有构建过程的纯 JavaScript 网页。这使得安装像 Axios 这样的 npm 包并在我们的前端 JavaScript 代码中使用它变得相当困难；我们没有将其捆绑到前端的方法。

剩下的最简单的方法是使用浏览器自带的功能来发起 HTTP 请求。我们可以使用古老的 XMLHttpRequest 来做这件事，但这有点复杂。相反，我们将使用更现代的 `fetch` 函数，它也更容易使用。不幸的是，`fetch` 并未在旧版网络浏览器中实现，这可能会影响我们的用户群。幸运的是，我们在这里只是用它来替代无法使用 Axios 的情况。

列表 9.12 使用 `fetch` 在前端代码中上传视频（摘自 chapter-9/example-1/gateway/public/js/upload.js）

```
fetch("/api/upload", {                                ①
        body: file,                                   ②
        method: "POST",                               ③
        headers: {                                    ④
            "File-Name": file.name,
            "Content-Type": file.type,
        },                                            ④
    })
    .then(() => { 
        // ... Update the UI after the upload ...     ⑤
    })
    .catch((err) => { 
        // ... Handle the upload error ...            ⑥
    });
```

① 使用浏览器的“fetch”函数向 /api/video 路径发起 HTTP 请求

② 将要上传的文件设置为 HTTP 请求的主体

③ 设置 HTTP 方法为 POST

④ 在请求头中存储文件名和 MIME 类型

⑤ 请求成功后执行

⑥ 如果请求失败则执行

在从网络浏览器上传后，HTTP POST 请求到达网关，在那里它被以下列表中显示的 /api/upload 路由处理。在这里，我们看到请求被转发到视频上传微服务。

列表 9.13 网关微服务将 HTTP POST 请求转发到视频上传微服务（摘自 chapter-9/example-1/gateway/src/index.js）

```
app.post("/api/upload", (req, res) => {        ①
    const forwardRequest = http.request(       ②
        {
            host: `video-upload`,
            path: `/upload`,
            method: 'POST',
            headers: req.headers,
        }, 
        forwardResponse => {
            res.writeHeader(forwardResponse.statusCode, 
            ➥ forwardResponse.headers);
            forwardResponse.pipe(res);         ③
        }
    );

    req.pipe(forwardRequest);                  ④
});
```

① 定义一个 HTTP POST 路由处理程序，将视频上传到网关微服务

② 将请求转发到视频上传微服务

③ 将视频上传微服务的响应（使用 Node.js 流）通过管道传输到当前请求的响应

④ 将请求本身（请求体是视频）通过管道传输到另一个请求

列表 9.14 展示了视频上传微服务如何处理传入的视频。在此阶段，我们通过创建 MongoDB 的 `ObjectId` 类的实例为视频创建一个唯一 ID。然后，请求被转发到视频存储微服务。

上传成功后，消息“视频已上传”被广播出去，以便让其他微服务知道系统中有了新的视频。元数据微服务处理此消息并在其数据库中记录新的视频。

列表 9.14 通过 HTTP POST 处理视频上传（摘自 chapter-9/example-1/video-upload/src/index.js）

```
app.post("/upload", (req, res) => {                   ①
    const fileName = req.headers["file-name"];        ②
    const videoId = new mongodb.ObjectId();           ③
    const newHeaders = Object.assign({}, req.headers, 
    ➥ { id: videoId });                              ④
    streamToHttpPost(req, `video-storage`, 
    ➥ `/upload`, newHeaders)                         ⑤
        .then(() => {
            res.sendStatus(200);                      ⑥
        })
        .then(() => {
            // Broadcast message to the world.
            broadcastVideoUploadedMessage(            ⑦
                /* params omitted */
            );
        })
        .catch(err => {
            console.error(`Failed to capture uploaded file ${fileName}.`);
            console.error(err);
            console.error(err.stack);
        });
});
```

① 定义一个 HTTP POST 路由处理程序，用于将视频上传到视频上传微服务

② 从请求头中提取原始文件名

③ 为新视频创建一个唯一 ID

④ 将视频 ID 添加到头信息中

⑤ 将 HTTP 请求转发到视频存储微服务

⑥ 成功捕获视频以供视频存储微服务使用。这是最重要的事情；我们不能丢失用户数据！

⑦ 广播视频上传消息，以便其他微服务知道已上传新的视频。

最后，上传的视频到达视频存储微服务，您可以在列表 9.15 中看到。从这里开始，视频被保存到 Azure 存储。一旦整个链路完成，我们就成功保存了用户上传的视频副本。如果您想深入了解文件如何添加到 Azure 存储，请将视频存储微服务的完整 index.js 文件加载到 VS Code 中。

列表 9.15 从 HTTP POST 流式传输视频到 Azure 存储（摘自 chapter-9/example-1/azure-storage/src/index.js）

```
app.post("/upload", (req, res) => {                ①
    const videoId = req.headers.id;                ②
    const mimeType = req.headers["content-type"];  ②
    const blobService = createBlobService();
    uploadStreamToAzure(req, mimeType, 
    ➥ videoId, blobService)                       ③
        .then(() => {
            res.sendStatus(200);                   ④
        })
        .catch(err => { 
            // ... error reporting omitted ...
            res.sendStatus(500);                   ⑤
        });    
});
```

① HTTP POST 路由处理程序，用于将视频上传到 Azure 存储

② 从请求头中提取视频详情

③ 将视频从 HTTP 请求流式传输到 Azure 存储

④ 表示上传成功

⑤ 表示上传失败

## 9.8 使用 Terraform 手动部署 FlixTube 到生产环境

如果您已经在开发环境中运行了 FlixTube，那么这是一个巨大的进步！为了使 FlixTube 可供公众使用，我们现在必须使用与第六章和第七章中相同的技术和工具将其部署到生产环境。

本章接下来的两个部分最具挑战性，但如果您能跟上并完成这些任务，这将是一次极好的体验。如果在任何时候感觉太难，请随时回顾第六章和第七章，那里有更详细的说明。

最终，我们希望为 FlixTube 有一个 CD 管道，每当我们将更新后的代码推送到我们的托管代码仓库时，它会自动部署到生产环境。不过，在我们达到那个目标之前，我们必须首先手动部署 FlixTube。原因如下：

+   *在最初开发部署脚本时，你会逐步进行。* 随着你对部署脚本的演变，你需要一种方法来测试这些脚本，获取反馈，并修复问题。

+   *在未来，如果你在你的 CD 管道中发现问题，你需要有在开发中运行部署脚本的能力。这是必要的，这样你就可以找出并修复问题。*

图 9.14 突出了我们将要做什么。我们将使用 Terraform 在云中创建我们的基础设施。然后，我们将使用 Docker 打包和发布我们的镜像，并使用 Terraform 将容器部署到我们的 Kubernetes 集群。

![图 9.14](img/CH09_F14_Davis4.png)

图 9.14 使用 Terraform 从开发工作站手动部署 FlixTube 到生产

### 9.8.1 Terraform 脚本结构

图 9.15 显示了 FlixTube 脚本目录的布局。你将在这里识别出一些来自第六章和第七章的代码文件，但也有一些是新的。

显著的是，我们现在看到了第一次使用 Terraform 模块。在`modules/microservice`目录中的`main.tf`文件是一个可重用的 Terraform 代码模块。我们可以使用它来部署我们所有的微服务，而无需重复此代码。我们稍后将查看此模块的代码。

![图 9.15](img/CH09_F15_Davis4.png)

图 9.15 脚本子目录包含将 FlixTube 部署到生产的 Terraform 脚本

### 9.8.2 前提条件

要部署 FlixTube，你需要安装一些工具。如果你在第六章和第七章中跟随了操作，你已经有这些工具了；如果没有，你现在可以安装这些工具。首先，你需要 Azure CLI 工具。检查它是否已安装，方法如下：

```
az --version
```

如果你还没有 Azure CLI 工具，请按照以下安装说明进行操作：

[`docs.microsoft.com/en-us/cli/azure/install-azure-cli`](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)

你还需要安装 Terraform。检查你是否已经安装了它，方法如下：

```
terraform --version
```

否则，从以下链接安装最新版本：

[`www.terraform.io/downloads.html`](https://www.terraform.io/downloads.html)

### 9.8.3 Azure 身份验证

在将基础设施部署到 Azure 之前，你首先需要使用你的账户进行身份验证。如果你在第六章和第七章中跟随了操作，你已经完成了这项操作。如果没有，请参阅第 6.6.2 节以获取详细说明。使用 Azure CLI 工具，通过输入以下命令来检查你当前正在使用哪个账户：

```
az account show
```

复制输出，因为你很快就需要从`id`字段（你的 Azure 订阅 ID）和`tenantID`字段中获取值。

注意：确保你使用的是正确的账户！如果你不小心将基础设施部署到工作账户，可能会很尴尬。

你还需要创建一个服务主体，你的 Kubernetes 集群将使用它来与你的 Azure 账户交互（例如，当它为你创建 Azure 负载均衡器时）。如果你已经在第六章中有一个服务主体，你现在可以重用它；否则，创建一个新的，如下所示：

```
az ad sp create-for-rbac --role="Contributor" 
➥-scopes="/subscriptions/<subscription-id>"
```

一定要将 `<subscription-id>` 替换为你刚才记下的实际订阅 ID。然后复制输出结果，你很快就会需要它。你需要的是来自 `appId`（在我们的 Terraform 脚本中我们称之为 `client_id`）和 `password`（我们称之为 `client_secret`）字段中的值。有关创建服务主体的更详细说明，请参阅第 6.11.2 节。

### 9.8.4 配置存储

要将 FlixTube 部署到生产环境，你还需要一个 Azure 存储账户。Azure 存储微服务使用此账户来存储和检索视频。如果你在第四章中跟随操作，你已经有一个账户，并且如果你愿意可以重用它。否则，按照第 4.4.1 节中的说明创建一个存储账户。

你需要在你的存储账户中创建一个视频容器。请注意，这并不是一个 Docker *容器*；这是 Azure 存储中的容器概念，一个可以存储任意文件的容器。记下你的存储账户名称和访问密钥。你很快就会需要这些信息。

### 9.8.5 部署应用程序

现在是时候调用 Terraform 来评估我们的脚本并将 FlixTube 部署到生产环境了。首先，我们必须初始化 Terraform。为此，初始化 Terraform 并安装我们需要的各种提供者：

```
cd chapter-9/example-1/scripts
terraform init
```

现在调用 Terraform 来部署我们的基础设施：

```
terraform apply
```

在开始之前，你必须为以下输入变量提供值：

+   `app_version` —第一次你可以直接输入 1。在随后的 `terraform apply` 调用中，你应该增加这个数字。

+   `client_id` —你在第 9.8.3 节中记下的 Azure 服务主体的 ID。

+   `client_secret` —你的服务主体的密码。

+   `storage_account_name` —你在第 9.8.4 节中记下的用于存储视频的 Azure 存储账户名称。

+   `storage_access_key` —你的存储账户的访问密钥。

部署 FlixTube 需要一些时间。你可以随意泡一杯茶（或者几杯）。

### 9.8.6 检查是否工作

要检查 FlixTube 是否已部署并正常工作，我们可以将其前端加载到我们的网页浏览器中。为此，我们必须知道它的 IP 地址。就像我们在第七章的各个部分中所做的那样，我们可以使用 Kubernetes CLI 工具来完成这个任务：

```
kubectl get services
```

要记住如何安装和使用 Kubectl，请回顾第 6.12 节。你可以看到 Kubectl 显示在图 9.16 中的表格输出。在网关容器的 `EXTERNAL-IP` 列中找到 IP 地址。将 IP 地址复制到你的网页浏览器中。

![图片](img/CH09_F16_Davis4.png)

图 9.16 使用 Kubernetes 命令行工具获取网关的 IP 地址，以便我们可以在浏览器中测试 FlixTube

你不需要使用任何特定的端口号。我们使用 4000 端口来访问 FlixTube 的开发版本，但生产中的 FlixTube 配置为使用端口 80。这是 HTTP 的默认端口（因为它默认，所以我们不需要指定它）。

你可能已经注意到我们在这里使用的是 HTTP 协议。这就是为什么浏览器会在 Fixtube 的 IP 地址旁边显示“不安全”。出于安全考虑，我们实际上应该使用（就像所有现代 Web 服务器一样）HTTP 的安全版本，称为 HTTPS。这代表安全超文本传输协议，我们将在第十一章再次提到。如果一切按计划进行，你现在应该能够导航到 FlixTube UI 来上传和播放视频。

在这个阶段，你可以随意尝试使用 FlixTube 和 Terraform。你可以修改 FlixTube 或 Terraform 代码，然后使用`terraform apply`应用你的更改。你可以这样做多少次就做多少次。

### 9.8.7 拆卸

当你完成 FlixTube 后，确保清理所有内容。在云中运行此基础设施将花费你金钱。如果你刚刚创建了 Azure 账户，你将使用它提供的免费信用额度来尝试它，但让我们不要浪费它。当你完成使用后，销毁你的基础设施：

```
terraform destroy
```

### 9.8.8 Terraform 模块

第九章代码库中的大部分 Terraform 代码与第六章和第七章中已经看到的代码相同。然而，这里有一个需要进一步解释的新内容。

记得部署微服务的 Terraform 代码吗？如果你需要复习，可以快速浏览第 7.6 节。我们可以通过简单地重复相同的代码来应对 FlixTube 中的每个微服务。但大部分代码是相同的，所以这不是一种高效的工作方式。为了应对这种情况，我们将引入 Terraform 的一个更高级的功能：Terraform 模块。

Terraform 模块允许我们编写可重用的代码模块，我们可以通过提供不同的输入变量在不同的环境中使用它们。列表 9.16 显示了用于部署 FlixTube 六个微服务的 Terraform 模块。这看起来或多或少就像任何其他 Terraform 代码文件。

列表从 Kubernetes 部署开始，该部署将微服务实例化到我们的 Kubernetes 集群中。它以 Kubernetes 服务结束，该服务通过 DNS 使微服务在集群内可访问。请注意，Kubernetes 服务的`type`字段是参数化的，这样我们就可以为微服务启用或禁用 Azure 负载均衡器。你将在下一章中了解更多关于负载均衡器可以做什么的信息。现在，这是为了我们可以分配一个 IP 地址给网关微服务，并使其对外界可访问。（这就是我们的客户将如何与我们应用程序交互的方式。）

列表 9.16 中的 Terraform 代码没有特别特殊的地方，除了它位于 modules/microservice 子目录中。请注意列表开头定义的多个输入变量。这些是正常的 Terraform 变量，但在这个例子中，这些是允许我们定制模块行为的输入。例如，我们可以通过`service_name`变量设置微服务的名称。我们还必须传递托管微服务镜像的容器注册库的详细信息。另一个重要的变量是`env`。它允许我们为每个微服务单独配置环境变量集。

列表 9.16 一个可重用的 Terraform 模块将微服务部署到 Kubernetes（摘自第九章/示例 1/脚本/modules/microservice/main.tf）

```
variable "app_version" {}                                                   ①
variable "service_name" {}
variable "dns_name" {
    default = ""
}
variable "login_server" {}
variable "username" {}
variable "password" {}
variable "service_type" {
    default = "ClusterIP"
}
variable "session_affinity" {
    default = ""
}
variable "env" {
    default = {}
    type = map(string)
}                                                                           ①
locals {                                                                    ②
    image_tag = "${var.login_server}/${var.service_name}:${var.app_version}"②
}                                                                           ②
# ... much code omitted for brevity ...
resource "kubernetes_deployment" 
➥ "service_deployment" {                                                   ③
    depends_on = [ null_resource.docker_push ]
    metadata {
        name = var.service_name                                             ④
    labels = {
            pod = var.service_name
        }
    }
    spec {
        replicas = 1
        selector {
            match_labels = {
                pod = var.service_name
            }
        }
        template {
            metadata {
                labels = {
                    pod = var.service_name
                }
            }
            spec {
                container {
                    image = local.image_tag
                    name  = var.service_name                                ④
                    env {
                        name = "PORT"
                        value = "80"
                    }
                    dynamic "env" {                                         ⑤
                        for_each = var.env
                        content {
                          name = env.key
                          value = env.value
                        } 
                    }                                                       ⑤
               }
                image_pull_secrets {
                    name = 
                    ➥ kubernetes_secret.docker_credentials.metadata[0].name
                }
            }
        }
resource "kubernetes_service" "service" {                                   ⑥
    metadata {
        name = var.dns_name != "" 
        ➥ ? var.dns_name : var.service_name                                ⑦
    }
    spec {
        selector = {
            pod = kubernetes_deployment.service_deployment.metadata[0].labels.pod
        }   
        session_affinity = var.session_affinity                             ⑦
        port {
            port        = 80
            target_port = 80
        }
        type             = var.service_type                                 ⑧
```

① 定义此 Terraform 模块的输入变量

② 定义在此模块中使用的局部变量

③ 将容器部署到我们的 Kubernetes 集群

④ 使用变量来为每个微服务定制此模块的配置

⑤ 使用变量来设置针对每个微服务特定的环境变量

⑥ 部署一个服务，使容器可以通过 DNS 从其他容器访问

⑦ 使用变量来为每个微服务定制此模块的配置

⑧ 使用变量来为每个微服务定制此模块的配置

列表 9.17 展示了我们如何使用 Terraform 模块来部署我们的微服务。这里只展示了网关微服务。其他服务被省略，因为它们看起来几乎相同，尽管它们的环境变量配置可能有所不同。在下面的列表中，请注意模块是如何导入的，其源文件是如何指定的，以及环境变量是如何配置的。

列表 9.17 Terraform 微服务模块将网关微服务部署到 Kubernetes（摘自第九章/示例 1/脚本/microservices.tf）

```
locals {                                                                     ①
    login_server = azurerm_container_registry.container_registry.login_server①
    username = azurerm_container_registry.container_registry.admin_username  ①
    password = azurerm_container_registry.container_registry.admin_password  ①
    rabbit = "amqp://guest:guest@rabbit:5672"                                ①
    database = "mongodb://db:27017"                                          ①
}                                                                            ①
module "gateway-microservice" {                                            ②
                                                                           ②
    source ="./modules/microservice"                                       ②③
                                                                           ②
    service_name = "gateway"                                               ②④
    service_type = "LoadBalancer"                                          ②④
    session_affinity = "ClientIP"                                          ②④
    login_server = local.login_server                                      ②④
    username = local.username                                              ②④
    password = local.password                                              ②④
    app_version = var.app_version                                          ②④
                                                                           ②
    env = {                                                                ②⑤
        RABBIT: local.rabbit                                               ②⑤
    }                                                                      ②⑤
}                                                                          ②
# ... all other microservices omitted for brevity ...
```

① 设置用于整个脚本的局部变量

② 导入微服务 Terraform 模块（来自列表 9.16），以部署我们的网关微服务

③ 指定从子目录./modules/microservice 加载的模块的来源，其中包含 main.tf 文件（来自列表 9.16）

④ 将输入变量设置为配置网关微服务的微服务模块

⑤ 配置针对单个微服务的特定环境变量

Terraform 模块是 Terraform 的高级功能之一，而且 Terraform 中还有更多内容等待您去探索。请参阅本章末尾，以获取深入了解 Terraform 的参考信息。

## 9.9 持续交付到生产环境

在手动将 FlixTube 部署到生产环境之后，我们现在可以上线持续交付（CD）管道。

您可以跟随操作，但这可能比上一节更具挑战性，尤其是如果出现问题的话！您可能需要回到手动部署（我们在 9.7 节中刚刚做过的）来找出问题。

就像在第七章中做的那样，我们将使用 Bitbucket Pipelines 创建我们的 CD 管道。对于您来说，将其转移到任何其他 CD 平台应该相当容易。正如我在第七章中所说的，CD 管道实际上只是一个被美化的 shell 脚本，即使某些提供商也提供了花哨的 UI。

将您的部署 shell 脚本从一个提供商迁移到另一个提供商并不困难。然而，Bitbucket 因其提供免费层而具有很高的性价比。在尝试将其迁移到不同的持续交付（CD）提供商之前，遵循这里的说明来练习使其工作是有价值的。图 9.17 展示了 FlixTube 的 CD 管道结构。

![](img/CH09_F17_Davis4.png)

图 9.17 FlixTube 的持续交付（CD）管道

### 9.9.1 前提条件

您需要一个 Bitbucket 账户。如果您在第七章中跟随了操作，您已经有了这个账户。否则，请前往[`bitbucket.org`](https://bitbucket.org)注册一个免费账户。

### 9.9.2 设置您的代码存储库

下一步是将 FlixTube 的代码导入到 Bitbucket 代码存储库中。将第九章代码存储库中 example-1 子目录的全部内容复制到一个新位置。在这里创建一个新的 Git 仓库，然后将代码推送到您的托管 Bitbucket 仓库。接下来，为存储库启用 Bitbucket Pipelines。请参阅 7.7.2 节以获取有关 Bitbucket 存储库设置的详细说明。

现在，配置您的环境变量以用于存储库。您需要添加与 Azure 身份验证相关的变量，就像在 7.7.6 节中所做的那样。出于安全原因，我们将敏感配置细节作为存储库变量存储，而不是将这些存储在代码中。就像在第七章中做的那样，添加 ARM_CLIENT_ID、ARM_CLIENT_SECRET、ARM_TENANT_ID 和 ARM_SUBSCRIPTION_ID 的变量。您在 9.8.3 节中已经记录了这些变量的值。

此外，我们还需要添加一些新变量来验证视频存储微服务对存储视频的 Azure 存储账户的访问权限。为此，也添加 STORAGE_ACCOUNT_NAME 和 STORAGE_ACCESS_KEY 的变量，并将这些值设置为在 9.8.4 节中记录的值。

### 9.9.3 准备后端

在您的 CD 管道首次调用之前，您需要配置后端，以便 Terraform 的状态文件在后续调用之间持久化。关于 Terraform 状态的复习，请参阅 6.8.7 节和 7.7.4 节。

为 Terraform 创建一个不同的 Azure 存储容器。您可以使用在第七章中创建的容器，或者创建一个新的容器。不要重复使用视频容器！这样会使用同一个容器进行不同的目的，这最终会使理解并推理您的应用程序变得更加困难。

Terraform 脚本 backend.tf 已经配置为将 Terraform 状态存储在我们的 Azure 存储账户中。您只需取消注释该文件中的代码（您在手动部署 FlixTube 时注释了它）。请确保设置详细信息以匹配您自己的存储账户和容器。

列表 9.18 显示了在取消注释代码后的 backend.tf。请确保将资源组和存储账户重命名为您自己的 Azure 账户中存在的名称。您还需要创建一个名为`terraform`的 Azure 存储容器。Terraform 将在此容器下以`terraform.tfstate`的名称持久化其状态。

列表 9.18 Terraform 后端配置（第九章/示例-1/脚本/backend.tf）

```
terraform {
    backend "azurerm" {
        resource_group_name  = 
        ➥ "<your-resource-group>"              ①
        storage_account_name = 
        ➥ "<your-storage-account>"             ②
        container_name = "terraform"            ③
        key = "terraform.tfstate"               ④
    }
}
```

① 设置包含存储账户的资源组名称。将其重命名为您 Azure 账户中存在的资源组。

② 设置存储 Terraform 状态的存储账户名称。将其重命名为您 Azure 账户中存在的存储账户。

③ 存储 Terraform 状态的容器名称。无需重命名，但请确保该容器存在于您的 Azure 账户中。

④ 存储 Terraform 状态的文件名。这可以设置为任何名称，但我们使用 Terraform 状态文件的默认名称，因为这更有意义，我们也容易记住它的含义。

### 9.9.4 部署 shell 脚本

如前所述，shell 脚本通常是任何持续交付（CD）管道的核心。列表 9.19 是 FlixTube 的部署 shell 脚本。请注意，它与第七章中的部署脚本几乎没有区别。在列表 9.19 中，为了简洁起见，省略了一些代码，并且有几个额外的环境变量被传递给 Terraform。

我们的大部分部署代码都在 Terraform 代码中，这就是为什么这个 shell 脚本如此之小。如果我们想在这里直接做更多的事情，比如构建和发布 Docker 镜像，我们完全可以做到。但至少对于 FlixTube 来说，我们已经成功地将整个部署包含在 Terraform 代码中。要更详细地修订部署 shell 脚本，请回顾第 7.7.3 节。

列表 9.19 使用 Terraform 进行部署的 shell 脚本（第九章/示例-1/脚本/deploy.sh 摘录）

```
cd ./scripts                                              ①
terraform init                                            ②
terraform apply -auto-approve \                           ③
    -var "app_version=$VERSION" \                         ④
    -var "client_id=$ARM_CLIENT_ID" \                     ④
    -var "client_secret=$ARM_CLIENT_SECRET" \             ④
    -var "storage_account_name=$STORAGE_ACCOUNT_NAME" \   ④
    -var "storage_access_key=$STORAGE_ACCESS_KEY"         ④
```

① 修改包含我们的 Terraform 脚本的目录

② 调用 Terraform 初始化

③ 启用自动批准后调用 Terraform apply，然后运行我们的 Terraform 脚本并部署我们的基础设施和微服务

④ 通过环境变量传递给我们的 Terraform 脚本

### 9.9.5 FlixTube 的 CD 配置

CD 管道拼图中最后一块是配置文件。对于 Bitbucket Pipelines，这是一个放置在你代码仓库根目录的 YAML 文件。它被称为 bitbucket-pipelines.yaml。一些其他 CD 提供商使用类似的 YAML 格式。

列表 9.20 显示了 FlixTube 的简单 CD 管道配置。这是因为我们在这里真正做的只是调用我们在列表 9.19 中看到的部署 shell 脚本。有关 Bitbucket Pipelines 配置的更多详细信息，请参阅第 7.7.5 节。

列表 9.20 Bitbucket Pipelines 的 CD 配置（chapter-9/example-1/bitbucket-pipelines.yaml）

```
image: hashicorp/terraform:0.12.6
pipelines:
    default:
      - step:
          name: Build and deploy
          services:
            - docker
          script:
            - export VERSION=$BITBUCKET_BUILD_NUMBER
            -  chmod +x ./scripts/deploy.sh
            - ./scripts/deploy.sh               ①
```

① 调用我们的部署 shell 脚本

### 9.9.6 测试持续交付（CD）管道

现在我们准备测试我们的 CD 管道。假设你的仓库已配置并且你已启用 Bitbucket Pipelines（见第 9.9.2 节），我们需要推送代码更新。你还可以在 Bitbucket Pipelines 仪表板中手动触发 CD 管道，但通常触发部署的方式是更改一些代码并将更改推送到我们的托管仓库。让我们测试一下它是否工作。

尝试推送一个代码更改——只需要进行一个小改动即可。也许更改一下 UI 中的文本？然后保存文件，提交更改，并将其推送到 Bitbucket。然后你可以在 Bitbucket Pipelines 仪表板中观察管道被触发。

注意：第一次调用管道时，由于部署了你的基础设施和微服务的第一个实例，所以会花费一些时间。

一旦准备就绪，你就可以再次使用 `kubectl get services`（如第 9.8.6 节中所述）来获取网关的 IP 地址，然后在你的网页浏览器中加载并进行一些测试。现在你已经为持续部署做好了准备！你推送到 Bitbucket 的任何代码更改都将自动部署到生产环境。

### 9.9.7 添加自动化测试

你可以使用 CD 管道执行的最后一步是添加自动化测试。第九章的示例代码包括一些你可能已经在第 9.6 节中尝试过的自动化测试。将自动化测试添加到你的应用程序就像将正确的命令放在正确的位置一样简单。

这是因为我们遵循了约定。这意味着我们只需要知道一个命令，那就是 `npm test`。我们不需要记住我们是否在使用 Jest、Cypress 或其他 JavaScript 测试框架。无论我们使用什么，我们只需要确保 `npm test` 的脚本配置正确，以便调用它。

至于调用此命令的正确位置，这更困难，因为我们有一个完整的应用程序在一个仓库中，并且配置了一个 CD 管道。我们可以从我们的部署 shell 脚本（列表 9.21）或直接从我们的 Bitbucket Pipelines 配置文件（列表 9.22）中调用 `npm test`。列表 9.21 和 9.22 显示了这两种方法可能适用于元数据微服务。

如你所想，这仅仅是冰山一角。随着我们为其他微服务构建更多的自动化测试，我们不得不为每个微服务重复调用 `npm test`。这看起来并不优雅，但当我们过渡到多仓库可扩展部署架构时，这个问题将会得到解决。只需耐心等待第十一章的更多内容。

列表 9.21 将自动化测试添加到部署 shell 脚本中

```
set -e              ①
cd ./metadata       ②
npm install         ③
npm test            ④
cd ..               ⑤

cd ./scripts
terraform init
terraform apply -auto-approve \
    -var "app_version=$VERSION" \
    -var "client_id=$ARM_CLIENT_ID" \
    -var "client_secret=$ARM_CLIENT_SECRET" 
```

① 导致 shell 脚本中后续失败的命令使整个脚本失败

② 切换到元数据微服务目录

③ 安装依赖项（这会安装 Jest）

④ 运行测试。如果失败，shell 会以错误代码终止。这反过来又会导致 CD 管道因错误而终止。

⑤ 切换回主项目目录

列表 9.22 将自动化测试直接添加到 CD 配置文件中

```
image: hashicorp/terraform:0.12.6
pipelines:
    default:
      - step:
          name: Build and deploy
          services:
            - docker
          script:
            - cd metadata && npm install && npm test     ①
            - export VERSION=$BITBUCKET_BUILD_NUMBER
            -  chmod +x ./scripts/deploy.sh
            - ./scripts/deploy.sh
```

① 直接从 Bitbucket Pipelines 配置文件中调用我们的自动化测试。如果测试失败，CD 管道会因错误而终止。

我们甚至可以直接从我们的 Terraform 代码中调用我们的自动化测试。但是，可以说，我们已经用构建和发布 Docker 镜像做了太多的事情，在 Terraform 中。我们已经将其用途扩展到了其原始的云基础设施供应目的之外，尽管这是启动我们的微服务应用程序的一种方便方式。在第十一章中，我们将讨论如何随着 FlixTube 的扩展来重构我们的开发和部署流程。

如第八章所述，在 CD 管道中运行 Cypress 实质上是一样的，尽管有一些额外的困难。我们再次将 `npm test` 调用，但这次配置为调用 Cypress 而不是 Jest。

Cypress 的问题在于它很大！每次管道被调用时都将其安装到我们的 CD 管道中是缓慢且低效的（尤其是如果你按分钟支付管道执行时间的话）。如果你想在 CD 管道中使用 Cypress，那么你将不得不学习如何使用你的 CD 提供者的缓存设施。但恐怕这本书已经太长了，所以你将不得不自己解决这个问题。你可以在 Bitbucket Pipelines 中了解更多关于缓存的信息：

[`support.atlassian.com/bitbucket-cloud/docs/cache-dependencies/`](https://support.atlassian.com/bitbucket-cloud/docs/cache-dependencies/)

## 9.10 回顾

恭喜！如果你跟随着本章的内容，你现在已经在生产环境中运行了 FlixTube，并且已经准备好继续演进 FlixTube。你可以进行代码更改，在开发环境中测试它们，然后使用 CD 部署更新到生产环境。使用表 9.2 来回顾本章中使用的命令。

表 9.2 第九章命令回顾（续）

| 命令 | 描述 |
| --- | --- |
| `npm start` | 不论主脚本文件命名如何或它期望的命令行参数如何，都是启动 Node.js 应用程序的传统 npm 脚本。通常，这会在 package.json 文件中转换为`node index.js`，但这完全取决于项目的作者以及他们如何设置。好事是，无论特定项目结构如何，您只需记住`npm start`即可。 |
| `npm run start:dev` | 我启动 Node.js 项目开发时的个人约定。我将此添加到 package.json 中的脚本中，通常它会运行类似 Nodemon 的东西，以便在您工作时实时重新加载您的代码。 |
| `docker-compose up`➥ `--build` | 根据当前工作目录中定义的 Docker Compose 文件（docker-compose.yaml）构建和实例化由多个容器组成的应用程序 |
| `docker-compose`➥ `down` | 停止并销毁应用程序，使开发工作站保持干净状态 |
| `npm test` | 运行测试的 npm 脚本约定。这可以运行 Jest 或 Cypress（甚至两者都可以），具体取决于您如何配置 package.json 文件。这是您应在 CD 管道中运行的命令，以执行测试套件。 |
| `npm run test:watch` | 这是我在实时重新加载模式下运行测试的个人约定。您需要配置 package.json 文件中的此脚本才能使用它。 |
| `terraform init` | 初始化 Terraform 项目并下载提供者插件 |
| `terraform apply` | 在工作目录中执行 Terraform 脚本，以增量方式对我们的基础设施应用更改 |
| `terraform destroy` | 销毁由 Terraform 项目创建的所有基础设施 |

## 9.11 FlixTube 的未来

FlixTube 接下来该往哪里发展？这需要您来想象！在第十一章中，我们将讨论 FlixTube 未来的技术方面：

+   我们如何扩大规模以适应不断增长的用户群体？

+   随着应用程序的增长和开发团队规模的增加，我们如何扩大我们的开发和部署流程？

目前，只需想象一下您将来希望添加到 FlixTube 中的微服务类型。图 9.18 为您提供了关于它随着增长可能看起来怎样的灵感。

## 9.12 继续你的学习

在本章中，我们研究了 FlixTube 示例应用程序的结构和布局。我们在开发环境中构建、运行和测试了它。然后我们通过其 CD 管道将其部署到生产环境。

您已经运行了 FlixTube，接下来该做什么呢？阅读任何书籍都只能让您走这么远。保留这些技能的关键是练习，练习，然后再练习。尝试对代码进行实验。尝试添加功能。尝试添加新的微服务。尝试破坏 FlixTube 以查看会发生什么。练习开发艺术是让您达到下一个层次的关键。

![](img/CH09_F18_Davis4.png)

图 9.18 FlixTube 未来可能的样子

开发并非没有挑战。事实上，它是一连串不断的问题和解决方案的永无止境的过山车。当你遇到任何工具或技术的难题时，回到这本书中相应的章节进行回顾。你可能会找到你需要的答案。否则，你可能需要深入研究并探索其他资源。

本书最后几章包含的指导将帮助您在微服务未来的开发道路上导航。每章末尾的参考文献（包括本章）将帮助您继续您的学习之旅。但请记住，您成功的关键和保持这些技能的关键是持续的练习。

要了解 UI 开发，请参阅以下书籍：

+   *《Angular 实战》* by *Jeremy Wilken (Manning, 2018)*

+   *《使用 Mongo, Express, Angular 和 Node 实现 MEAN》*，第 2 版，by *Simon D. Holmes and Clive Harber (Manning, 2019)*

+   *《微前端实战》* by *Michael Geers (Manning, 2020)*

要了解更多关于使用微服务进行开发的信息，请参阅以下书籍：

+   *《微服务实战》* by Morgan Bruce, Paulo A. Pereira *(Manning, 2018)*

+   *《微服务模式》* by Chris Richardson *(Manning, 2018)*

+   *《微服务之道》* by Richard Rodger *(Manning, 2017)*

+   *《.NET Core 微服务》*，第 2 版，by Christian Horsdal Gammelgaard *(Manning, 2020)*

+   *《使用 Python 开发微服务 API》* by José Haro Peralta *(Manning, est, Spring 2021)*

要深入了解 Terraform，请参阅以下书籍：

+   *《Terraform 实战》* by *Scott Winkler (Manning, est Spring, 2021)*

## 摘要

+   我们了解了 FlixTube 的整体工作原理，并在过程中遇到了一些新的微服务。

+   我们回顾了构建、运行、测试和部署微服务所需的一些基本工具。

+   我们在 FlixTube 应用程序的生产模式和开发模式下分别运行了一个单独的微服务，这使我们可以实现实时重载，然后再启动整个应用程序。

+   我们使用 Jest 和 Cypress 测试了 FlixTube。

+   为了使 FlixTube 可供公众使用，我们将其部署到生产环境，并上线了持续交付（CD）管道。
