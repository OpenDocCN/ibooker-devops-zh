# 第四章：使用 Docker 镜像

每个 Linux 容器都基于一个镜像。镜像是被重新组合成运行容器的基本定义，就像启动虚拟磁盘时会形成一个虚拟机一样。Docker 或者 [开放容器倡议（OCI）](https://opencontainers.org) 的镜像提供了你将使用 Docker 部署和运行的所有内容的基础。要启动一个容器，你必须下载一个公共镜像或者创建自己的镜像。你可以将镜像视为一个主要表示容器文件系统的单个资产。然而，实际上，每个镜像由一个或多个链接的文件系统层组成，这些层通常直接一对一地映射到创建该镜像所使用的每个构建步骤。

因为镜像是从单独的层构建起来的，它们对 Linux 内核提出了特殊要求，这需要提供 Docker 运行存储后端所需的驱动程序。对于镜像管理，Docker 在很大程度上依赖于这个存储后端，它与底层的 Linux 文件系统通信，以构建和管理组合成一个可用镜像的多个层。支持的主要存储后端包括以下内容：

+   [Overlay2](https://oreil.ly/r4JHY)^(1)

+   [B-Tree 文件系统（Btrfs）](https://btrfs.wiki.kernel.org/index.php/Main_Page)

+   [Device Mapper](https://www.sourceware.org/dm)

每个存储后端都为镜像管理提供了快速的写时复制（CoW）系统。我们将在 第十一章 中讨论各种后端的具体情况。目前，我们将使用默认后端并探索镜像的工作原理，因为它们几乎构成了你在 Docker 中做的所有其他工作的基础，包括以下内容：

+   构建镜像

+   将镜像上传（推送）到镜像注册表

+   从镜像注册表下载（拉取）镜像

+   从镜像创建和运行容器

# Dockerfile 的解剖

要使用默认工具创建自定义 Docker 镜像，你需要熟悉 *Dockerfile*。这个文件描述了创建镜像所需的所有步骤，通常包含在应用程序源代码库的根目录中。

典型的 *Dockerfile* 可能看起来像这里显示的这个，它创建了一个基于 Node.js 的应用程序容器：

```
FROM node:18.13.0

ARG email="anna@example.com"
LABEL "maintainer"=$email
LABEL "rating"="Five Stars" "class"="First Class"

USER root

ENV AP /data/app
ENV SCPATH /etc/supervisor/conf.d

RUN apt-get -y update

# The daemons
RUN apt-get -y install supervisor
RUN mkdir -p /var/log/supervisor

# Supervisor Configuration
COPY ./supervisord/conf.d/* $SCPATH/

# Application Code
COPY *.js* $AP/

WORKDIR $AP

RUN npm install

CMD ["supervisord", "-n"]
```

解剖这个 *Dockerfile* 将提供对控制如何组装镜像的多个可能指令的初步了解。*Dockerfile* 中的每一行都会创建一个由 Docker 存储的新镜像层。这个层包含因发出该命令而产生的所有更改。这意味着当你构建新镜像时，Docker 只需构建与之前构建不同的层：你可以重用所有未更改的层。

尽管您可以从一个普通的基础 Linux 镜像构建一个 Node 实例，但您也可以在[Docker Hub](https://registry.hub.docker.com)上探索官方的 Node 镜像。Node.js 社区维护了一系列[Docker 镜像](https://registry.hub.docker.com/_/node)和标签，让您可以快速确定可用的版本。如果您想将镜像锁定到 Node 的特定点发布版本，您可以指向类似`node:18.13.0`的内容。以下基础镜像将为您提供一个运行 Node 11.11.x 的 Ubuntu Linux 镜像：

```
FROM docker.io/node:18.13.0
```

`ARG`参数提供了一种方式，让您设置变量及其默认值，这些值仅在镜像构建过程中可用：

```
ARG email="anna@example.com"
```

为镜像和容器应用标签允许您通过键/值对添加元数据，以便以后用于搜索和识别 Docker 镜像和容器。您可以使用`docker image inspect`命令查看应用于任何镜像的标签。对于维护者标签，我们正在利用在*Dockerfile*的前一行中定义的`email`构建参数的值。这意味着每当我们构建此镜像时，此标签都可以更改：

```
LABEL "maintainer"=$email
LABEL "rating"="Five Stars" "class"="First Class"
```

默认情况下，Docker 在容器内以`root`身份运行所有进程，但您可以使用`USER`指令来更改这一点：

```
USER root
```

###### 注意

尽管容器在一定程度上提供了与底层操作系统的隔离，但它们仍在主机内核上运行。由于潜在的安全风险，生产容器几乎总是应以非特权用户的身份运行。

与`ARG`指令不同，`ENV`指令允许您设置 shell 变量，这些变量可以被您运行的应用程序用于配置，同时也可在构建过程中使用。`ENV`和`ARG`指令可用于简化*Dockerfile*并帮助保持 DRY（不要重复自己）：

```
ENV AP /data/app
ENV SCPATH /etc/supervisor/conf.d
```

在下面的代码中，您将使用一系列`RUN`指令来启动和创建所需的文件结构，并安装一些必需的软件依赖：

```
RUN apt-get -y update

# The daemons
RUN apt-get -y install supervisor
RUN mkdir -p /var/log/supervisor
```

###### 警告

虽然我们在这里为了简单起见演示，但不建议在应用程序的*Dockerfile*中运行类似`apt-get -y update`或`dnf -y update`这样的命令。这是因为每次运行构建时都需要爬取存储库索引，这意味着您的构建不能保证重复性，因为软件包版本可能在构建之间发生变化。相反，考虑基于已应用这些更新并且版本处于已知状态的另一个镜像构建您的应用程序镜像。这样会更快速和更可重复。

`COPY` 指令用于将文件从本地文件系统复制到镜像中。最常见的情况是将应用程序代码和任何所需的支持文件包括在内。由于 `COPY` 将文件复制到镜像中，一旦构建完成，您就不再需要访问本地文件系统来访问这些文件。您还将开始使用您在上一节中定义的构建变量，以节省一些工作并帮助您避免打字错误：

```
# Supervisor Configuration
COPY ./supervisord/conf.d/* $SCPATH/

# Application Code
COPY *.js* $AP/
```

###### 提示

每个指令都会创建一个新的 Docker 镜像层，因此通常将一些逻辑上分组的命令组合成一行是有意义的。甚至可以在 *Dockerfile* 中仅使用两个命令结合 `COPY` 指令和 `RUN` 指令将复杂脚本复制到镜像中并执行该脚本。

使用 `WORKDIR` 指令，您可以更改镜像中的工作目录，以便为其余的构建指令和默认进程启动设置任何生成的容器：

```
WORKDIR $AP

RUN npm install
```

###### 警告

*Dockerfile* 中的命令顺序对持续构建时间有很大影响。您应该尝试将每次构建都会改变的事物放在靠近底部的位置。这意味着添加您的代码和类似步骤应该推迟到最后。当您重建镜像时，每个引入变更的第一个层之后的所有层都需要重新构建。

最后，使用 `CMD` 指令定义启动容器内所需运行的进程的命令：

```
CMD ["supervisord", "-n"]
```

###### 注意

虽然没有硬性规定，但通常认为在容器内只运行单个进程是最佳实践。核心思想是容器应提供单一功能，以便轻松地在体系结构中水平扩展各个功能。在示例中，您正在使用 `supervisord` 作为进程管理器来提高容器内的节点应用程序的可靠性，并确保其保持运行状态。这在开发过程中也很有用，因此您可以重新启动服务而不必重新启动整个容器。

您也可以通过在 `docker container run` 命令行参数中使用 `--init` 命令达到类似效果，我们在 “控制进程” 中进行了讨论。

# 构建镜像

要构建您的第一个镜像，请继续克隆包含名为 *docker-node-hello* 的示例应用程序的 Git 存储库，如下所示：^(2)

```
$ git clone https://github.com/spkane/docker-node-hello.git \
    --config core.autocrlf=input
Cloning into 'docker-node-hello'…
remote: Counting objects: 41, done.
remote: Total 41 (delta 0), reused 0 (delta 0), pack-reused 41
Unpacking objects: 100% (41/41), done.

$ cd docker-node-hello
```

###### 注意

Git 经常安装在 Linux 和 macOS 系统上，但如果您尚未安装 Git，可以从 [*git-scm.com*](https://git-scm.com/downloads) 下载一个简单的安装程序。

我们使用的 `--config core.autocrlf=input` 选项有助于确保行尾不会意外地从预期的 Linux 标准更改。

这将下载一个名为*docker-node-hello*的目录中的工作*Dockerfile*和相关的源代码文件。 如果您忽略 Git 仓库目录查看内容，您应该会看到以下内容：

```
$ tree -a -I .git
.
├── .dockerignore
├── .gitignore
├── Dockerfile
├── index.js
├── package.json
└── supervisord
 └── conf.d
 ├── node.conf
 └── supervisord.conf
```

让我们回顾一下仓库中最相关的文件。

*Dockerfile* 应该与您刚刚审查过的文件相同。

*.dockerignore*文件允许您定义在构建镜像时不想上传到 Docker 主机的文件和目录。 在这个例子中，*.dockerignore*文件包含以下内容：

```
.git
```

这条命令指示`docker image build`排除了包含整个源代码库的*.git*目录。 其余文件反映了当前检出分支上源代码的当前状态。 您不需要*.git*目录的内容来构建 Docker 镜像，而且由于随着时间的推移它可能会变得很大，您不希望每次构建时都浪费时间复制它。 *package.json*定义了 Node.js 应用程序并列出了它所依赖的任何依赖项。 *index.js*是应用程序的主要源代码。

*supervisord*目录包含用于启动和监视应用程序的`supervisord`的配置文件。

###### 注意

在这个示例中使用[`supervisord`](http://supervisord.org)来监视应用程序可能有些过度，但它旨在为您提供一些有关在容器中使用一些技术来更好地控制应用程序及其运行状态的洞察。

正如我们在第三章中讨论的那样，您需要确保 Docker 服务器正在运行，并且客户端已正确设置以与其进行通信，然后才能构建 Docker 镜像。 假设一切正常运行，您应该能够通过运行即将提供的命令启动一个新的构建，该命令将基于当前目录中的文件构建并标记一个镜像。

下面的输出中标识的每个步骤直接映射到*Dockerfile*中的一行，并且每个步骤都基于前一个步骤创建一个新的镜像层。 第一次构建将需要几分钟，因为您需要下载基础的 node 镜像。 除非发布了我们基础镜像标签的新版本，否则后续构建应该会快得多。

###### 注意

下面的输出来自 Docker 中包含的新 BuildKit。 如果您看到明显不同的输出，则您可能仍在使用旧的镜像构建代码。

您可以通过将`DOCKER_BUILDKIT`环境变量设置为`1`来在您的环境中启用 BuildKit。

您可以在[Docker 网站](https://docs.docker.com/build/buildkit)上找到更多详细信息。

在 `build` 命令的末尾，你会注意到一个句点。这是指构建上下文，告诉 Docker 应该上传哪些文件到服务器，以便它可以构建我们的镜像。在许多情况下，你将只看到 `build` 命令的末尾有一个 `.`，因为一个句点代表当前目录。这个构建上下文就是 *.dockerignore* 文件正在过滤的内容，以便我们不会上传更多不必要的文件。

###### 提示

Docker 假设 *Dockerfile* 在当前目录中，但如果不在，可以直接使用 `-f` 参数指向它。

让我们来运行构建：

```
$ docker image build -t example/docker-node-hello:latest .

 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 37B
 => [internal] load .dockerignore
 => => transferring context: 34B
 => [internal] load metadata for docker.io/library/node:18.13.0
 => CACHED [1/8] FROM docker.io/library/node:18.13.0@19a9713dbaf3a3899ad…
 => [internal] load build context
 => => transferring context: 233B
 => [2/8] RUN apt-get -y update
 => [3/8] RUN apt-get -y install supervisor
 => [4/8] RUN mkdir -p /var/log/supervisor
 => [5/8] COPY ./supervisord/conf.d/* /etc/supervisor/conf.d/
 => [6/8] COPY *.js* /data/app/
 => [7/8] WORKDIR /data/app
 => [8/8] RUN npm install
 => exporting to image
 => => exporting layers
 => => writing image sha256:991844271ca5b984939ab49d81b24d4d53137f04a1bd…
 => => naming to docker.io/example/docker-node-hello:latest
```

###### 提示

为了提高构建速度，当 Docker 认为安全时会使用本地缓存。这有时可能会导致意外问题，因为它并不总是注意到某些更低层发生了变化。在前面的输出中，你会注意到像 `⇒ [2/8] RUN apt-get -y update` 这样的行。如果你看到 `⇒ CACHED [2/8] RUN apt-get -y update`，你就知道 Docker 决定使用缓存。你可以通过在 `docker image build` 命令中使用 `--no-cache` 参数来禁用构建时的缓存。

如果你在一个同时用于其他进程的系统上构建你的 Docker 镜像，可以通过使用我们将在第五章中讨论的许多相同的 cgroup 方法来限制你的构建可用资源。你可以在[官方文档](https://docs.docker.com/engine/reference/commandline/image_build)中找到关于 `docker image build` 参数的详细文档。

###### 提示

使用 `docker image build` 在功能上与使用 `docker build` 是相同的。

如果在正确运行构建时遇到任何问题，你可能需要跳过并阅读本章中的 “多阶段构建” 和 “故障排除破损构建” 部分。

# 运行你的镜像

一旦成功构建了镜像，你可以使用以下命令在你的 Docker 主机上运行它：

```
$ docker container run --rm -d -p 8080:8080 example/docker-node-hello:latest
```

这个命令告诉 Docker 从带有 `example/docker-node-hello:latest` 标签的镜像中在后台创建一个运行中的容器，并将容器中的端口 8080 映射到 Docker 主机的端口 8080。如果一切顺利，新的 Node.js 应用程序应该在主机上的容器中运行。你可以通过运行 `docker container ls` 来验证这一点。要查看运行中的应用程序，请打开一个网页浏览器并指向 Docker 主机上的端口 8080。通常可以通过检查带有星号标记的 `docker context list` 条目或检查 `DOCKER_HOST` 环境变量的值来确定 Docker 主机的 IP 地址。如果 `DOCKER ENDPOINT` 设置为 Unix 套接字，则 IP 地址很可能是 `127.0.0.1`：

```
$ docker context list
NAME      TYPE … DOCKER ENDPOINT             …
default * moby … unix:///var/run/docker.sock …
…
```

获取 IP 地址，并输入类似[*http://127.0.0.1:8080/*](http://127.0.0.1:8080/)（或者如果它与此不同，则输入您的远程 Docker 地址）到您的网络浏览器地址栏，或者使用类似`curl`的命令行工具。您应该看到以下文本：

```
Hello World. Wish you were here.
```

## 构建参数

如果你检查我们构建的镜像，你将看到`maintainer`标签被设置为`anna@example.com`：

```
$ docker image inspect \
  example/docker-node-hello:latest | grep maintainer
 "maintainer": "anna@example.com",
```

如果我们想要更改`maintainer`标签，我们可以简单地重新运行构建，并通过`--build-arg`命令行参数提供`email` `ARG`的新值，就像这样：

```
$ docker image build --build-arg email=me@example.com \
    -t example/docker-node-hello:latest .

…
 => => naming to docker.io/example/docker-node-hello:latest
```

构建完成后，我们可以通过重新检查新镜像来检查结果：

```
$ docker image inspect \
  example/docker-node-hello:latest | grep maintainer
 "maintainer": "me@example.com",
```

`ARG`和`ENV`指令可以帮助使*Dockerfile*非常灵活，同时避免许多难以保持更新的重复值。

## 作为配置的环境变量

如果你阅读*index.js*文件，你会注意到文件的一部分引用了变量`$WHO`，该应用程序用于确定要向谁说 Hello：

```
var DEFAULT_WHO = "World";
var WHO = process.env.WHO || DEFAULT_WHO;

app.get('/', function (req, res) {
  res.send('Hello ' + WHO + '. Wish you were here.\n');
});
```

让我们快速介绍如何在启动应用程序时通过传递环境变量来配置此应用程序。首先，你需要使用两个命令停止现有容器。第一个命令将为你提供容器 ID，你需要在第二个命令中使用它：

```
$ docker container ls
CONTAINER ID  IMAGE                             STATUS       …
b7145e06083f  example/centos-node-hello:latest  Up 4 minutes …
```

###### 注意

你可以通过使用[Go 模板](https://developer.hashicorp.com/nomad/tutorials/templates/go-template-syntax)格式化`docker container ls`的输出，以便只看到你关心的信息。在上述示例中，你可能决定运行类似`docker container ls --format "table {{.ID}}\t{{.Image}}\t{{.Status}}"`来限制输出到你关心的三个字段。此外，运行`docker container ls --quiet`且没有格式选项将仅限制输出到容器 ID。

然后，使用前面输出的容器 ID，您可以输入以下命令来停止运行的容器：

```
$ docker container stop b7145e06083f
b7145e06083f
```

###### 提示

使用`docker container ls`功能上等同于使用`docker container list`、`docker container ps`或`docker ps`。

使用`docker container stop`也等同于使用`docker stop`。

在前述`docker container run`命令中添加单个`--env`参数后，你可以重新启动容器：

```
$ docker container run --rm -d \
    --publish mode=ingress,published=8080,target=8080 \
    --env WHO="Sean and Karl" \
    example/docker-node-hello:latest
```

如果重新加载你的网络浏览器，你应该看到网页上的文本现在如下所示：

```
Hello Sean and Karl. Wish you were here.
```

###### 注意

如果你希望，你可以将前述的`docker`命令简化为以下命令：

```
$ docker container run --rm -d -p 8080:8080 \
    -e WHO="Sean and Karl" \
    example/docker-node-hello:latest
```

你可以通过使用`docker container stop`并传递正确的容器 ID 来停止此容器。

# 自定义基础镜像

基础镜像是其他 Docker 镜像将构建在其上的最底层镜像。通常情况下，这些基础镜像基于像 Ubuntu、Fedora 或 Alpine Linux 这样的 Linux 发行版的最小安装，但它们也可以小得多，只包含单个静态编译的二进制文件。对于大多数人来说，使用他们喜欢的发行版或工具的官方基础镜像是一个很好的选择。

但是，有时候构建自己的基础镜像而不是使用他人创建的镜像更可取。其中一个原因是为了在硬件、虚拟机和容器的所有部署方法中保持一致的操作系统镜像。另一个原因是大幅减小镜像大小。例如，如果你的应用程序是一个静态构建的 C 或 Go 应用程序，那就没有必要传输整个 Ubuntu 发行版了。你可能发现你只需要用于调试的工具和其他一些 Shell 命令和二进制文件。努力构建这样的镜像可能会带来更好的部署时间和更容易的应用程序分发。

在这两种方法之间的常见折衷方案是使用 Alpine Linux 构建镜像，该系统设计非常小巧，并且作为 Docker 镜像的基础非常受欢迎。为了保持发行版的小巧，Alpine Linux 基于现代轻量级的[musl 标准库](https://musl.libc.org)，而不是传统的[GNU C 库（glibc）](https://www.gnu.org/software/libc)。总体而言，这并不是一个大问题，因为许多软件包支持*musl*，但这也需要注意。对基于 Java 的应用程序和 DNS 解析影响最大。然而，由于其小巧的镜像大小，Alpine Linux 在生产中被广泛使用。Alpine Linux 经过高度优化，其原因在于它默认只包含*/bin/sh*而不是*/bin/bash*。但是，如果需要的话，你也可以在 Alpine Linux 中安装*glibc 和 bash*，这在 JVM 容器的情况下经常会这样做。

在官方的 Docker 文档中，有一些关于如何在各种[Linux 发行版](https://dockr.ly/2N1FZcU)上构建基础镜像的良好信息。

# 存储图像

现在你已经创建了一个令你满意的 Docker 镜像，你会希望把它存储在某个地方，以便任何你想要部署它的 Docker 主机都能轻松访问它。这也是构建镜像和将其存储在某处以便未来部署的正常交接点。通常情况下，你不会在生产服务器上构建镜像然后运行它们。这个过程在我们讨论应用程序部署团队之间的交接时有描述过。通常情况下，部署是从存储库中拉取镜像并在一个或多个 Linux 服务器上运行它的过程。有几种方法可以将您的镜像存储到一个中央存储库中以便轻松检索。

## 公共注册表

Docker 为公共镜像提供了一个 [镜像注册表](https://registry.hub.docker.com)，社区希望共享的。这些包括用于 Linux 发行版的官方镜像、即用型的 WordPress 容器等。

如果您有可以在互联网上发布的镜像，最佳位置是公共注册表，比如 [Docker Hub](https://hub.docker.com)。但是也有其他选择。在核心 Docker 工具首次获得流行之时，Docker Hub 并不存在。为填补社区中的这个明显空白，创建了 [Quay.io](https://quay.io)。此后，Quay.io 经历了几次收购，现在由 Red Hat 拥有。谷歌等云供应商和 GitHub 等 SaaS 公司也有自己的注册表服务。这里我们将仅讨论其中的两个。

Docker Hub 和 Quay.io 都提供了集中式的 Docker 镜像注册表，可以从互联网的任何地方访问，并提供了存储私有镜像的方法，除了公共镜像。两者都有友好的用户界面，并具有分离团队访问权限和管理用户的能力。它们也都为私有 SaaS 托管提供了合理的商业选项，类似于 GitHub 在其系统上销售私有注册表。如果您对 Docker 感兴趣，但还没有发布足够的代码以需要内部托管解决方案，这可能是正确的第一步。

对于大量使用 Docker 的公司来说，这些注册表的最大缺点之一是它们不在部署应用程序的网络内部。这意味着每个部署的每个层可能需要通过互联网拉取。互联网的延迟对软件部署有着非常实际的影响，影响这些注册表的故障可能会对公司的顺利部署能力产生非常不利的影响。通过良好的图像设计可以减轻这种影响，其中您制作的每一层都很薄，便于在互联网上移动。

## 私有注册表

许多公司考虑的另一种选择是内部托管某种类型的 Docker 镜像注册表，它可以与 Docker 客户端交互，支持推送、拉取和搜索镜像。开源项目 [Distribution](https://github.com/distribution/distribution) 提供了大多数其他注册表构建在其上的基本功能。

在私有注册表领域的其他强大竞争者包括 [Harbor](https://goharbor.io) 和 [Red Hat Quay](https://www.redhat.com/en/technologies/cloud-computing/quay)。除了基本的 Docker 注册表功能外，这些产品还具有坚实的 GUI 界面和许多附加功能，如图像验证。

## 认证到注册表

与存储容器镜像的注册表进行通信是使用 Docker 的日常生活的一部分。对于许多注册表来说，这意味着您需要进行身份验证才能访问镜像。但 Docker 还尝试使自动化变得更容易，以便在您请求像下载私有镜像这样的事物时，它可以存储您的登录信息并代表您使用。默认情况下，Docker 假设注册表将是由 Docker，Inc.托管的公共存储库 Docker Hub。

###### 提示

尽管有点更高级，但值得注意的是，您还可以配置 Docker 守护程序以使用[自定义注册表镜像](https://oreil.ly/16Kns)^(3)或[拉取通过镜像缓存](https://oreil.ly/2Am1f).^(4)

### 创建一个 Docker Hub 帐户

对于这些示例，您将在 Docker Hub 上创建一个帐户。您不需要帐户来下载公开共享的镜像，但为了避免速率限制并上传任何构建的容器，您需要登录。

要创建您的帐户，请使用您选择的 Web 浏览器导航到[Docker Hub](https://hub.docker.com)。

从这里，您可以通过现有帐户登录或根据您的电子邮件地址创建新的登录。在创建帐户时，Docker Hub 会向您在注册时提供的地址发送验证电子邮件。您应立即登录您的电子邮件帐户，并点击邮件中的验证链接以完成验证过程。

此时，您已经创建了一个公共注册表，可以向其上传新镜像。在您的个人资料图片下的[帐户设置](https://hub.docker.com/settings/default-privacy)选项中有一个`默认隐私`部分，允许您将您的注册表默认可见性更改为`私有`，如果这是您需要的。

###### 警告

为了更好的安全性，您应该使用[有限权限个人访问令牌](https://docs.docker.com/go/access-tokens)创建并登录到 Docker Hub。

### 登录到注册表

现在让我们使用我们的帐户登录到 Docker Hub 注册表：

```
$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you
don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: <hub_username>
Password: <hub_password/token>
Login Succeeded
```

###### 注意

命令`docker login`功能上等同于`docker login docker.io`。

当您从服务器获得`登录成功`时，您就知道您可以从注册表中拉取镜像了。但是幕后发生了什么？事实证明，Docker 已经在您的主目录中为您写了一个点文件来缓存此信息。权限设置为 0600，以防其他用户读取您的凭据。您可以使用类似以下的内容检查文件：

```
$ ls -la ${HOME}/.docker/config.json
-rw-------@ 1 …  158 Dec 24 10:37 /Users/someuser/.docker/config.json

$ cat ${HOME}/.docker/config.json
```

在 Linux 上，您将看到类似于这样的东西：

```
{
    "auths": {
    "https://index.docker.io/v1/": {
      "auth":"cmVsaEXamPL3hElRmFCOUE=",
      "email":"someuser@example.com"
    }
  }
}
```

###### 注意

Docker 不断发展，并且已经添加了对许多操作系统本地秘密管理系统的支持，例如 macOS Keychain 或 Windows Credential Manager。因此，您的*config.json*文件可能与示例大不相同。还有一组不同平台的[凭证管理器](https://github.com/docker/docker-credential-helpers)，可以在这里为您简化生活。

###### 警告

Docker 客户端配置文件中的 `auth` 值仅经过 base64 编码。它 *不* 加密。这通常只在多用户 Linux 系统上是一个重要问题，因为没有一个默认的系统范围的凭据管理器可以正常工作，系统上的其他特权用户很可能可以读取你的 Docker 客户端配置文件并访问这些秘密。在 Linux 上可以配置 `gpg` 或 `pass` 来加密这些文件。

在这里，你可以看到 *${HOME}/.docker/config.json* 文件中以 JSON 格式包含 `docker.io` 凭据的用户 `someuser@example.com`。此配置文件支持存储多个注册表的凭据。在这种情况下，你只有一个条目，用于 Docker Hub，但如果需要的话，你可以有更多条目。从现在开始，当注册表需要认证时，Docker 将查看 *${HOME}/.docker/config.json*，看看你是否为此主机名存储了凭据。如果是，它将提供这些凭据。你会注意到这里完全缺少一个值：时间戳。这些凭据将永久缓存，或者直到你告诉 Docker 将它们删除为止。

与登录类似，如果你不再希望缓存凭据，也可以注销注册表：

```
$ docker logout
Removing login credentials for https://index.docker.io/v1/
$ cat ${HOME}/.docker/config.json
```

```
{
  "auths": {
  }
}
```

在这里，你已经移除了缓存的凭据，它们不再被 Docker 存储。某些版本的 Docker 可能会在文件为空时甚至删除此文件。如果你尝试登录到除 Docker Hub 注册表之外的其他地方，你可以在命令行上提供主机名：

```
$ docker login someregistry.example.com
```

这将在你的 *${HOME}/.docker/config.json* 文件中添加另一个 auth 条目。

### 推送镜像到仓库

推送镜像所需的第一步是确保你已登录到打算使用的 Docker 仓库。在本例中，我们将专注于 Docker Hub，因此请确保你已使用首选凭据登录到 Docker Hub：

```
$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you
don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: <hub_username>
Password: <hub_password/token>
Login Succeeded

Logging in with your password grants your terminal complete access to
your account.
```

一旦你登录，你就可以上传一个镜像。之前，你使用命令 `docker image build -t example/docker-node-hello:latest .` 来构建 `docker-node-hello` 镜像。

实际上，Docker 客户端，以及出于兼容性原因，许多其他容器工具，实际上将 `example/docker-node-hello:latest` 解释为 `docker.io/example/docker-node-hello:latest`。这里，`docker.io` 表示镜像注册表主机名，而 `example/docker-node-hello` 是注册表内包含相关镜像的仓库。

当你在本地构建镜像时，注册表和仓库名称可以是任何你想要的。然而，当你要将你的镜像上传到真实的注册表时，你需要与登录匹配。

你可以通过运行以下命令并将 `${<myuser>}` 替换为你的 Docker Hub 用户名，轻松编辑已创建的镜像的标签：

```
$ docker image tag example/docker-node-hello:latest \
    docker.io/${<myuser>}/docker-node-hello:latest
```

如果你需要使用新的命名约定重建镜像或只是想试试，你可以在*docker-node-hello*工作目录中运行以下命令，该目录是在本章早期执行 Git 检出时生成的。

###### 注意

对于接下来的示例，你需要在所有示例中用你在 Docker Hub 创建的用户替换`${<myuser>}`。如果你使用不同的注册表，你还需要用你使用的注册表的主机名替换`docker.io`。

```
$ docker image build -t docker.io/${<myuser>}/docker-node-hello:latest .
…
```

第一次构建时会花费一些时间。如果重新构建镜像，可能会发现速度非常快。这是因为大多数，如果不是所有层已经存在于你的 Docker 服务器中，来自前一个构建。我们可以通过运行`docker image ls ${<myuser>}/docker-node-hello`快速验证我们的镜像确实在服务器上：

```
$ docker image ls ${<myuser>}/docker-node-hello
REPOSITORY                 TAG      IMAGE ID       CREATED             SIZE
myuser/docker-node-hello   latest   f683df27f02d   About an hour ago   649MB
```

###### 提示

你可以使用`docker image ls --format="table {{.ID}}\t{{.Repository}}"`这样的格式化输出`docker image ls`来使输出更加简洁。

此时，你可以使用`docker image push`命令将镜像上传到 Docker 仓库：

```
$ docker image push ${<myuser>}/docker-node-hello:latest
Using default tag: latest
The push refers to repository [docker.io/myuser/docker-node-hello]
5f3ee7afc69c: Pushed
…
5bb0785f2eee: Mounted from library/node
latest: digest: sha256:f5ceb032aec36fcacab71e468eaf0ba8a832cfc8244fbc784d0…
```

如果此镜像已上传到公共仓库，则任何人都可以通过运行`docker image pull`命令轻松下载它。

###### 提示

如果你将镜像上传到私有仓库，则用户必须使用`docker login`命令登录具有访问权限的凭据，然后才能将镜像拉取到他们的本地系统中。

```
$ docker image pull ${<myuser>}/docker-node-hello:latest
Using default tag: latest
latest: Pulling from myuser/docker-node-hello
Digest: sha256:f5ceb032aec36fcacab71e468eaf0ba8a832cfc8244fbc784d040872be041cd5
Status: Image is up to date for myuser/docker-node-hello:latest
docker.io/myuser/docker-node-hello:latest
```

### 探索 Docker Hub 中的镜像

除了简单地使用[Docker Hub 网站](https://hub.docker.com)浏览可用的镜像外，你还可以使用`docker search`命令找到可能有用的镜像。

运行`docker search node`将返回一个包含名称或描述中包含`node`关键词的镜像列表：

```
$ docker search node
NAME                     DESCRIPTION                 STARS OFFICIAL AUTOMATED
node                     Node.js is a JavaScript-ba… 12267 [OK]
mongo-express            Web-based MongoDB admin in… 1274  [OK]
nodered/node-red         Low-code programming for e… 544
nodered/node-red-docker  Deprecated - older Node-RE… 356            [OK]
circleci/node            Node.js is a JavaScript-ba… 130
kindest/node             sigs.k8s.io/kind node imag… 78
bitnami/node             Bitnami Node.js Docker Ima… 69             [OK]
cimg/node                The CircleCI Node.js Docke… 14
opendronemap/nodeodm     Automated build for NodeOD… 10             [OK]
bitnami/node-exporter    Bitnami Node Exporter Dock… 9              [OK]
appdynamics/nodejs-agent Agent for monitoring Node.… 5
wallarm/node             Wallarm: end-to-end API se… 5              [OK]
…
```

`OFFICIAL`标头告诉你该镜像是[Docker Hub 上的官方精选镜像](https://docs.docker.com/docker-hub/official_images)之一。通常意味着该镜像由维护该应用程序的公司或官方开发社区维护。`AUTOMATED`表示该镜像是通过 CI/CD 流程自动构建和上传的，通过提交到底层源代码仓库触发。官方镜像始终是自动化的。

## 运行私有注册表

符合开源社区精神，Docker 鼓励社区通过 Docker Hub 默认分享 Docker 镜像。然而，由于商业、法律、镜像保留或可靠性问题，有时这不是可行的选择。

在这些情况下，建立内部私有注册表是有意义的。设置基本注册表并不难，但是对于生产使用，您应该花时间熟悉[开源 Docker Registry（分发）](https://docs.docker.com/registry)的所有可用配置选项。

对于本示例，我们将使用 SSL 和 HTTP 基本认证创建一个非常简单的安全注册表。

首先，在我们的 Docker 服务器上创建几个目录和文件。如果您使用虚拟机或云实例来运行 Docker 服务器，则需要 SSH 到该服务器执行接下来的命令。如果您使用 Docker Desktop 或 Community Edition，则应该能够在本地系统上运行这些命令。

###### 提示

Windows 用户可能需要下载额外的工具，例如 `htppaswd`，或者修改非 Docker 命令以在本地系统上完成相同的任务。

首先让我们克隆一个包含设置简单认证 Docker 注册表所需基本文件的 Git 仓库：

```
$ git clone https://github.com/spkane/basic-registry \
  --config core.autocrlf=input
Cloning into 'basic-registry'…
remote: Counting objects: 10, done.
remote: Compressing objects: 100% (8/8), done.
remote: Total 10 (delta 0), reused 10 (delta 0), pack-reused 0
Unpacking objects: 100% (10/10), done.
```

一旦文件下载到本地，您可以切换目录并查看刚刚下载的文件：

```
$ cd basic-registry
$ ls
Dockerfile          config.yaml.sample  registry.crt.sample
README.md           htpasswd.sample     registry.key.sample
```

*Dockerfile* 简单地从 Docker Hub 的上游注册表镜像中获取，并将一些本地配置和支持文件复制到新镜像中。

用于测试时，您可以使用一些包含的示例文件，但是请*不要*在生产环境中使用这些文件。

如果您的 Docker 服务器通过 `localhost` (127.0.0.1) 可用，则可以通过简单复制每个文件来使用这些文件而无需修改，如下所示：

```
$ cp config.yaml.sample config.yaml
$ cp registry.key.sample registry.key
$ cp registry.crt.sample registry.crt
$ cp htpasswd.sample htpasswd
```

然而，如果您的 Docker 服务器位于远程 IP 地址上，则需要做一些额外的工作。

首先，将 *config.yaml.sample* 复制到 *config.yaml*：

```
$ cp config.yaml.sample config.yaml
```

然后编辑 *config.yaml* 并将 `127.0.0.1` 替换为您的 Docker 服务器的 IP 地址，以便：

```
http:
  host: https://127.0.0.1:5000
```

变成类似这样：

```
http:
  host: https://172.17.42.10:5000
```

###### 注意

使用完全合格的域名（FQDN）如 `my-registry.example.com` 创建注册表非常容易，但是为了本示例，使用 IP 地址更加简单，因为不需要 DNS。

接下来，您需要为注册表的 IP 地址创建 SSL 密钥对。

一种方法是使用以下 OpenSSL 命令。请注意，您需要在命令的这一部分 `/CN=172.17.42.10` 中设置 IP 地址，以匹配您的 Docker 服务器的 IP 地址：

```
$ openssl req -x509 -nodes -sha256 -newkey rsa:4096 \
  -keyout registry.key -out registry.crt \
  -days 14 -subj '{/CN=172.17.42.10}'
```

最后，您可以通过复制使用示例 `htpasswd` 文件：

```
$ cp htpasswd.sample htpasswd
```

或者您可以通过使用以下命令创建自己的用户名和密码对进行身份验证，替换 `${<username>}` 和 `${<password>}` 为您首选的值：

```
$ docker container run --rm --entrypoint htpasswd g \
  -Bbn ${<username>} ${<password>} > htpasswd
```

如果再次查看目录列表，现在应该是这样的：

```
$ ls
Dockerfile          config.yaml.sample  registry.crt        registry.key.sample
README.md           htpasswd            registry.crt.sample
config.yaml         htpasswd.sample     registry.key
```

如果缺少任何这些文件，请回顾前面的步骤，确保您没有遗漏任何一个文件，然后继续。

如果一切看起来正确，那么您应该准备构建和运行注册表：

```
$ docker image build -t my-registry .
$ docker container run --rm -d -p 5000:5000 --name registry my-registry
$ docker container logs registry
```

###### 提示

如果你看到类似“docker: Error response from daemon: Conflict. The container name “/registry” is already in use.”的错误，那么你需要更改前面的容器名称或删除具有该名称的现有容器。你可以通过运行 `docker container rm registry` 来删除容器。

### 测试私有仓库

现在仓库正在运行，你可以进行测试。你需要做的第一件事就是对其进行身份验证。你需要确保 `docker login` 中的 IP 地址与运行仓库的 Docker 服务器的 IP 地址匹配。

###### 注意

`myuser` 是默认用户名，`myuser-pw!` 是默认密码。如果你生成了自己的 `htpasswd`，那么这些将是你选择的任何内容。

```
$ docker login 127.0.0.1:5000
Username: <registry_username>
Password: <registry_password>
Login Succeeded
```

###### 警告

这个仓库容器有一个嵌入式 SSL 密钥，并且没有使用任何外部存储，这意味着它包含一个秘密，当你删除运行中的容器时，所有你的镜像也将被删除。这是设计上的考虑。

在生产环境中，你将希望让你的容器从一个秘密管理系统中拉取秘密，并使用某种类型的冗余外部存储，比如对象存储。如果你想在容器之间保留开发仓库镜像，你可以在 `docker container run` 命令中添加类似 `--mount type=bind,source=/tmp/registry-data,target=/var/lib/registry` 的内容，将仓库数据存储在 Docker 服务器上。

现在，让我们看看你是否可以将刚刚构建的镜像推送到你的本地私有仓库。

###### 提示

在所有这些命令中，请确保使用正确的 IP 地址来访问你的仓库。

```
$ docker image tag my-registry 127.0.0.1:5000/my-registry
$ docker image push 127.0.0.1:5000/my-registry
Using default tag: latest
The push refers to repository [127.0.0.1:5000/my-registry]
f09a0346302c: Pushed
…
4fc242d58285: Pushed
latest: digest: sha256:c374b0a721a12c41d5b298930d11e658fbd37f22dc2a0fac7d6a2…
```

然后，你可以尝试从你的仓库中拉取相同的镜像：

```
$ docker image pull 127.0.0.1:5000/my-registry
Using default tag: latest
latest: Pulling from my-registry
Digest: sha256:c374b0a721a12c41d5b298930d11e658fbd37f22dc2a0fac7d6a2ecdc0ba5490
Status: Image is up to date for 127.0.0.1:5000/my-registry:latest
127.0.0.1:5000/my-registry:latest
```

###### 提示

值得注意的是，Docker Hub 和 Docker Distribution 都暴露了一个可供查询有用信息的 API 端点。你可以通过 [官方文档](https://github.com/distribution/distribution/blob/main/docs/spec/api.md) 了解更多关于 API 的信息。

如果你没有遇到任何错误，那么你已经拥有一个可用于开发的仓库，并可以在此基础上构建生产仓库。此时，你可能希望暂时停止仓库。你可以通过运行以下命令轻松实现：

```
$ docker container stop registry
```

###### 提示

随着你对 Docker Distribution 的熟悉程度增加，你可能还想考虑探索云原生计算基金会（CNCF）的开源项目 [Harbor](https://goharbor.io)，它通过许多安全和可靠性功能扩展了 Docker Distribution。

# 优化镜像

当你花费一点时间使用 Docker 后，你很快会注意到，保持镜像大小小和构建时间快可以极大地减少构建和部署新软件版本所需的时间。在本节中，我们将讨论设计镜像时应始终牢记的一些考虑因素，以及一些可以帮助你实现这些目标的技术。

## 保持镜像小巧

在大多数现代企业中，从互联网上的远程位置下载一个 1 GB 的单个文件并不是人们经常担心的事情。在互联网上找到软件非常容易，人们通常会依赖重新下载它，而不是为未来保留本地副本。当你确实需要在单个服务器上的单个副本时，这可能是可以接受的，但当你需要在 100+ 节点上安装相同的软件并且每天部署新版本时，它很快会成为一个扩展性问题。下载这些大文件可能会迅速导致网络拥塞和更慢的部署周期，这对生产环境有真正的影响。

为了方便起见，许多 Linux 容器继承自一个包含最小 Linux 发行版的基础镜像。尽管这是一个简单的起点，但并非必需。容器只需要包含在主机内核上运行应用程序所需的文件，而不需要其他内容。最好的解释方法是探索一个非常精简的容器。

Go 是一种编译型编程语言，可以轻松生成静态编译的二进制文件。在这个例子中，我们将使用一款非常小的用 Go 编写的网络应用程序，可以在 [GitHub](https://github.com/spkane/scratch-helloworld) 找到。

让我们来试试这个应用程序，看看它的作用。运行以下命令，然后打开一个 Web 浏览器，将其指向你的 Docker 主机的 8080 端口（例如，*http://127.0.0.1:8080* 适用于 Docker Desktop 和 Community Edition）：

```
$ docker container run --rm -d -p 8080:8080 spkane/scratch-helloworld
```

如果一切顺利，你应该在 Web 浏览器中看到以下消息：“Hello World from Go in minimal Linux container.” 现在让我们来看看这个容器包含哪些文件。可以合理地假设，至少包括一个工作中的 Linux 环境和编译 Go 程序所需的所有文件，但你很快会发现情况并非如此。

在容器仍在运行时，执行以下命令来确定容器的 ID 是什么。以下命令返回你创建的最后一个容器的信息：

```
$ docker container ls -l
CONTAINER ID IMAGE                     COMMAND       CREATED           …
ddc3f61f311b spkane/scratch-helloworld "/helloworld" 4 minutes ago     …
```

然后，你可以使用之前运行命令获取的容器 ID 来将容器中的文件导出为一个 tarball，这样可以很容易地进行检查：

```
$ docker container export ddc3f61f311b -o web-app.tar
```

使用 `tar` 命令，你现在可以查看导出时容器的内容：

```
$ tar -tvf web-app.tar
-rwxr-xr-x  0 0      0           0 Jan  7 15:54 .dockerenv
drwxr-xr-x  0 0      0           0 Jan  7 15:54 dev/
-rwxr-xr-x  0 0      0           0 Jan  7 15:54 dev/console
drwxr-xr-x  0 0      0           0 Jan  7 15:54 dev/pts/
drwxr-xr-x  0 0      0           0 Jan  7 15:54 dev/shm/
drwxr-xr-x  0 0      0           0 Jan  7 15:54 etc/
-rwxr-xr-x  0 0      0           0 Jan  7 15:54 etc/hostname
-rwxr-xr-x  0 0      0           0 Jan  7 15:54 etc/hosts
lrwxrwxrwx  0 0      0           0 Jan  7 15:54 etc/mtab -> /proc/mounts
-rwxr-xr-x  0 0      0           0 Jan  7 15:54 etc/resolv.conf
-rwxr-xr-x  0 0      0     3604416 Jul  2  2014 helloworld
drwxr-xr-x  0 0      0           0 Jan  7 15:54 proc/
drwxr-xr-x  0 0      0           0 Jan  7 15:54 sys/
```

你可能会注意到这个容器里几乎没有文件，几乎所有文件的大小都是零字节。所有长度为零的文件都需要存在于每个 Linux 容器中，并在首次创建容器时自动从主机进行 [绑定挂载](https://unix.stackexchange.com/questions/198590/what-is-a-bind-mount) 到容器中。除了 *.dockerenv* 之外的所有这些文件都是内核需要正常工作的关键文件。在这个容器中唯一具有实际大小并与我们的应用程序相关的文件是静态编译的 `helloworld` 二进制文件。

这次练习的要点是，你的容器只需要包含在底层内核上运行所需的内容。其他都是不必要的。由于在故障排除时通常需要访问容器中的工作 shell，人们经常会妥协并使用像 Alpine Linux 这样非常轻量的 Linux 发行版构建他们的镜像。

###### 小贴士

如果你发现自己经常探索镜像文件，你可能想看看工具 [dive](https://github.com/wagoodman/dive)，它提供了一个良好的 CLI 接口来理解一个镜像包含的内容。

为了更深入地探讨一下，让我们再次看看同一个容器，以便我们可以深入研究底层文件系统，并将其与流行的 `alpine` 基础镜像进行比较。

尽管我们可以简单地通过运行 `docker container run -ti alpine:latest /bin/sh` 来探索 `alpine` 镜像，但我们无法对 `spkane/scratch-helloworld` 镜像进行此操作，因为它不包含 shell 或 SSH。这意味着我们无法使用 `ssh`、`nsenter` 或 `docker container exec` 来检查它，尽管在 “调试无 Shell 容器” 中讨论了一个高级技巧。此前，我们利用了 `docker container export` 命令创建了一个 *.tar* 文件，其中包含容器中所有文件的副本，但这一次我们将通过直接连接到 Docker 服务器并查看容器的文件系统来检查容器的文件系统。为此，我们需要找出镜像文件在服务器磁盘上的位置。

要确定服务器上实际存储我们文件的位置，请在 `alpine:latest` 镜像上运行 `docker image inspect`：

```
$ docker image inspect alpine:latest
```

```
[
    {
        "Id": "sha256:3fd…353",
        "RepoTags": [
            "alpine:latest"
        ],
        "RepoDigests": [
            "alpine@sha256:7b8…f8b"
        ],
…
        "GraphDriver": {
            "Data": {
                "MergedDir":
                "/var/lib/docker/overlay2/ea8…13a/merged",
                "UpperDir":
                "/var/lib/docker/overlay2/ea8…13a/diff",
                "WorkDir":
                "/var/lib/docker/overlay2/ea8…13a/work"
            },
            "Name": "overlay2"
…
        }
    }
…
]
```

然后在 `spkane/scratch-helloworld:latest` 镜像上：

```
$ docker image inspect spkane/scratch-helloworld:latest
```

```
[
    {
        "Id": "sha256:4fa…06d",
        "RepoTags": [
            "spkane/scratch-helloworld:latest"
        ],
        "RepoDigests": [
            "spkane/scratch-helloworld@sha256:46d…a1d"
        ],
…
        "GraphDriver": {
            "Data": {
                "LowerDir":
                "/var/lib/docker/overlay2/37a…84d/diff:
 /var/lib/docker/overlay2/28d…ef4/diff",
                "MergedDir":
                "/var/lib/docker/overlay2/fc9…c91/merged",
                "UpperDir":
                "/var/lib/docker/overlay2/fc9…c91/diff",
                "WorkDir":
                "/var/lib/docker/overlay2/fc9…c91/work"
            },
            "Name": "overlay2"
…
        }
    }
…
]
```

###### 注意

在这个特定的例子中，我们将使用运行在 macOS 上的 Docker Desktop，但这种一般方法也适用于大多数 Docker 服务器。但是，你可以通过最简单的方法访问你的 Docker 服务器。

由于我们使用的是 Docker Desktop，我们需要使用我们的 `nsenter` 技巧进入无 SSH 的虚拟机并探索文件系统：

```
$ docker container run --rm -it --privileged --pid=host debian \
  nsenter -t 1 -m -u -n -i sh

/ #
```

在虚拟机中，我们现在应该能够探索 `docker image inspect` 命令的 `GraphDriver` 部分中列出的各种目录。

在这个例子中，如果我们查看`alpine`镜像的第一个条目，我们会看到它标记为`MergedDir`并列出了文件夹*/var/lib/docker/overlay2/ea86408b2b15d33ee27d78ff44f82104705286221f055ba1331b58673f4b313a/merged*。如果我们列出那个目录，会得到一个错误，但从列出父目录开始，我们很快发现我们实际上想要查看*diff*目录：

```
/ # ls -lFa /var/lib/docker/overlay2/ea…3a/merged

ls: /var/lib/docker/overlay2/ea..3a/merged: No such file or directory

/ # ls -lF /var/lib/docker/overlay2/ea…3a/

total 8
drwxr-xr-x   18 root     root          4096 Mar 15 19:27 diff/
-rw-r--r--    1 root     root            26 Mar 15 19:27 link

/ # ls -lF /var/lib/docker/overlay2/ea…3a/diff

total 64
drwxr-xr-x    2 root     root          4096 Jan  9 19:37 bin/
drwxr-xr-x    2 root     root          4096 Jan  9 19:37 dev/
drwxr-xr-x   15 root     root          4096 Jan  9 19:37 etc/
drwxr-xr-x    2 root     root          4096 Jan  9 19:37 home/
drwxr-xr-x    5 root     root          4096 Jan  9 19:37 lib/
drwxr-xr-x    5 root     root          4096 Jan  9 19:37 media/
drwxr-xr-x    2 root     root          4096 Jan  9 19:37 mnt/
dr-xr-xr-x    2 root     root          4096 Jan  9 19:37 proc/
drwx------    2 root     root          4096 Jan  9 19:37 root/
drwxr-xr-x    2 root     root          4096 Jan  9 19:37 run/
drwxr-xr-x    2 root     root          4096 Jan  9 19:37 sbin/
drwxr-xr-x    2 root     root          4096 Jan  9 19:37 srv/
drwxr-xr-x    2 root     root          4096 Jan  9 19:37 sys/
drwxrwxrwt    2 root     root          4096 Jan  9 19:37 tmp/
drwxr-xr-x    7 root     root          4096 Jan  9 19:37 usr/
drwxr-xr-x   11 root     root          4096 Jan  9 19:37 var/

/ # du -sh  /var/lib/docker/overlay2/ea…3a/diff
4.5M    /var/lib/docker/overlay2/ea…3a/diff
```

现在，`alpine`碰巧是一个非常小的基础镜像，仅有 4.5 MB，非常适合在其上构建容器。但是，在我们开始构建任何内容之前，我们可以看到这个容器中仍然有很多内容。

现在，让我们来看看`spkane/scratch-helloworld`镜像中的文件。在这种情况下，我们要查看`docker image inspect`输出中`LowerDir`条目的第一个目录，你还会注意到这个目录也以名为*diff*的目录结尾：

```
/ # ls -lFh /var/lib/docker/overlay2/37…4d/diff

total 3520
-rwxr-xr-x    1 root     root        3.4M Jul  2  2014 helloworld*

/ # exit
```

你会注意到该目录中只有一个文件，大小为 3.4 MB。这个`helloworld`二进制文件是此容器中唯一的文件，并且比`alpine`镜像的起始大小小，这还没有添加任何应用程序文件。

###### 注意

你可以在 Docker 服务器上的该目录中运行`helloworld`应用程序，因为它不需要任何其他文件。但除了开发环境外，你真的不想这样做，但它可以帮助强调这类静态编译应用程序的实用性。

### 多阶段构建

在许多情况下，有一种方法可以将容器限制在更小的大小范围内：多阶段构建。这是我们建议您构建大多数生产容器的方式。您无需过多担心引入额外的资源来构建应用程序，并且仍然可以运行一个精简的生产容器。多阶段容器还鼓励在 Docker 内部进行构建，这是构建系统中重复性的一个很好的模式。

正如[scratch-helloworld 应用程序的原始作者所写](https://medium.com/@adriaandejonge/simplify-the-smallest-possible-docker-image-62c0e0d342ef)，Docker 本身对多阶段构建支持的发布使得创建小容器的过程比过去更容易。过去，要做多阶段提供几乎免费的同样事情，你需要构建一个编译代码的镜像，提取生成的二进制文件，然后构建第二个镜像，而不包含所有构建依赖项，然后将该二进制文件注入其中。这通常很难设置，并且在标准部署流水线中不总是能正常工作。

如今，你可以使用一个如下简单的*Dockerfile*来实现类似的结果：

```
# Build container
FROM docker.io/golang:alpine as builder
RUN apk update && \
    apk add git && \
    CGO_ENABLED=0 go install -a -ldflags '-s' \
    github.com/spkane/scratch-helloworld@latest

# Production container
FROM scratch
COPY --from=builder /go/bin/scratch-helloworld /helloworld
EXPOSE 8080
CMD ["/helloworld"]
```

第一件你会注意到的关于这个*Dockerfile*的事情是它看起来很像两个*Dockerfile*合并在一起。实际上确实如此，但还有更多。`FROM`命令已经扩展，以便你可以在构建阶段命名镜像。在这个例子中，第一行是`FROM docker.io/golang as builder`，意味着你想要基于`golang`镜像进行构建，并在构建图像/阶段中引用它为`builder`。

在第四行，你会看到另一个`FROM`行，这在多阶段构建引入之前是不允许的。这个`FROM`行使用一个特殊的镜像名称`scratch`，告诉 Docker 从一个空镜像开始，其中不包含任何附加文件。接下来的一行，即`COPY --from=builder /go/bin/scratch-helloworld /helloworld`，允许你直接将在*builder*镜像中构建的二进制文件复制到当前镜像中。这将确保你最终得到尽可能小的容器。

`EXPOSE 8080`行是文档，旨在通知用户服务监听的端口(s)和协议（TCP 是默认协议）。

让我们尝试构建并查看结果。首先，创建一个工作目录，然后使用你喜欢的文本编辑器，将前面示例中的内容粘贴到名为*Dockerfile*的文件中：

```
$ mkdir /tmp/multi-build
$ cd /tmp/multi-build
$ vi Dockerfile
```

###### 提示

你可以从[GitHub](https://oreil.ly/C1TSz)下载此*Dockerfile*的副本。^(5)

现在我们可以开始多阶段构建：

```
$ docker image build .
[+] Building 9.7s (7/7) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 37B
 => [internal] load .dockerignore
 => => transferring context: 2B
 => [internal] load metadata for docker.io/library/golang:alpine
 => CACHED [builder 1/2] FROM docker.io/library/golang:alpine@sha256:7cc6257…
 => [builder 2/2] RUN apk update && apk add git && CGO_ENABLED=0 go install …
 => [stage-1 1/1] COPY --from=builder /go/bin/scratch-helloworld /helloworld
 => exporting to image
 => => exporting layers
 => => writing image sha256:bb853f23418161927498b9631f54692cf11d84d6bde3af2d…
```

你会注意到输出看起来像大多数其他构建，并最终报告成功创建了我们的最终极简镜像。

###### 警告

如果你在本地系统上编译使用共享库的二进制文件，你需要小心确保这些共享库的正确版本也对容器内的进程可用。

你不限于两个阶段，实际上，这些阶段不需要彼此相关。它们将按顺序运行。例如，你可以基于公共 Go 镜像创建一个阶段，构建你的基础 Go 应用程序以提供 API，还可以基于 Angular 容器创建另一个阶段，构建你的前端 Web UI。最终阶段可以组合来自两者的输出。

###### 提示

当你开始构建更复杂的镜像时，你可能会发现仅限于单个构建上下文是有挑战性的。我们在本章末讨论的`docker-buildx`插件能够支持[多个构建上下文](https://www.docker.com/blog/dockerfiles-now-support-multiple-build-contexts)，可以支持一些非常先进的工作流程。

## 层次是累加的

直到深入挖掘镜像构建方式的更多细节之前，可能不明显的一点是构成您镜像的文件系统层是严格累加的设计。虽然您可以在先前的层中遮蔽/掩盖文件，但不能删除这些文件。实际上，这意味着您不能通过简单地删除在较早步骤中生成的文件来减小镜像的大小。

###### 注意

如果在您的 Docker 服务器上启用了实验性功能，可以使用`docker image build --squash`将一堆层压缩成单个层。这将导致所有在中间层中删除的文件实际上从最终镜像中消失，并且因此可以恢复大量浪费的空间，但这也意味着每个需要该层的系统都必须下载整个层，即使只更新了一行源代码，因此在使用这种方法时存在实际的权衡。

解释镜像层次累加性质最简单的方法是使用一些实际的例子。在一个新目录中，[下载](https://github.com/bluewhalebook/docker-up-and-running-3rd-edition/blob/main/chapter_04/additive)或创建以下文件，这将生成一个在 Fedora Linux 上运行 Apache Web 服务器的镜像：

```
FROM docker.io/fedora
RUN dnf install -y httpd
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

然后像这样构建它：

```
$ docker image build .
[+] Building 63.5s (6/6) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 130B
 => [internal] load .dockerignore
 => => transferring context: 2B
 => [internal] load metadata for docker.io/library/fedora:latest
 => [1/2] FROM docker.io/library/fedora
 => [2/2] RUN dnf install -y httpd
 => exporting to image
 => => exporting layers
 => => writing image sha256:543d61c956778b8ea3b32f1e09a9354a864467772e6…
```

让我们继续标记生成的镜像，这样您就可以在后续命令中轻松引用它：

```
$ docker image tag sha256:543d61c956778b8ea3b32f1e09a9354a864467772e6… size1
```

现在让我们用`docker image history`命令来查看我们的镜像。这个命令将为我们提供有关文件系统层和构建步骤的一些见解：

```
$ docker image history size1
IMAGE        CREATED            CREATED BY                            SIZE  …
543d61c95677 About a minute ago CMD ["/usr/sbin/httpd" "-DFOREGROU…"] 0B
<missing>    About a minute ago RUN /bin/sh -c dnf install -y httpd … 273MB
<missing>    6 weeks ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]… 0B
<missing>    6 weeks ago        /bin/sh -c #(nop) ADD file:58865512c… 163MB
<missing>    3 months ago       /bin/sh -c #(nop)  ENV DISTTAG=f36co… 0B
<missing>    15 months ago      /bin/sh -c #(nop)  LABEL maintainer=… 0B
```

您会注意到三个层未增加我们最终镜像的大小，但两个层大大增加了大小。163 MB 的层次是有道理的，因为这是包含最小 Linux 发行版的基本 Fedora 镜像；然而，273 MB 的层次令人惊讶。Apache Web 服务器不应该那么大，到底发生了什么？

如果您有使用`apk`、`apt`、`dnf`或`yum`等包管理器的经验，那么您可能知道大多数工具都严重依赖于一个包含平台上所有可安装软件包详细信息的大缓存。这个缓存占用了大量空间，并且在安装完您需要的软件包后就完全没有用了。最明显的下一步是简单地删除缓存。在 Fedora 系统上，您可以通过编辑您的*Dockerfile*来实现这一点，使其看起来像这样：

```
FROM docker.io/fedora
RUN dnf install -y httpd
RUN dnf clean all
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

然后构建、标记和检查生成的镜像：

```
$ docker image build .
[+] Building 0.5s (7/7) FINISHED
…
 => => writing image sha256:b6bf99c6e7a69a1229ef63fc086836ada20265a793cb8f2d…

$ docker image tag sha256:b6bf99c6e7a69a1229ef63fc086836ada20265a793cb8f2d17…
IMAGE        CREATED            CREATED BY                            SIZE  …
b6bf99c6e7a6 About a minute ago CMD ["/usr/sbin/httpd" "-DFOREGROU…"] 0B
<missing>    About a minute ago RUN /bin/sh -c dnf clean all # build… 71.8kB
<missing>    10 minutes ago     RUN /bin/sh -c dnf install -y httpd … 273MB
<missing>    6 weeks ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]… 0B
<missing>    6 weeks ago        /bin/sh -c #(nop) ADD file:58865512c… 163MB
<missing>    3 months ago       /bin/sh -c #(nop)  ENV DISTTAG=f36co… 0B
<missing>    15 months ago      /bin/sh -c #(nop)  LABEL maintainer=… 0B
```

如果仔细观察`docker image history`命令的输出，您会注意到已创建了一个新层，将`71.8kB`添加到镜像中，但并未减少问题层的大小。到底发生了什么？

重要的是要理解图像层的严格*增量*性质。一旦创建了一个层，就无法从中删除任何内容。这意味着您不能通过删除后续层中的文件来缩小图像中的较早层。当您删除或编辑后续层中的文件时，您只是用新层中的修改或移除版本掩盖了旧版本。这意味着您可以通过在保存层之前删除文件来使层变小的唯一方法。

处理这个问题的最常见方法是将命令串联在一个单独的*Dockerfile*行上。您可以通过利用`&&`运算符来轻松做到这一点。此运算符充当布尔`AND`语句，并且基本上可以翻译为“如果前一个命令成功运行，则运行此命令”。除此之外，您还可以利用`\`运算符，该运算符用于指示命令在换行后继续。这可以提高长命令的可读性。

拥有这些知识后，你可以像这样重写*Dockerfile*：

```
FROM docker.io/fedora
RUN dnf install -y httpd && \
    dnf clean all
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

现在，您可以重新构建图像，看看这种更改如何影响包含`http`守护程序的层的大小：

```
$ docker image build .
[+] Building 0.5s (7/7) FINISHED
…
 => => writing image sha256:14fe7924bb0b641ddf11e08d3dd56f40aff4271cad7a421fe…

$ docker image tag sha256:14fe7924bb0b641ddf11e08d3dd56f40aff4271cad7a421fe9b…
IMAGE        CREATED            CREATED BY                            SIZE   …
14fe7924bb0b About a minute ago CMD ["/usr/sbin/httpd" "-DFOREGROUN"]… 0B
<missing>    About a minute ago RUN /bin/sh -c dnf install -y httpd &… 44.8MB
<missing>    6 weeks ago        /bin/sh -c #(nop)  CMD ["/bin/bash"] … 0B
<missing>    6 weeks ago        /bin/sh -c #(nop) ADD file:58865512ca… 163MB
<missing>    3 months ago       /bin/sh -c #(nop)  ENV DISTTAG=f36con… 0B
<missing>    15 months ago      /bin/sh -c #(nop)  LABEL maintainer=C… 0B
```

在前两个示例中，涉及的层大小为 273 MB，但现在您已删除了许多不必要的文件，这些文件被添加到该层中，因此可以将该层缩小至 44.8 MB。这是非常大的空间节省，特别是考虑到在任何给定部署期间可能有多少服务器在下载镜像。

## 利用层缓存

我们将在这里介绍的最终构建技术与尽可能保持构建时间快速相关。DevOps 运动的一个重要目标是尽可能保持反馈循环紧凑。这意味着重要的是尽快发现和报告问题，以便在人们仍然完全专注于相关代码而未转向其他无关任务时进行修复。

在任何标准构建过程中，Docker 使用层缓存来尝试避免重建它已经构建的任何图像层，而这些层不包含任何显著的更改。由于这个缓存，您在*Dockerfile*中执行操作的顺序可能会对平均构建时间产生显著影响。

首先，让我们从之前的示例中获取*Dockerfile*，稍作定制，使其看起来像这样。

###### 提示

除了其他示例，您还可以在[GitHub](https://github.com/bluewhalebook/docker-up-and-running-3rd-edition/blob/main/chapter_04/cache)上找到这些文件。

```
FROM docker.io/fedora
RUN dnf install -y httpd && \
    dnf clean all
RUN mkdir -p /var/www && \
    mkdir -p /var/www/html
ADD index.html /var/www/html
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

现在，在同一个目录中，让我们还创建一个名为*index.html*的新文件，其内容如下：

```
<html>
  <head>
    <title>My custom Web Site</title>
  </head>
  <body>
    <p>Welcome to my custom Web Site</p>
  </body>
</html>
```

对于第一次测试，让我们通过使用以下命令，在完全不使用 Docker 缓存的情况下计时构建：

```
$ time docker image build --no-cache .
time docker image build --no-cache .
[+] Building 48.3s (9/9) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 238B
 => [internal] load .dockerignore
 => => transferring context: 2B
 => [internal] load metadata for docker.io/library/fedora:latest
 => CACHED [1/4] FROM docker.io/library/fedora
 => [internal] load build context
 => => transferring context: 32B
 => [2/4] RUN dnf install -y httpd &&     dnf clean all
 => [3/4] RUN mkdir -p /var/www &&     mkdir -p /var/www/html
 => [4/4] ADD index.html /var/www/html
 => exporting to image
 => => exporting layers
 => => writing image sha256:7f94d0d6492f2d2c0b8576f0f492e03334e6a535cac85576c…

real  1m21.645s
user  0m0.428s
sys   0m0.323s
```

###### 提示

Windows 用户应该能够在 WSL2 会话中运行此命令，或者使用 PowerShell [`Measure-Command`](https://oreil.ly/MQQY_)^(6) 函数替换这些示例中使用的 Unix `time` 命令。

从 `time` 命令的输出可以看出，没有使用缓存的构建大约花了 1 分钟 21 秒，只从层缓存中获取了基础镜像。如果立即之后重新构建镜像并允许 Docker 使用缓存，你会发现构建速度非常快：

```
$ time docker image build .
[+] Building 0.1s (9/9) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 37B
 => [internal] load .dockerignore
 => => transferring context: 2B
 => [internal] load metadata for docker.io/library/fedora:latest
 => [1/4] FROM docker.io/library/fedora
 => [internal] load build context
 => => transferring context: 32B
 => CACHED [2/4] RUN dnf install -y httpd &&     dnf clean all
 => CACHED [3/4] RUN mkdir -p /var/www &&     mkdir -p /var/www/html
 => CACHED [4/4] ADD index.html /var/www/html
 => exporting to image
 => => exporting layers
 => => writing image sha256:0d3aeeeeebd09606d99719e0c5197c1f3e59a843c4d7a21af…

real  0m0.416s
user  0m0.120s
sys   0m0.087s
```

由于没有任何层发生更改，并且可以完全利用所有四个构建步骤的缓存，构建只需不到一秒钟即可完成。现在，让我们对 *index.html* 文件进行小改进，使其看起来像这样：

```
<html>
  <head>
    <title>My custom Web Site</title>
  </head>
  <body>
    <div align="center">
      <p>Welcome to my custom Web Site!!!</p>
    </div>
  </body>
</html>
```

然后让我们再次计算重建时间：

```
$ time docker image build .
[+] Building 0.1s (9/9) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 37B
 => [internal] load .dockerignore
 => => transferring context: 2B
 => [internal] load metadata for docker.io/library/fedora:latest
 => [internal] load build context
 => => transferring context: 214B
 => [1/4] FROM docker.io/library/fedora
 => CACHED [2/4] RUN dnf install -y httpd &&     dnf clean all
 => CACHED [3/4] RUN mkdir -p /var/www &&     mkdir -p /var/www/html
 => [4/4] ADD index.html /var/www/html
 =>  ADD index.html /var/www/html
 => exporting to image
 => => exporting layers
 => => writing image sha256:daf792da1b6a0ae7cfb2673b29f98ef2123d666b8d14e0b74…

real  0m0.456s
user  0m0.120s
sys   0m0.068s
```

如果你仔细查看输出，你会发现大部分构建都使用了缓存。直到第 `4/4` 步，当 Docker 需要复制 *index.html* 时，缓存才会失效，需要重新创建层。因为大部分构建可以使用缓存，所以构建仍然没有超过一秒钟。

但是如果你改变 *Dockerfile* 中命令的顺序，使其看起来像这样，会发生什么：

```
FROM docker.io/fedora
RUN mkdir -p /var/www && \
    mkdir -p /var/www/html
ADD index.html /var/www/html
RUN dnf install -y httpd && \
    dnf clean all
CMD ["/usr/sbin/httpd", "-DFOREGROUND"]
```

让我们快速计时另一个不使用缓存的测试构建，以获得一个基准：

```
$ time docker image build --no-cache .
[+] Building 51.5s (9/9) FINISHED
…
 => => writing image sha256:1cc5f2c5e4a4d1cf384f6fb3a34fd4d00e7f5e7a7308d5f1f…

real  0m51.859s
user  0m0.237s
sys   0m0.159s
```

在这种情况下，构建过程需要 51 秒才能完成：因为我们使用了 `--no-cache` 参数，我们知道除了基础镜像外，没有从层缓存中获取任何内容。从第一次测试开始到现在的时间差完全是由于网络速度波动引起的，与你对 *Dockerfile* 所做的更改无关。

现在，让我们再次编辑 *index.html*，如下所示：

```
<html>
  <head>
    <title>My custom Web Site</title>
  </head>
  <body>
    <div align="center" style="font-size:180%">
      <p>Welcome to my custom Web Site</p>
    </div>
  </body>
</html>
```

现在，让我们在使用缓存的情况下再次计时镜像重建：

```
$ time docker image build .
[+] Building 43.4s (9/9) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 37B
 => [internal] load .dockerignore
 => => transferring context: 2B
 => [internal] load metadata for docker.io/library/fedora:latest
 => [1/4] FROM docker.io/library/fedora
 => [internal] load build context
 => => transferring context: 233B
 => CACHED [2/4] RUN mkdir -p /var/www &&     mkdir -p /var/www/html
 => [3/4] ADD index.html /var/www/html
 => [4/4] RUN dnf install -y httpd &&     dnf clean all
 => exporting to image
 => => exporting layers
 => => writing image sha256:9a05b2d01b5870649e0ad1d7ad68858e0667f402c8087f0b4…

real  0m43.695s
user  0m0.211s
sys   0m0.133s
```

第一次编辑 *index.html* 文件后重新构建镜像时，仅花费了 0.456 秒，但这次花费了 43.695 秒，几乎与完全不使用缓存构建整个镜像的时间相同。

这是因为你修改了 *Dockerfile*，使得 *index.html* 文件在构建过程中很早就被复制到镜像中。这样做的问题是 *index.html* 文件经常更改，并且通常会使缓存失效。另一个问题是它不必要地放在我们 *Dockerfile* 中一个非常耗时的步骤之前：安装 Apache Web 服务器。

从所有这些中获得的重要教训是顺序很重要，一般来说，你应该尽量将 *Dockerfile* 的最稳定和耗时的部分放在最前面，并尽可能晚地添加你的代码。

对于需要使用像`npm`和`bundle`等工具根据您的代码安装依赖项的项目，建议您对优化这些平台上的 Docker 构建进行一些研究。这通常包括锁定依赖版本并将它们与代码一起存储，以便不需要在每次构建时重新下载它们。

## 目录缓存

BuildKit 为镜像构建体验增加的众多功能之一是目录缓存。目录缓存是一个极其有用的工具，可以加快构建时间，同时避免将对运行时不必要的大量文件保存到您的映像中。本质上，它允许您将目录内容保存在映像内的一个特殊层中，在构建时可以将其绑定挂载，然后在生成映像快照之前卸载。通常用于处理诸如 Linux 软件安装程序（`apt`、`apk`、`dnf`等）和语言依赖管理器（`npm`、`bundler`、`pip`等）下载其数据库和归档文件的目录。

###### Tip

如果您不熟悉绑定挂载及其用途，可以在 Docker 文档中找到[绑定挂载概述](https://docs.docker.com/storage/bind-mounts)。

要使用目录缓存，您必须启用 BuildKit。在大多数情况下，这应该已经实现了，但是您可以通过设置环境变量`DOCKER_BUILDKIT=`为`1`来强制从客户端进行操作：

```
$ export DOCKER_BUILDKIT=1
```

让我们通过检出以下 git 仓库来探索目录缓存，并看看如何利用目录缓存可以显著改善连续构建的效率，同时保持生成的映像大小更小：

```
$ git clone https://github.com/spkane/open-mastermind.git \
  --config core.autocrlf=input

$ cd open-mastermind
$ cat Dockerfile
```

```
FROM python:3.9.15-slim-bullseye
RUN mkdir /app
WORKDIR /app
COPY . /app
RUN pip install -r requirements.txt
WORKDIR /app/mastermind
CMD ["python", "mastermind.py"]
```

这个代码库有一个非常通用的*Dockerfile*提交到仓库中。让我们继续看看构建此映像所需的时间，以及是否使用层缓存，还要检查生成的映像大小：

```
$ time docker build --no-cache -t docker.io/spkane/open-mastermind:latest .

[+] Building 67.5s (12/12) FINISHED
…
 => => naming to docker.io/spkane/open-mastermind:latest                  0.0s

real    0m28.934s
user    0m0.222s
sys     0m0.248s

$ docker image ls --format "{{ .Size }}" spkane/open-mastermind:latest
293MB

$ time docker build -t docker.io/spkane/open-mastermind:latest .

[+] Building 1.5s (12/12) FINISHED
…
 => => naming to docker.io/spkane/open-mastermind:latest                  0.0s

real    0m1.083s
user    0m0.098s
sys     0m0.095s
```

从此输出中，我们可以看到，如果完全利用层缓存，则构建此映像仅需不到 2 秒，而不使用层缓存则需要将近 29 秒。生成的映像总大小为 293 MB。

###### Tip

BuildKit 最终支持[修改或完全禁用输出中使用的颜色](https://github.com/moby/buildkit#color-output-controls)。对于那些在终端中使用黑色背景的用户来说，这尤其方便。您可以通过设置类似于`export BUILDKIT_COLORS=run=green:warning=yellow:error=red:cancel=cyan`这样的方式来配置这些颜色，或者通过设置`export NO_COLOR=true`来完全禁用颜色。

请注意，各种`docker`组件和第三方工具中使用的 BuildKit 版本仍在更新中，因此在每种情况下可能尚不起作用。

如果您想测试构建，请运行它：

```
$ docker container run -ti --rm docker.io/spkane/open-mastermind:latest
```

这将启动一个基于终端的开源版本的猜数字游戏（Mastermind）。屏幕上有游戏的说明，如果需要，您可以随时通过键入 Ctrl-C 退出。

由于这是一个 Python 应用程序，它使用 *requirements.txt* 文件列出应用程序所需的所有库，然后在 *Dockerfile* 中使用 `pip` 应用程序安装这些依赖项。

###### 注意

我们安装了一些不必要的依赖项，只是为了更明显地展示目录缓存的好处。

请打开 *requirements.txt* 文件并添加一行，内容为 `log-symbols`，使其看起来像这样：

```
colorama
# These are not required - but are used for demonstration purposes
pandas
flask
log-symbols
```

现在让我们重新运行构建：

```
$ time docker build -t docker.io/spkane/open-mastermind:latest \
  --progress=plain .

#1 [internal] load build definition from Dockerfile
…
#9 [5/6] RUN pip install -r requirements.txt
#9 sha256:82dbc10f1bb9fa476d93cc0d8104b76f46af8ece7991eb55393d6d72a230919e
#9 1.954 Collecting colorama
#9 2.058   Downloading colorama-0.4.5-py2.py3-none-any.whl (16 kB)
…
real    0m16.379s
user    0m0.112s
sys     0m0.082s
```

如果您查看第 `5/6` 步的完整输出，您会注意到所有依赖项都再次下载，尽管 `pip` 通常会将大多数依赖项缓存在 */root/.cache* 中。这种低效是由于构建器看到我们已经做出了影响此层的更改，因此完全重新创建了该层，因此我们失去了该缓存，尽管我们已将其存储在图像层中。

让我们继续改善这种情况。为此，我们需要利用 [BuildKit 目录缓存](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/reference.md#run---mounttypecache)，并且我们需要对 *Dockerfile* 进行一些更改，使其看起来像这样：

```
# syntax=docker/dockerfile:1
FROM python:3.9.15-slim-bullseye
RUN mkdir /app
WORKDIR /app
COPY . /app
RUN --mount=type=cache,target=/root/.cache pip install -r requirements.txt
WORKDIR /app/mastermind
CMD ["python", "mastermind.py"]
```

其中有两个重要的更改。首先，我们添加了以下行：

```
# syntax=docker/dockerfile:1
```

这告诉 Docker 我们将使用一个更新版本的 [*Dockerfile* 前端](https://hub.docker.com/r/docker/dockerfile)，这为我们提供了访问 BuildKit 的新特性。

然后我们编辑了 `RUN` 行，使其看起来像这样：

```
RUN --mount=type=cache,target=/root/.cache pip install -r requirements.txt
```

此行告诉 BuildKit 在本次构建步骤中将一个缓存层挂载到容器的 */root/.cache* 中。这将为我们达成两个目标。它将从生成的图像中删除该目录的内容，并且在连续的构建中，它也将被重新挂载并可供 `pip` 使用。

现在让我们使用这些更改对图像进行完整重建，以生成初始缓存目录内容。如果您跟随输出，您将看到 `pip` 下载了所有依赖项，正如以前一样：

```
$ time docker build --no-cache -t docker.io/spkane/open-mastermind:latest .

[+] Building 15.2s (15/15) FINISHED
…
 => => naming to docker.io/spkane/open-mastermind:latest                  0.0s
…
real    0m15.493s
user    0m0.137s
sys     0m0.096s
```

现在让我们打开 *requirements.txt* 文件并添加一行，内容为 `py-events`：

```
colorama
# These are not required - but are used for demonstration purposes
pandas
flask
log-symbols
py-events
```

这就是更改的收益所在。现在，当我们重新构建图像时，我们将看到只有 `py-events` 及其依赖项被下载；其他所有内容都使用了我们上一个构建的现有缓存，这些缓存已经被挂载到本次构建步骤的图像中：

```
$ time docker build -t docker.io/spkane/open-mastermind:latest \
  --progress=plain .

#1 [internal] load build definition from Dockerfile
…
#14 [stage-0 5/6] RUN --mount=type=cache,target=/root/.cache pip install …
#14 sha256:9bc72441fdf2ec5f5803d4d5df43dbe7bc6eeef88ebee98ed18d8dbb478270ba
#14 1.711 Collecting colorama
#14 1.714   Using cached colorama-0.4.5-py2.py3-none-any.whl (16 kB)
…
#14 2.236 Collecting py-events
#14 2.356   Downloading py_events-0.1.2-py3-none-any.whl (5.8 kB)
…
#16 DONE 1.4s

real    0m12.624s
user    0m0.180s
sys     0m0.112s

$ docker image ls --format "{{ .Size }}" spkane/open-mastermind:latest
261MB
```

由于不再需要每次重新下载所有内容，构建时间已经缩短，而且图像大小也减小了 32 MB，尽管我们已向图像添加了新的依赖项。这仅仅是因为缓存目录不再直接存储在包含应用程序的图像中。

BuildKit 和新的*Dockerfile*前端为镜像构建过程带来了许多非常有用的功能，您会希望了解这些功能。我们强烈建议您花时间阅读[参考指南](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/reference.md)，并熟悉所有可用的功能。

# 故障排除断裂构建

通常情况下，我们期望构建工作正常，特别是当我们已经将其脚本化时，但在现实世界中，事情可能会出错。让我们花点时间讨论一下您可以做些什么来排查 Docker 构建失败的问题。在本节中，我们将探讨两个选项：一个适用于使用传统构建方法的镜像构建，另一个适用于 BuildKit。

为了演示，我们将重复使用本章早期的`docker-hello-node`仓库。如果需要，您可以再次克隆它，就像这样：

```
$ git clone https://github.com/spkane/docker-node-hello.git \
    --config core.autocrlf=input
Cloning into 'docker-node-hello'…
remote: Counting objects: 41, done.
remote: Total 41 (delta 0), reused 0 (delta 0), pack-reused 41
Unpacking objects: 100% (41/41), done.

$ cd docker-node-hello
```

## 调试传统构建方法的镜像

我们需要一个病人进行接下来的一系列练习，因此让我们创建一个失败的构建。为此，请编辑*Dockerfile*，使其读取：

```
RUN apt-get -y update
```

现在读取：

```
RUN apt-get -y update-all
```

###### 警告

如果您在 Windows 上使用 PowerShell，则在运行以下`docker image build`命令之前，可能需要设置禁用 BuildKit 的环境变量，并在之后重新设置：

```
PS C:\> $env:DOCKER_BUILDKIT = 0
PS C:\> docker image build `
 -t example/docker-node-hello:latest `
 --no-cache .
PS C:\> $env:DOCKER_BUILDKIT = 1
```

如果您现在尝试构建镜像，应该会收到以下错误：

```
$ DOCKER_BUILDKIT=0 docker image build -t example/docker-node-hello:latest \
  --no-cache .

Sending build context to Docker daemon  9.216kB
Step 1/14 : FROM docker.io/node:18.13.0
 ---> 9ff38e3a6d9d
…
Step 6/14 : ENV SCPATH /etc/supervisor/conf.d
 ---> Running in e903367eaeb8
Removing intermediate container e903367eaeb8
 ---> 2a236efc3f06
Step 7/14 : RUN apt-get -y update-all
 ---> Running in c7cd72f7d9bf
E: Invalid operation update-all
The command '/bin/sh -c apt-get -y update-all' returned a non-zero code: 100
```

那么，我们如何进行排查，特别是如果我们不是在 Linux 系统上开发的情况下？这里真正的技巧是记住几乎所有 Docker 镜像都是基于其他 Docker 镜像构建的，您可以从任何镜像启动容器。虽然这一层面的含义并不明显，但如果您查看`Step 6`的输出，您将看到：

```
Step 6/14 : ENV SCPATH /etc/supervisor/conf.d
 ---> Running in e903367eaeb8
Removing intermediate container e903367eaeb8
 ---> 2a236efc3f06
```

第一行读取的`Running in e903367eaeb8`告诉您构建过程已经启动了一个新的容器，基于`Step 5`中创建的镜像。接下来的一行读取`Removing intermediate container e903367eaeb8`告诉您 Docker 现在正在移除容器，在`Step 6`的指令基础上修改了它。在这种情况下，它仅仅通过`ENV SCPATH /etc/supervisor/conf.d`添加了一个默认的环境变量。最后一行读取的`--→ 2a236efc3f06`是我们真正关心的，因为这给出了由`Step 6`生成的镜像 ID。您需要此 ID 来排查构建问题，因为它是构建中最后一个成功步骤生成的镜像。

有了这些信息，可以运行一个交互式容器，这样您可以尝试确定为何您的构建无法正常工作。请记住，每个容器镜像都基于其下面的镜像层。其中一个巨大的好处是，我们可以直接将较低的层作为一个容器本身运行，使用 shell 来查看周围的情况！

```
$ docker container run --rm -ti 2a236efc3f06 /bin/bash
root@b83048106b0f:/#
```

从容器内部，您现在可以运行任何您可能需要执行的命令，以确定导致构建失败的原因及您需要执行的操作以修复您的*Dockerfile*：

```
root@b83048106b0f:/# apt-get -y update-all
E: Invalid operation update-all

root@b83048106b0f:/# apt-get --help
apt 1.4.9 (amd64)
…

Most used commands:
 update - Retrieve new lists of packages
…

root@b83048106b0f:/# apt-get -y update
Get:1 http://security.debian.org/debian-security stretch/updates … [53.0 kB]
…
Reading package lists… Done

root@b83048106b0f:/# exit
exit
```

一旦确定了根本原因，*Dockerfile*可以修复，这样`RUN apt-get -y update-all`现在读作`RUN apt-get -y update`，然后重新构建镜像应该会成功：

```
$ DOCKER_BUILDKIT=0 docker image build -t example/docker-node-hello:latest .
Sending build context to Docker daemon  15.87kB
…
Successfully built 69f5e83bb86e
Successfully tagged example/docker-node-hello:latest
```

## 调试 BuildKit 镜像

当使用 BuildKit 时，我们必须采用稍微不同的方法来获取构建失败点的访问权限，因为构建容器中的任何中间构建层都不会导出到 Docker 守护程序。

调试 BuildKit 的选项随着我们的前进几乎肯定会发生变化，但让我们看看现在有效的一种方法。

假设*Dockerfile*已经恢复到原始状态，让我们修改读取的那一行：

```
RUN npm install
```

现在它应该是：

```
RUN npm installer
```

然后尝试构建镜像。

###### 提示

确保已启用 BuildKit！

```
$ docker image build -t example/docker-node-hello:debug --no-cache .

[+] Building 51.7s (13/13) FINISHED
 => [internal] load build definition from Dockerfile                      0.0s
…
 => [7/8] WORKDIR /data/app                                               0.0s
 => ERROR [8/8] RUN npm installer                                         0.4s
______
 > [8/8] RUN npm installer:
#13 0.399
#13 0.399 Usage: npm <command>
…
#13 0.402 Did you mean one of these?
#13 0.402     install
#13 0.402     install-test
#13 0.402     uninstall
______
executor failed running [/bin/sh -c npm installer]: exit code: 1
```

正如我们预期的那样看到错误，但我们如何获取访问权限以便进行故障排除？

有效的一种方法是利用多阶段构建和`docker image build`的`--target`参数。

让我们从两个地方修改*Dockerfile*。更改这一行：

```
FROM docker.io/node:18.13.0
```

现在它应该是：

```
FROM docker.io/node:18.13.0 as deploy
```

然后在导致错误的那一行之前立即添加一个新的`FROM`行：

```
FROM deploy
RUN npm installer
```

通过这样做，我们正在创建一个多阶段构建，其中第一阶段包含我们知道正在工作的所有步骤，第二阶段从我们的有问题的步骤开始。

如果我们尝试使用与之前相同的命令重新构建它，它仍然会失败：

```
$ docker image build -t example/docker-node-hello:debug .

[+] Building 51.7s (13/13) FINISHED
…
executor failed running [/bin/sh -c npm installer]: exit code: 1
```

因此，与其这样做，让我们告诉 Docker 我们只想在我们的多阶段*Dockerfile*中构建第一个镜像：

```
$ docker image build -t example/docker-node-hello:debug --target deploy .

[+] Building 0.8s (12/12) FINISHED
 => [internal] load build definition from Dockerfile                   0.0s
 => => transferring dockerfile: 37B                                    0.0s
…
 => exporting to image                                                 0.1s
 => => exporting layers                                                0.1s
 => => writing image sha256:a42dfbcfc7b18ee3d30ace944ad4134ea2239a2c0  0.0s
 => => naming to docker.io/example/docker-node-hello:debug             0.0s
```

现在，我们可以从这个镜像创建一个容器，并进行我们需要的任何测试：

```
$ docker container run --rm -ti docker.io/example/docker-node-hello:debug \
  /bin/bash

root@17807997176e:/data/app# ls
index.js  package.json

root@17807997176e:/data/app# npm install
…
added 18 packages from 16 contributors and audited 18 packages in 1.248s
…

root@17807997176e:/data/app# exit
exit
```

然后一旦我们理解了*Dockerfile*有什么问题，我们可以恢复我们的调试更改，并修复`npm`行，以便整个构建按预期工作。

# 多架构构建

自从 Docker 发布以来，*AMD64/X86_64*架构一直是大多数容器所针对的主要平台。然而，这种情况已经开始显著改变。越来越多的开发人员正在使用基于 ARM64/AArch64 的系统，云公司也开始通过其平台提供基于 ARM 的 VM，这是因为 ARM 平台具有更低的计算成本。

这可能会对需要构建和维护面向多个架构的镜像的任何人造成一些有趣的挑战。在支持所有这些不同目标的同时，如何保持单一，简化的代码库和流水线？

幸运的是，Docker 已经发布了一个名为`buildx`的`docker` CLI 插件，可以帮助简化这个过程。在许多情况下，`docker-buildx`已经安装在您的系统上，您可以像这样验证：

```
$ docker buildx version
github.com/docker/buildx v0.9.1 ed00243a0ce2a0aee75311b06e32d33b44729689
```

###### 提示

如果需要安装插件，可以按照从[Github 仓库的说明](https://github.com/docker/buildx#installing)进行操作。

默认情况下，`docker-buildx`将利用[基于 QEMU 的虚拟化](https://www.qemu.org)和[`binfmt_misc`](https://docs.kernel.org/admin-guide/binfmt-misc.html)来支持与基础系统不同的架构。这可能已经在您的 Linux 系统上设置好了，但以防万一，建议在首次设置新的 Docker 服务器时运行以下命令，以确保 QEMU 文件已正确注册并更新到最新状态：

```
$ docker container run --rm --privileged multiarch/qemu-user-static \
    --reset -p yes

Setting /usr/bin/qemu-alpha-static as binfmt interpreter for alpha
Setting /usr/bin/qemu-arm-static as binfmt interpreter for arm
Setting /usr/bin/qemu-armeb-static as binfmt interpreter for armeb
…
Setting /usr/bin/qemu-aarch64-static as binfmt interpreter for aarch64
Setting /usr/bin/qemu-aarch64_be-static as binfmt interpreter for aarch64_be
…
```

与原始嵌入式 Docker 构建功能不同，BuildKit 在构建镜像时可以利用一个构建容器，这意味着可以提供许多功能灵活性。在下一步中，我们将创建一个名为`builder`的默认`buildx`容器。

###### 提示

如果您已经存在名为此名称的`buildx`容器，您可以通过运行`docker buildx rm builder`来删除它，或者您可以在即将到来的`docker buildx create`命令中更改名称。

使用下面的两个命令，我们将创建构建容器，并将其设置为默认值，然后启动它：

```
$ docker buildx create --name builder --driver docker-container --use
builder

$ docker buildx inspect --bootstrap
[+] Building 9.6s (1/1) FINISHED
 => [internal] booting buildkit                                           9.6s
 => => pulling image moby/buildkit:buildx-stable-1                        8.6s
 => => creating container buildx_buildkit_builder0                        0.9s
Name:   builder
Driver: docker-container

Nodes:
Name:      builder0
Endpoint:  unix:///var/run/docker.sock
Status:    running
Buildkit:  v0.10.5
Platforms: linux/amd64, linux/amd64/v2, linux/arm64, linux/riscv64,
 linux/ppc64le, linux/s390x, linux/386, linux/mips64le,
 linux/mips64, linux/arm/v7, linux/arm/v6
```

例如，让我们继续下载包含一个有用工具的`wordchain` Git 存储库，该工具可以生成随机和确定性的单词序列，以帮助动态命名需求：

```
$ git clone https://github.com/spkane/wordchain.git \
  --config core.autocrlf=input
$ cd wordchain
```

让我们继续查看附带的*Dockerfile*。您会注意到它是一个相当正常的多阶段*Dockerfile*，并且没有任何与平台架构相关的特殊内容：

```
FROM golang:1.18-alpine3.15 AS build

RUN apk --no-cache add \
    bash \
    gcc \
    musl-dev \
    openssl

ENV CGO_ENABLED=0

COPY . /build
WORKDIR /build

RUN go install github.com/markbates/pkger/cmd/pkger@latest && \
    pkger -include /data/words.json && \
    go build .

FROM alpine:3.15 AS deploy

WORKDIR /
COPY --from=build /build/wordchain /

USER 500
EXPOSE 8080

ENTRYPOINT ["/wordchain"]
CMD ["listen"]
```

在第一步中，我们将构建我们的静态编译的 Go 二进制文件，然后在第二步中，我们将把它打包成一个小的部署镜像。

###### 注意

*Dockerfile*中的`ENTRYPOINT`指令是一个高级指令，允许您将容器运行的默认进程（`ENTRYPOINT`）与传递给该进程的命令行参数（`CMD`）分开。当*Dockerfile*中缺少`ENTRYPOINT`时，预期`CMD`指令将包含进程及其所有必需的命令行参数。

我们可以继续构建此镜像，并通过运行以下命令将其侧加载到我们的本地 Docker 服务器中：

```
$ docker buildx build --tag wordchain:test --load .

[+] Building 2.4s (16/16) FINISHED
 => [internal] load .dockerignore                                         0.0s
 => => transferring context: 93B                                          0.0s
 => [internal] load build definition from Dockerfile                      0.0s
 => => transferring dockerfile: 461B                                      0.0s
…
 => exporting to oci image format                                         0.3s
 => => exporting layers                                                   0.0s
 => => exporting manifest sha256:4bd1971f2ed820b4f64ffda97707c27aac3e8eb7 0.0s
 => => exporting config sha256:ce8f8564bf53b283d486bddeb8cbb074ff9a9d4ce9 0.0s
 => => sending tarball                                                    0.2s
 => importing to docker                                                   0.0s
```

我们可以通过运行以下命令快速测试该镜像：

```
$ docker container run wordchain:test random

witty-stack

$ docker container run wordchain:test random -l 3 -d .

odd.goo

$ docker container run wordchain:test --help

wordchain is an application that can generate a readable chain
 of customizable words for naming things like
 containers, clusters, and other objects.
…
```

只要您在第一个两个命令中收到了一些随机单词对返回的结果，那么一切都按预期运行。

现在，要为多个架构构建此镜像，只需在我们的构建中添加`--platform`参数即可。

###### 注意

通常情况下，我们会将 `--load` 替换为 `--push`，这样会将所有生成的镜像推送到标记的仓库。但在这种情况下，我们只需简单地移除 `--load`，因为 Docker 服务器目前无法加载多个平台的镜像，并且我们也没有设置推送这些镜像的仓库。如果我们有一个仓库并正确标记了这些镜像，那么我们可以轻松地使用如下命令一步构建和推送所有生成的镜像：

`docker buildx build --platform linux/amd64,linux/arm64 --tag docker.io/spkane/wordchain:latest --push .`

你可以像这样为 linux/amd64 和 linux/arm64 平台构建此镜像：

```
$ docker buildx build --platform linux/amd64,linux/arm64 \
    --tag wordchain:test .

[+] Building 114.9s (23/23) FINISHED
…
 => [linux/arm64 internal] load metadata for docker.io/library/alpine:3.1 2.7s
 => [linux/amd64 internal] load metadata for docker.io/library/alpine:3.1 2.7s
 => [linux/arm64 internal] load metadata for docker.io/library/golang:1.1 3.0s
 => [linux/amd64 internal] load metadata for docker.io/library/golang:1.1 2.8s
…
 => CACHED [linux/amd64 build 5/5] RUN go install github.com/markbates/pk 0.0s
 => CACHED [linux/amd64 deploy 2/3] COPY --from=build /build/wordchain /  0.0s
 => [linux/arm64 build 5/5] RUN go install github.com/markbates/pkger/c 111.7s
 => [linux/arm64 deploy 2/3] COPY --from=build /build/wordchain /         0.0s
WARNING: No output specified with docker-container driver. Build result will
 only remain in the build cache. To push result image into registry
 use --push or to load image into docker use --load
```

###### 注意

在为非本地架构构建镜像时，由于需要进行仿真，你可能会注意到某些步骤比正常情况下花费更多时间。这是可以预期的，因为额外的计算开销来自仿真过程。

可以配置 Docker，使其在具有匹配架构的工作节点上构建每个镜像，这在很多情况下可以显著加快构建速度。有关此内容的更多信息可以在这篇[Docker 博客文章](https://www.docker.com/blog/speed-up-building-with-docker-buildx-and-graviton2-ec2)中找到。

在构建输出中，你会注意到以类似 `=> [linux/amd64 *]` 或 `=> [linux/arm64 *]` 开头的行。每行代表了构建步骤在指定平台上的工作状态。许多这样的步骤会并行运行，由于缓存和其他考虑因素，每个构建的进度可能会不同步。

由于我们没有在构建中添加 `--push`，所以你还会注意到在构建结束时收到了一个警告。这是因为构建器使用的 *docker-container* 驱动器只是将一切留在构建缓存中，这意味着我们不能运行生成的镜像；在这一点上，我们只能确信构建是有效的。

###### 提示

有一些[`build` 参数](https://docs.docker.com/engine/reference/builder/#automatic-platform-args-in-the-global-scope)是 Docker 自动设置的，特别是在进行多架构构建时，这些参数可以非常有帮助。例如，`TARGETARCH` 经常用于确保给定的构建步骤下载当前镜像平台的正确预构建二进制文件。

所以，当我们将这个镜像上传到仓库时，Docker 如何知道要使用哪个镜像来适配本地平台？这些信息通过称为 *镜像清单* 的东西提供给 Docker 服务器。我们可以通过以下命令查看 *docker.io/spkane/workdchain* 的清单：

```
$ docker manifest inspect docker.io/spkane/wordchain:latest
```

```
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 739,
         "digest": "sha256:4bd1…bfc0",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
…
         "platform": {
            "architecture": "arm64",
            "os": "linux"
         }
      },
…
   ]
}
```

如果您查看输出，您会看到每个平台支持的图像都需要识别的块。这通过单独的[`digest`](https://github.com/opencontainers/image-spec/blob/main/descriptor.md#digests)条目完成，然后与一个*platform*块配对。服务器在需要图像时下载此清单文件，然后引用清单后，服务器将为本地平台下载正确的图像。这就是我们的*Dockerfile*能够正常工作的原因。每个`FROM`行列出了我们要使用的基础图像，但是 Docker 服务器利用此清单文件来确定为构建所针对的每个平台下载哪个图像。

# 总结

此时，您应该对为 Docker 创建图像感到非常自在，并且应该对许多可以利用来简化构建流水线的核心工具和功能有了牢固的理解。在下一章中，我们将开始探讨如何使用您的图像为项目创建容器化进程。

^(1) 完整网址: [*https://github.com/torvalds/linux/commit/e9be9d5e76e34872f0c37d72e25bc27fe9e2c54c*](https://github.com/torvalds/linux/commit/e9be9d5e76e34872f0c37d72e25bc27fe9e2c54c)

^(2) 此代码最初是从[GitHub](https://github.com/enokd/docker-node-hello)分支出来的。

^(3) 完整网址: [*https://docs.docker.com/registry/recipes/mirror/#configure-the-docker-daemon*](https://docs.docker.com/registry/recipes/mirror/#configure-the-docker-daemon)

^(4) 完整网址: [*https://docs.docker.com/registry/recipes/mirror/#run-a-registry-as-a-pull-through-cache*](https://docs.docker.com/registry/recipes/mirror/#run-a-registry-as-a-pull-through-cache)

^(5) 完整网址: [*https://github.com/bluewhalebook/docker-up-and-running-3rd-edition/blob/main/chapter_04/multistage/Dockerfile*](https://github.com/bluewhalebook/docker-up-and-running-3rd-edition/blob/main/chapter_04/multistage/Dockerfile)

^(6) 完整网址: [*https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-command?view=powershell-7.3*](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/measure-command?view=powershell-7.3)
