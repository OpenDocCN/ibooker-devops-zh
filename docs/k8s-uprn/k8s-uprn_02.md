# 第二章：创建和运行容器

Kubernetes 是一个用于创建、部署和管理分布式应用的平台。这些应用有各种不同的形状和规模，但最终它们都由在各个单独的机器上运行的一个或多个程序组成。这些程序接受输入，处理数据，然后返回结果。在我们考虑构建分布式系统之前，我们必须首先考虑如何构建*应用程序容器镜像*，这些镜像包含这些程序并构成我们分布式系统的组成部分。

应用程序通常由语言运行时、库和您的源代码组成。在许多情况下，您的应用程序依赖于外部共享库，如`libc`和`libssl`。这些外部库通常作为操作系统的共享组件随特定机器上安装的操作系统一起提供。

这种对共享库的依赖性会在程序员的笔记本上开发的应用程序依赖于在将程序部署到生产操作系统时不可用的共享库时造成问题。即使开发和生产环境共享完全相同版本的操作系统，当开发人员忘记将依赖的资产文件包含在他们部署到生产环境的包中时，也可能会出现问题。

在单个机器上运行多个程序的传统方法要求所有这些程序在系统上共享相同版本的共享库。如果不同的程序由不同的团队或组织开发，这些共享依赖性会增加不必要的复杂性和团队之间的耦合。

一个程序只有在可以可靠地部署到应该运行的机器上时才能成功执行。部署的最新状态往往涉及运行命令式脚本，这些脚本不可避免地会有复杂和扭曲的失败情况。这使得部署分布式系统的新版本或部分版本成为一项费时且困难的任务。

在第一章中，我们强烈主张不可变映像和基础设施的价值。这种不可变性正是容器映像提供的。正如我们将看到的，它轻松解决了刚刚描述的所有依赖管理和封装问题。

在处理应用程序时，将其打包成便于与他人共享的方式通常是很有帮助的。Docker 是大多数人用于容器的默认工具，它使得将可执行文件打包并推送到远程注册表变得很容易，以便其他人稍后可以拉取。在撰写本文时，容器注册表在所有主要公共云中都可用，并且在许多云中也提供构建镜像的服务。您还可以使用开源或商业系统运行自己的注册表。这些注册表使用户能够轻松管理和部署私有镜像，而镜像构建器服务则提供了与持续交付系统的简单集成。

在本章以及本书的其余部分中，我们将使用一个简单的示例应用程序来演示这一工作流程。您可以在 GitHub 上找到该应用程序 [on GitHub](https://oreil.ly/unTLs)。

容器镜像将程序及其依赖项捆绑到一个根文件系统下的单个构件中。最流行的容器镜像格式是 Docker 镜像格式，已由开放容器倡议标准化为 OCI 镜像格式。Kubernetes 通过 Docker 和其他运行时同时支持 Docker 和 OCI 兼容镜像。Docker 镜像还包括供容器运行时使用的附加元数据，以根据容器镜像的内容启动运行中的应用实例。

本章涵盖以下主题：

+   如何使用 Docker 镜像格式打包应用程序

+   如何使用 Docker 容器运行时启动应用程序

# 容器镜像

对于几乎每个人来说，他们与任何容器技术的第一次互动都是通过容器镜像。*容器镜像*是一个二进制包，封装了在操作系统容器内运行程序所需的所有文件。根据您首次尝试容器的方式，您将从本地文件系统构建容器镜像，或者从*容器注册表*下载现有镜像。无论哪种情况，一旦容器镜像存在于您的计算机上，您就可以运行该镜像以在操作系统容器内生成运行中的应用程序。

最流行和普遍的容器镜像格式是 Docker 镜像格式，由 Docker 开源项目开发，用于使用`docker`命令打包、分发和运行容器。随后，Docker 公司及其他人员开始通过 Open Container Initiative（OCI）项目标准化容器镜像格式。虽然 OCI 标准在 2017 年中发布了 1.0 版本，但这些标准的采纳进展缓慢。Docker 镜像格式仍然是事实上的标准，由一系列文件系统层组成。每个层在文件系统中添加、删除或修改前一层的文件。这是*覆盖*文件系统的一个例子。覆盖系统在打包图像时和实际使用图像时都会使用。在运行时，有多种不同的具体文件系统实现，包括`aufs`、`overlay`和`overlay2`。

容器镜像通常与容器配置文件结合使用，该文件提供了如何设置容器环境并执行应用程序入口点的说明。容器配置通常包括如何设置网络、命名空间隔离、资源约束（cgroups）以及应该对运行中的容器实例施加什么样的`syscall`限制。容器根文件系统和配置文件通常使用 Docker 镜像格式捆绑在一起。

容器主要分为两大类：

+   系统容器

+   应用容器

系统容器旨在模仿虚拟机，并经常运行完整的启动过程。它们通常包含一组通常在虚拟机中找到的系统服务，例如`ssh`、`cron`和`syslog`。在 Docker 刚推出时，这些类型的容器非常常见。随着时间的推移，它们被认为是不良实践，应用容器逐渐受到青睐。

应用容器与系统容器不同之处在于，它们通常运行单个程序。虽然每个容器运行单个程序可能看起来是一个不必要的约束，但这提供了组合可扩展应用程序所需的理想粒度，并且是 Pods 大量利用的设计哲学。我们将详细查看 Pods 在第五章中的工作原理。

# 使用 Docker 构建应用程序镜像

总的来说，像 Kubernetes 这样的容器编排系统专注于构建和部署由应用容器组成的分布式系统。因此，本章剩余部分将重点放在应用容器上。

## Dockerfile

可以使用 Dockerfile 自动创建 Docker 容器镜像。

让我们从为一个简单的 Node.js 程序构建一个应用程序镜像开始。对于许多其他动态语言（如 Python 或 Ruby），这个例子都非常类似。

最简单的 npm/Node/Express 应用程序有两个文件：*package.json*（示例 2-1）和 *server.js*（示例 2-2）。将它们放入一个目录中，然后运行 `npm install express --save` 来建立对 Express 的依赖并安装它。

##### 示例 2-1\. package.json

```
{
  "name": "simple-node",
  "version": "1.0.0",
  "description": "A sample simple application for Kubernetes Up & Running",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "author": ""
}
```

##### 示例 2-2\. server.js

```
var express = require('express');

var app = express();
app.get('/', function (req, res) {
  res.send('Hello World!');
});
app.listen(3000, function () {
  console.log('Listening on port 3000!');
  console.log('  http://localhost:3000');
});
```

要将其打包为 Docker 映像，创建两个额外的文件：*.dockerignore*（示例 2-3）和 Dockerfile（示例 2-4）。Dockerfile 是构建容器映像的配方，而 *.dockerignore* 定义了在将文件复制到映像中时应忽略的文件集。有关 Dockerfile 语法的完整描述，请访问[Docker 网站](https://dockr.ly/2XUanvl)。

##### 示例 2-3\. .dockerignore

```
node_modules
```

##### 示例 2-4\. Dockerfile

```
# Start from a Node.js 16 (LTS) image ![1](img/1.png)
FROM node:16

# Specify the directory inside the image in which all commands will run ![2](img/2.png)
WORKDIR /usr/src/app

# Copy package files and install dependencies ![3](img/3.png)
COPY package*.json ./
RUN npm install
RUN npm install express

# Copy all of the app files into the image ![4](img/4.png)
COPY . .

# The default command to run when starting the container ![5](img/5.png)
CMD [ "npm", "start" ]
```

![1](img/#co_creating_and_running_containers_CO1-1)

每个 Dockerfile 都基于其他容器映像构建。此行指定我们从 Docker Hub 上的`node:16`映像开始。这是一个预配置的带有 Node.js 16 的映像。

![2](img/#co_creating_and_running_containers_CO1-2)

此行设置容器映像中所有后续命令的工作目录。

![3](img/#co_creating_and_running_containers_CO1-3)

这三行初始化了 Node.js 的依赖关系。首先，我们将包文件复制到映像中。这将包括 *package.json* 和 *package-lock.json*。然后，`RUN` 命令在容器中运行正确的命令以安装必要的依赖项。

![4](img/#co_creating_and_running_containers_CO1-4)

现在，我们将其余的程序文件复制到映像中。这将包括除了 *node_modules* 之外的所有内容，因为 *.dockerignore* 文件排除了它。

![5](img/#co_creating_and_running_containers_CO1-5)

最后，我们指定了容器运行时应运行的命令。

运行以下命令以创建 `simple-node` Docker 映像：

```
$ docker build -t simple-node .
```

当您想要运行此映像时，可以使用以下命令。导航至 *http://localhost:3000* 以访问在容器中运行的程序：

```
$ docker run --rm -p 3000:3000 simple-node
```

此时，我们的`simple-node`映像存储在本地 Docker 注册表中，该映像是在构建时创建的，并且只能被单台机器访问。Docker 的真正强大之处在于能够在成千上万台机器和更广泛的 Docker 社区之间共享映像。

## 优化映像大小

当人们开始尝试使用容器映像进行实验时，会遇到几个容易导致映像过大的问题。首先要记住的是，系统中后续层次删除的文件实际上仍然存在于映像中；它们只是不可访问。考虑以下情况：

```
.
└── layer A: contains a large file named 'BigFile'
    └── layer B: removes 'BigFile'
        └── layer C: builds on B by adding a static binary
```

您可能认为*BigFile*不再存在于这个图像中。毕竟，当您运行图像时，它是不可访问的。但事实上，它仍然存在于层 A 中，这意味着每当您推送或拉取图像时，*BigFile*仍然通过网络传输，即使您无法再访问它。

另一个陷阱围绕图像缓存和构建。请记住，每一层都是独立于其下层的增量。每当您更改一层时，它会改变其后的每一层。更改前面的层意味着它们需要重新构建、重新推送和重新拉取，以便将您的图像部署到开发环境。

要更全面地理解这一点，请考虑两个图像：

```
.
└── layer A: contains a base OS
    └── layer B: adds source code server.js
        └── layer C: installs the 'node' package
```

对比：

```
.
└── layer A: contains a base OS
    └── layer B: installs the 'node' package
        └── layer C: adds source code server.js
```

看起来显而易见，这两个图像的行为将是相同的，确实在它们第一次被拉取时是这样。然而，考虑一下*server.js*发生变化后会发生什么。在第二种情况下，只需要拉取或推送这个变化，但在第一种情况下，需要拉取和推送*server.js*和提供`node`包的层，因为`node`层依赖于*server.js*层。一般来说，您希望将图像层按从最不可能改变到最可能改变的顺序排列，以优化推送和拉取的图像大小。这就是为什么在示例 2-4 中，我们首先复制*package.json*文件并安装依赖项，然后再复制其余的程序文件。开发人员更频繁地会更新和改变程序文件，而不是依赖项。

## 图像安全性

在安全性方面，没有捷径可走。在构建最终将在生产 Kubernetes 集群中运行的图像时，请务必遵循最佳实践来打包和分发应用程序。例如，不要在容器中嵌入密码——这不仅限于最终层，还包括图像中的任何层。容器层引入的一个违反直觉的问题之一是，在一个层中删除文件并不会从前面的层中删除该文件。它仍然占用空间，并且可以被具备正确工具的任何人访问——一位有进取心的攻击者可以简单地创建一个仅包含包含密码的层的图像。

机密和图像*绝对不能*混合在一起。如果这样做，您将会被黑客攻击，并给整个公司或部门带来耻辱。我们都希望有朝一日能上电视，但有更好的方法去做这件事。

另外，由于容器图像专注于运行单个应用程序，最佳实践是尽量减少容器图像中的文件。图像中每增加一个库都会为您的应用程序提供一个潜在的漏洞向量。根据语言的不同，您可以通过非常严格的依赖关系实现非常小的图像。这个较小的集合确保您的图像不会受到它永远不会使用的库的漏洞的影响。

# 多阶段图像构建

意外构建大型镜像的最常见方法之一是将实际的程序编译作为构建应用程序容器镜像的一部分。作为镜像构建的一部分进行代码编译看起来很自然，也是从程序构建容器镜像的最简单方式。但这样做的问题在于，会留下所有不必要的开发工具，这些工具通常非常庞大，仍然保存在镜像内部，从而减慢部署速度。

为了解决这个问题，Docker 引入了 *多阶段构建*。使用多阶段构建，Docker 文件不再只生成一个镜像，实际上可以生成多个镜像。每个镜像被视为一个阶段。可以从前面的阶段复制构件到当前阶段。

为了具体说明这一点，我们将看一下如何构建我们的示例应用程序 `kuard`。这是一个稍微复杂的应用程序，涉及到一个有自己构建过程的 *React.js* 前端，然后嵌入到一个 Go 程序中。该 Go 程序运行一个后端 API 服务器，*React.js* 前端与其交互。

一个简单的 Dockerfile 可能看起来像这样：

```
FROM golang:1.17-alpine

# Install Node and NPM
RUN apk update && apk upgrade && apk add --no-cache git nodejs bash npm

# Get dependencies for Go part of build
RUN go get -u github.com/jteeuwen/go-bindata/...
RUN go get github.com/tools/godep
RUN go get github.com/kubernetes-up-and-running/kuard

WORKDIR /go/src/github.com/kubernetes-up-and-running/kuard

# Copy all sources in
COPY . .

# This is a set of variables that the build script expects
ENV VERBOSE=0
ENV PKG=github.com/kubernetes-up-and-running/kuard
ENV ARCH=amd64
ENV VERSION=test

# Do the build. This script is part of incoming sources.
RUN build/build.sh

CMD [ "/go/bin/kuard" ]
```

这个 Dockerfile 生成一个包含静态可执行文件的容器镜像，但它还包含所有的 Go 开发工具以及构建 *React.js* 前端和应用程序源代码的工具，这两者对最终应用程序都不是必需的。整个镜像，包括所有层，总共超过 500 MB。

要查看如何使用多阶段构建来做到这一点，请查看以下多阶段 Dockerfile：

```
# STAGE 1: Build
FROM golang:1.17-alpine AS build

# Install Node and NPM
RUN apk update && apk upgrade && apk add --no-cache git nodejs bash npm

# Get dependencies for Go part of build
RUN go get -u github.com/jteeuwen/go-bindata/...
RUN go get github.com/tools/godep

WORKDIR /go/src/github.com/kubernetes-up-and-running/kuard

# Copy all sources in
COPY . .

# This is a set of variables that the build script expects
ENV VERBOSE=0
ENV PKG=github.com/kubernetes-up-and-running/kuard
ENV ARCH=amd64
ENV VERSION=test

# Do the build. Script is part of incoming sources.
RUN build/build.sh

# STAGE 2: Deployment
FROM alpine

USER nobody:nobody
COPY --from=build /go/bin/kuard /kuard

CMD [ "/kuard" ]
```

这个 Dockerfile 生成两个镜像。第一个是 *构建* 镜像，其中包含 Go 编译器、*React.js* 工具链和程序源代码。第二个是 *部署* 镜像，只包含编译后的二进制文件。使用多阶段构建构建容器镜像可以将最终镜像大小减少数百兆字节，从而显著加快部署时间，因为通常情况下，部署延迟取决于网络性能。从这个 Dockerfile 生成的最终镜像大约为 20 MB。

这些脚本位于 [GitHub](https://oreil.ly/6c9MX) 上 `kuard` 仓库中，您可以使用以下命令构建和运行此镜像：

```
# Note: if you are running on Windows you may need to fix line-endings using:
# --config core.autocrlf=input
$ git clone https://github.com/kubernetes-up-and-running/kuard
$ cd kuard
$ docker build -t kuard .
$ docker run --rm -p 8080:8080 kuard
```

# 在远程注册表中存储镜像

如果一个容器镜像只能在单台机器上使用，那有什么用呢？

Kubernetes 依赖于 Pod 清单中描述的镜像在集群中的每台机器上都可用的事实。将此镜像传输到集群中每台机器的一种选项是在每台机器上导出 `kuard` 镜像并导入它们。我们认为没有比这种方式更烦人的事情了。手动导入和导出 Docker 镜像过程中充满了人为错误。坚决不要这样做！

Docker 社区的标准是将 Docker 镜像存储在远程注册表中。在选择 Docker 注册表时，有大量选项，您的选择将主要基于安全性和协作功能的需求。

一般来说，关于注册表，您需要做出的第一个选择是使用私有注册表还是公共注册表。公共注册表允许任何人下载存储在注册表中的镜像，而私有注册表则需要身份验证才能下载镜像。在选择公共还是私有注册表时，考虑您的用例是很有帮助的。

公共注册表非常适合与世界分享镜像，因为它们允许轻松、无需身份验证地使用容器镜像。您可以将软件作为容器镜像分发，并确信用户在任何地方都会有完全相同的体验。

相比之下，私有注册表最适合存储您服务中私有的并且您不希望外界使用的应用程序。此外，私有注册表通常提供更好的可用性和安全性保证，因为它们专门为您和您的镜像而设计，而不是全球服务。

无论如何，要推送镜像，您需要对注册表进行身份验证。通常可以使用`docker login`命令完成这一操作，尽管对于某些注册表可能有一些差异。在本书的示例中，我们将推送到 Google Cloud Platform 注册表，称为 Google Container Registry（GCR）；其他云服务商，包括 Azure 和 Amazon Web Services（AWS），也有托管的容器注册表。对于托管公开可读镜像的新用户，[Docker Hub](https://hub.docker.com)是一个很好的起点。

登录后，您可以通过在目标 Docker 注册表前置标签化`kuard`镜像。您还可以附加一个标识符，通常用于该镜像的版本或变体，用冒号（`:`）分隔：

```
$ docker tag kuard gcr.io/kuar-demo/kuard-amd64:blue
```

然后您可以推送`kuard`镜像：

```
$ docker push gcr.io/kuar-demo/kuard-amd64:blue
```

现在`kuard`镜像已经在远程注册表上可用，是时候使用 Docker 部署它了。当我们将镜像推送到 GCR 时，它被标记为公共，因此无需身份验证即可在任何地方使用。

# 容器运行时接口

Kubernetes 提供了描述应用部署的 API，但依赖于容器运行时使用目标操作系统的特定容器 API 来设置应用程序容器。在 Linux 系统上，这意味着配置 cgroups 和命名空间。这种容器运行时的接口由容器运行时接口（Container Runtime Interface，CRI）标准定义。CRI API 由许多不同的程序实现，包括 Docker 构建的`containerd-cri`和 Red Hat 贡献的`cri-o`实现。安装 Docker 工具时，也会安装并使用`containerd`运行时由 Docker 守护进程使用。

从 Kubernetes 1.25 版本开始，只有支持 CRI 的容器运行时才能与 Kubernetes 兼容。幸运的是，托管 Kubernetes 提供商已经使用户在托管 Kubernetes 上的过渡几乎自动化。

## 使用 Docker 运行容器

在 Kubernetes 中，通常通过每个节点上的一个叫做 *kubelet* 的守护进程启动容器；然而，使用 Docker 命令行工具更容易开始使用容器。Docker CLI 工具可用于部署容器。要从 `gcr.io/kuar-demo/kuard-amd64:blue` 镜像部署容器，请运行以下命令：

```
$ docker run -d --name kuard \
  --publish 8080:8080 \
  gcr.io/kuar-demo/kuard-amd64:blue
```

此命令启动 `kuard` 容器，并将本地机器上的端口 8080 映射到容器中的 8080 端口。`--publish` 选项可以缩写为 `-p`。这种转发是必要的，因为每个容器都有自己的 IP 地址，所以在容器内部监听 *localhost* 不会导致您在本机上监听。如果没有端口转发，连接将无法访问您的机器。`-d` 选项指定此操作应在后台（守护进程）运行，而 `--name kuard` 给容器一个友好的名称。

## 探索 kuard 应用程序

`kuard` 提供了一个简单的 Web 接口，您可以通过浏览器加载 *http://localhost:3000* 或通过命令行来访问：

```
$ curl http://localhost:8080
```

`kuard` 还暴露了许多我们将在本书后续部分探索的有趣功能。

## 限制资源使用

Docker 通过暴露 Linux 内核提供的底层 cgroup 技术，使应用程序能够使用更少的资源。Kubernetes 也利用这些能力来限制每个 Pod 使用的资源。

### 限制内存资源

在容器内运行应用程序的一个关键好处是能够限制资源利用。这允许多个应用程序在同一台硬件上共存，并确保公平使用。

要限制 `kuard` 的内存为 200 MB 和交换空间为 1 GB，请使用 `docker run` 命令的 `--memory` 和 `--memory-swap` 标志。

停止并删除当前的 `kuard` 容器：

```
$ docker stop kuard
$ docker rm kuard
```

然后，使用适当的标志启动另一个 `kuard` 容器以限制内存使用：

```
$ docker run -d --name kuard \
  --publish 8080:8080 \
  --memory 200m \
  --memory-swap 1G \
  gcr.io/kuar-demo/kuard-amd64:blue
```

如果容器中的程序使用了过多的内存，它将被终止。

### 限制 CPU 资源

机器上的另一个关键资源是 CPU。使用 `docker run` 命令的 `--cpu-shares` 标志来限制 CPU 利用率：

```
$ docker run -d --name kuard \
  --publish 8080:8080 \
  --memory 200m \
  --memory-swap 1G \
  --cpu-shares 1024 \
  gcr.io/kuar-demo/kuard-amd64:blue
```

# 清理

构建镜像完成后，您可以使用 `docker rmi` 命令删除它：

```
docker rmi <*tag-name*>
```

或者：

```
docker rmi <*image-id*>
```

镜像可以通过它们的标签名称（例如 `gcr.io/kuar-demo/kuard-amd64:blue`）或它们的镜像 ID 来删除。与 `docker` 工具中的所有 ID 值一样，只要保持唯一性，镜像 ID 可以缩短。通常只需要三到四个字符的 ID。

需要注意的是，除非您明确删除镜像，否则它将永远存在于您的系统中，*即使*您使用相同名称构建新镜像。构建此新镜像仅将标签移至新镜像；它不会删除或替换旧镜像。

因此，在创建新镜像时进行迭代时，您通常会创建许多不必要占用计算机空间的不同镜像。要查看当前计算机上的镜像，可以使用`docker images`命令。然后可以删除不再使用的标签。

Docker 提供了一个名为`docker system prune`的工具用于进行一般清理。这将删除所有停止的容器、所有未标记的镜像以及作为构建过程的一部分缓存的所有未使用的镜像层。请谨慎使用。

更复杂一点的方法是设置一个`cron`任务来运行镜像垃圾收集器。例如，您可以轻松地将`docker system prune`设置为定期的`cron`任务，每天一次或每次，具体取决于您创建的图像数量。

# 概要

应用程序容器为应用程序提供了一个清晰的抽象，并且当打包为 Docker 镜像格式时，应用程序变得易于构建、部署和分发。容器还在同一台机器上运行的应用程序之间提供隔离，有助于避免依赖冲突。

在接下来的章节中，我们将看到挂载外部目录的能力意味着我们不仅可以在容器中运行无状态应用程序，还可以运行生成大量数据的应用程序，例如 MySQL 和其他应用。
