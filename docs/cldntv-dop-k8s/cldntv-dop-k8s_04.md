# 第二章：Kubernetes 初步

> 你已经踏出了进入更大世界的第一步。
> 
> 奥比-旺·克诺比，《星球大战：新希望》

理论够了，让我们开始使用 Kubernetes 和容器工作吧。在本章中，你将构建一个简单的容器化应用程序，并将其部署到运行在你的机器上的本地 Kubernetes 集群中。在这个过程中，你将会接触到一些非常重要的云原生技术和概念：Docker、Git、Go、容器注册表和`kubectl`工具。

###### 提示

本章是互动的！在本书中，我们会要求你按照示例在自己的计算机上安装软件、输入命令并运行容器。我们认为这比仅仅通过文字解释来学习更为有效。你可以在[GitHub](https://oreil.ly/LAI8f)上找到所有的示例。

# 运行你的第一个容器

正如我们在第一章中看到的，容器是云原生开发的关键概念之一。构建和运行容器最流行的工具是 Docker。还有其他工具用于运行容器，但我们稍后会详细介绍。

在本节中，我们将使用 Docker Desktop 工具构建一个简单的演示应用程序，在本地运行它，并将镜像推送到容器注册表。

如果你已经对容器非常熟悉，请直接跳到“你好，Kubernetes”，那里才是真正的乐趣所在。如果你想知道容器是什么，它们如何工作，并在开始学习 Kubernetes 之前对它们进行一些实际操作，可以继续阅读。

## 安装 Docker Desktop

Docker Desktop 是 Mac 和 Windows 的免费软件包。它带有一个完整的 Kubernetes 开发环境，你可以在笔记本电脑或台式机上测试你的应用程序。

现在让我们安装 Docker Desktop 并使用它来运行一个简单的容器化应用程序。如果你已经安装了 Docker，请跳过本节，直接进入“运行容器镜像”。

下载适合你的计算机版本的[Docker Desktop Community Edition](https://oreil.ly/Klafa)，然后按照你的平台上的说明安装 Docker 并启动它。

###### 注意

目前 Docker Desktop 尚不支持 Linux，因此 Linux 用户需要安装[Docker 引擎](https://oreil.ly/4B1K5)，然后安装[Minikube](https://oreil.ly/ExEjc)（参见“Minikube”）。

一旦完成了这一步，你应该能够打开终端并运行以下命令：

```
`docker version`
 ... Version:           20.10.7 ...
```

根据你的平台，确保 Docker 正确安装和运行，你将看到类似示例输出的内容。

在 Linux 系统上，你可能需要运行`sudo docker version`。你可以使用`sudo usermod -aG docker $USER && newgrp docker`将你的帐户添加到 docker 组，然后你就不需要每次都使用`sudo`了。

## Docker 是什么？

[Docker](https://docs.docker.com) 实际上是几个不同但相关的东西：一个容器镜像格式，一个管理容器生命周期的容器运行库，一个打包和运行容器的命令行工具，以及一个容器管理的 API。这里不需要关注细节，因为 Kubernetes 支持 Docker 容器作为其众多组件之一，尽管它是一个重要的组件。

## 运行一个容器镜像

到底什么是容器镜像？从技术细节上讲并不重要，但您可以将镜像看作是类似于 ZIP 文件。它是一个唯一 ID 的单一二进制文件，包含了运行容器所需的一切。

无论您是直接使用 Docker 运行容器，还是在 Kubernetes 集群上运行，您只需指定一个容器镜像 ID 或 URL，系统将负责查找、下载、解压和启动容器。

我们编写了一个小型演示应用程序，将在整本书中用它来说明我们所讲的内容。您可以下载并运行我们之前准备好的容器镜像来使用该应用程序。运行以下命令以试用：

```
`docker container run -p 9999:8888 --name hello cloudnatived/demo:hello`
```

让此命令保持运行状态，并将浏览器指向 *http://localhost:9999/*。

您应该会看到一个友好的消息：

```
Hello, 世界
```

每当您请求这个 URL，我们的演示应用程序都将准备好并等待着迎接您。

当您玩够了，请通过在终端中按下 Ctrl-C 来停止容器。

# 演示应用程序

它是如何工作的？让我们下载运行在此容器中的演示应用程序的源代码，并查看一下。

对于这部分，您需要安装 Git。^(1) 如果您不确定是否已安装 Git，请尝试以下命令：

```
`git version`
git version 2.32.0
```

如果您还没有安装 Git，请按照您平台的[安装说明](https://git-scm.com/download)进行操作。

安装完 Git 后，运行此命令：

```
`git clone https://github.com/cloudnativedevops/demo.git`
Cloning into *`demo`*... ...
```

## 查看源代码

此 Git 仓库包含了我们将在整本书中使用的演示应用程序。为了更好地了解每个阶段的情况，该仓库在不同的子目录中包含了每个连续版本的应用程序。第一个版本简单命名为 *hello*。要查看源代码，请运行此命令：

```
`cd demo/hello`
`ls`
Dockerfile  README.md go.mod      main.go
```

在您喜爱的编辑器中打开 *main.go* 文件（我们推荐使用[Visual Studio Code](https://code.visualstudio.com)，它对 Go、Docker 和 Kubernetes 开发有很好的支持）。您将看到以下源代码：

```
package main

import (
        "fmt"
        "log"
        "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello, 世界")
}

func main() {
        http.HandleFunc("/", handler)
        fmt.Println("Running demo app. Press Ctrl+C to exit...")
        log.Fatal(http.ListenAndServe(":8888", nil))
}
```

## 介绍 Go

我们的演示应用程序是用 Go 编程语言编写的。

Go 是一种现代编程语言（自 2009 年起由 Google 开发），注重简单性、安全性和可读性，并专为构建大规模并发应用程序（特别是网络服务）而设计。在这种语言中编程也非常有趣。^(2)

Kubernetes 本身就是用 Go 编写的，Docker、Terraform 和许多其他流行的开源项目也是如此。这使得 Go 成为开发云原生应用程序的一个不错选择。

## 演示应用程序的工作原理

正如您所见，演示应用程序相当简单，尽管它实现了一个 HTTP 服务器（Go 提供了一个强大的标准库）。它的核心是这个名为 `handler` 的函数：

```
func handler(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintln(w, "Hello, 世界")
}
```

如其名称所示，它处理 HTTP 请求。请求作为参数传递给函数（尽管函数目前没有对其进行任何操作）。

HTTP 服务器还需要一种方法向客户端发送响应。`http.ResponseWriter` 对象使我们的函数能够向用户发送消息，以在其浏览器中显示，本例中只是字符串 `Hello, 世界`。

任何语言中的第一个示例程序传统上会打印 `Hello, world`。但是因为 Go 本地支持 Unicode（国际文本表示标准），示例 Go 程序通常打印 `Hello, 世界`，只是为了炫耀。如果您不会说中文，没关系：Go 会处理！

程序的其余部分负责将 `handler` 函数注册为处理 HTTP 请求的处理程序，并打印应用程序正在启动的消息，然后实际启动 HTTP 服务器以监听并提供在 8888 端口上提供服务。

这就是整个应用程序！它现在还没有做太多事情，但随着我们的继续，我们将为其添加功能。

# 构建容器

您知道容器镜像是一个包含容器运行所需一切的单个文件，但首先如何构建镜像呢？嗯，要做到这一点，您使用 `docker image build` 命令，它以特殊的文本文件（称为*Dockerfile*）作为输入。Dockerfile 精确地指定了需要放入容器镜像中的内容。

容器的一个关键优势是能够基于现有镜像构建新镜像。例如，您可以获取一个包含完整 Ubuntu 操作系统的容器镜像，向其添加一个文件，结果将是一个新的镜像。

通常，一个 Dockerfile 包含一些指令，用于获取一个起始镜像（所谓的*基础镜像*），以某种方式进行转换，并将结果保存为一个新的镜像。

## 理解 Dockerfile

让我们看看我们演示应用程序的 Dockerfile（它位于应用程序存储库的*hello*子目录中）：

```
FROM golang:1.17-alpine AS build

WORKDIR /src/
COPY main.go go.* /src/
RUN CGO_ENABLED=0 go build -o /bin/demo

FROM scratch
COPY --from=build /bin/demo /bin/demo
ENTRYPOINT ["/bin/demo"]
```

目前不需要详细了解它是如何工作的确切细节，但它使用了一种称为*多阶段构建*的标准 Go 容器构建过程。第一阶段从官方的 `golang` 容器镜像开始，它只是一个安装了 Go 语言环境的操作系统（在本例中是 Alpine Linux）。它运行 `go build` 命令来编译我们之前看到的 *main.go* 文件。

这个操作的结果是一个名为*demo*的可执行二进制文件。第二阶段获取一个完全空的容器镜像（称为*scratch*镜像，即*从零开始*），并将*demo*二进制文件复制到其中。

## 极简容器镜像

为什么需要第二个构建阶段？嗯，Go 语言环境以及其余的 Alpine Linux，实际上只是用来 *构建* 程序的。运行程序只需要 *demo* 二进制文件，因此 Dockerfile 创建一个新的 scratch 容器来放置它。生成的镜像非常小（约 6 MiB）— 这就是可以在生产中部署的镜像。

如果没有第二阶段，你最终得到的容器镜像将会有大约 350 MiB 的大小，其中 98% 是不必要的，永远不会被执行。镜像越小，上传和下载速度就越快，启动速度也就越快。

最小化容器也有减少安全问题攻击面的好处。容器中程序越少，潜在的漏洞就越少。

因为 Go 是一种可以生成自包含可执行文件的编译语言，所以非常适合编写最小化的容器。相比之下，官方的 Ruby 容器镜像大小为 850 MB；比我们的 Alpine Go 镜像大约大 140 倍，而且这还不包括你添加的 Ruby 程序！另一个用于使用精简容器的好资源是 [*distroless* images](https://oreil.ly/V3AFm)，它们只包含运行时依赖项，保持最终容器镜像的大小小。

## 运行 Docker 镜像构建

我们已经看到 Dockerfile 包含 `docker image build` 工具将我们的 Go 源代码转换为可执行容器的指令。让我们继续尝试一下。在 *hello* 目录中，运行以下命令：

```
`docker image build -t myhello .`
Sending build context to Docker daemon  4.096kB Step 1/7 : FROM golang:1.17-alpine AS build ... Successfully built eeb7d1c2e2b7 Successfully tagged myhello:latest
```

恭喜！你刚刚构建了你的第一个容器！你可以从输出中看到，Docker 在 Dockerfile 中逐步执行每个动作，最终生成一个可用的镜像。

## 给你的镜像命名

当你构建一个镜像时，默认情况下它只会得到一个十六进制的 ID，你可以稍后用它来引用（例如，运行它）。这些 ID 并不是特别容易记住或输入，因此 Docker 允许你使用 `-t` 开关为镜像指定一个易读的名称。在前面的例子中，你将镜像命名为 `myhello`，所以现在应该能够使用这个名称来运行镜像。

让我们看看它是否正常工作：

```
`docker container run -p 9999:8888 myhello`
```

现在你正在运行自己的演示应用程序副本，并且可以通过浏览到与之前相同的 URL (*http://localhost:9999/*) 来检查它。

你应该看到 `Hello, 世界`。当你运行完这个镜像后，按 Ctrl-C 停止 `docker container run` 命令。

## 端口转发

在容器中运行的程序与在同一台机器上运行的其他程序隔离，这意味着它们不能直接访问网络端口等资源。

演示应用程序侦听端口 8888 上的连接，但这是*容器*的私有端口 8888，并非您计算机上的端口。为了连接到容器的端口 8888，您需要将本地机器上的端口*转发*到容器的端口。它可以是（*几乎*）任何端口，包括 8888，但我们将使用 9999，以清楚表明哪个是您的端口，哪个是容器的端口。

要告诉 Docker 转发端口，您可以使用`-p`开关，就像在“运行容器镜像”中一样：

```
`docker container run -p HOST_PORT:CONTAINER_PORT ...`
```

一旦容器运行起来，本地计算机上对`HOST_PORT`的任何请求都会自动转发到容器中的`CONTAINER_PORT`，这就是您如何通过浏览器连接到应用程序的方式。

我们之前说过，您可以使用*几乎*任何端口，因为低于`1024`的任何端口号都被视为[*特权*端口](https://oreil.ly/q5SAU)，这意味着为了使用这些端口，您的进程必须以特殊权限的用户（如`root`）运行。普通非管理员用户无法使用低于 1024 的端口，因此为了避免权限问题，我们将在示例中使用较高的端口号。

# 容器注册表

在“运行容器镜像”中，只需提供镜像名称，Docker 就会自动下载并运行镜像。

您可能会合理地想知道它是从哪里下载的。虽然您可以通过仅构建和运行本地镜像来完美使用 Docker，但如果能够从*容器注册表*推送和拉取镜像，则更为实用。注册表允许您存储镜像并使用唯一名称（如`cloudnatived/demo:hello`）检索它们。

`docker container run`命令的默认注册表是 Docker Hub，但您可以指定一个不同的注册表，或者设置自己的注册表。

现在，让我们继续使用 Docker Hub。虽然您可以从 Docker Hub 下载和使用任何公共容器镜像，但要推送您自己的镜像，您需要一个帐户（称为*Docker ID*）。请按照[Docker Hub](https://hub.docker.com)上的说明创建您的 Docker ID。

## 认证注册表

一旦您获得了您的 Docker ID，下一步就是使用您的 ID 和密码将本地 Docker 客户端与 Docker Hub 连接起来：

```
docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't
have a Docker ID, head over to https://hub.docker.com to create one.
Username: *YOUR_DOCKER_ID*
Password: *YOUR_DOCKER_PASSWORD*
Login Succeeded
```

## 给镜像命名并推送

为了能够将本地镜像推送到注册表，您需要使用以下格式命名它：`_YOUR_DOCKER_ID_/myhello`。

要创建此名称，您无需重新构建镜像；相反，请运行此命令：

```
docker image tag myhello *YOUR_DOCKER_ID*/myhello
```

这样做是为了在将镜像推送到注册表时，Docker 知道要将其存储在哪个帐户中。

请继续使用以下命令将镜像推送到 Docker Hub：

```
docker image push *YOUR_DOCKER_ID*/myhello
The push refers to repository [docker.io/*YOUR_DOCKER_ID*/myhello]
b2c591f16c33: Pushed
latest: digest:
sha256:7ac57776e2df70d62d7285124fbff039c9152d1bdfb36c75b5933057cefe4fc7
size: 528
```

## 运行您的镜像

恭喜！您的容器镜像现在可以在任何地方运行（至少是在有互联网访问的地方），使用以下命令：

```
docker container run -p 9999:8888 *YOUR_DOCKER_ID*/myhello
```

# 你好，Kubernetes

现在你已经构建并推送了第一个容器镜像到镜像仓库，你可以使用 `docker container run` 命令运行它，但那不是很有趣。让我们做一些更有冒险精神的事情，并在 Kubernetes 中运行它。

有很多方法可以获得 Kubernetes 集群，在第三章中我们将更详细地探讨其中的一些。如果你已经可以访问 Kubernetes 集群，那很棒，如果愿意，你可以在本章的其余示例中使用它。

如果没有，不要担心。Docker Desktop 包含 Kubernetes 支持（Linux 用户，请参阅 “Minikube”）。要启用它，打开 Docker Desktop 首选项，选择 Kubernetes 选项卡，然后选中启用。详细信息请参阅 [Docker Desktop Kubernetes 文档](https://oreil.ly/4eBpc)。

安装和启动 Kubernetes 可能需要几分钟时间。一旦完成，你就可以运行演示应用程序了！

Linux 用户还需要安装 `kubectl` 工具，按照 [Kubernetes 文档网站](https://oreil.ly/EXGeU) 上的说明操作。

## 运行演示应用

让我们首先运行你之前构建的演示镜像。打开终端并使用以下参数运行 `kubectl` 命令：

```
kubectl run demo --image=*YOUR_DOCKER_ID*/myhello --port=9999 --labels app=demo
pod/demo created
```

现在暂时不必担心这个命令的细节：它基本上是 Kubernetes 中 `docker container run` 命令的等价物，你之前在本章中用来运行演示镜像的命令。如果你还没有构建自己的镜像，你可以使用我们的：`--image=cloudnatived/demo:hello`。

请记住，为了使用浏览器连接到容器的端口 8888，你需要在本地机器上将端口 9999 转发到那里。在这里也需要使用 `kubectl port-forward` 命令：

```
`kubectl port-forward pod/demo 9999:8888`
Forwarding from 127.0.0.1:9999 -> 8888 Forwarding from [::1]:9999 -> 8888
```

让这个命令保持运行状态，并在新终端中继续进行操作。

使用浏览器连接到 *http://localhost:9999/* 查看 `Hello, 世界` 消息。

可能需要几秒钟来启动容器并使应用程序可用。如果半分钟后还没有准备好，尝试这个命令：

```
`kubectl get pods --selector app=demo`
NAME                    READY     STATUS    RESTARTS   AGE demo                    1/1       Running   0          9m
```

当容器运行并使用浏览器连接时，你会在终端看到这条消息：

```
Handling connection for 9999
```

## 如果容器无法启动

如果 `STATUS` 没有显示为 `Running`，可能会有问题。例如，如果状态是 `ErrImagePull` 或 `ImagePullBackoff`，这意味着 Kubernetes 无法找到并下载你指定的镜像。你可能在镜像名称中有拼写错误；检查你的 `kubectl run` 命令。

如果状态是 `ContainerCreating`，那一切都很好；Kubernetes 仍在下载和启动镜像。稍等几秒钟，然后再次检查。

完成后，你会想要清理你的演示容器：

```
`kubectl delete pod demo`
pod "demo" deleted
```

我们将在接下来的章节中详细介绍更多 Kubernetes 的术语，但现在你可以将 *Pod* 理解为在 Kubernetes 中运行的容器，类似于你在电脑上运行 Docker 容器的方式。

# Minikube

如果您不想使用或无法使用 Docker Desktop 中的 Kubernetes 支持，还有一个替代方案：备受喜爱的 Minikube。与 Docker Desktop 类似，Minikube 提供在您自己的机器上运行的单节点 Kubernetes 集群（实际上在虚拟机中，但这并不重要）。

要安装 Minikube，请按照官方[Minikube “入门指南”](https://oreil.ly/zGROa)中的说明操作。

# 总结

如果您像我们一样，对于关于 Kubernetes 如此伟大的冗长文章感到不耐烦，希望您喜欢在本章中掌握一些实际任务。如果您已经是经验丰富的 Docker 或 Kubernetes 用户，也许您会原谅这个复习课程。我们希望确保每个人都能在基本方式下构建和运行容器，并且在进入更高级内容之前，您可以拥有一个可以进行玩耍和实验的 Kubernetes 环境。

这是您应该从本章中了解到的内容：

+   所有源代码示例（以及更多）都可以在伴随本书的[demo 代码库](https://oreil.ly/LAI8f)中找到。

+   Docker 工具允许您在本地构建容器，将其推送到或从 Docker Hub 等容器注册表中拉取，并在本地机器上运行容器映像。

+   一个容器映像完全由 Dockerfile 指定：这是一个包含有关如何构建容器的说明的文本文件。

+   Docker Desktop 允许您在 Mac 或 Windows 机器上运行一个小型（单节点）Kubernetes 集群。Minikube 是另一个选项，在 Linux 上也可以运行。

+   `kubectl` 工具是与 Kubernetes 集群交互的主要方式。它可以用来在 Kubernetes 中创建资源，查看集群和 Pod 的状态，并应用 YAML 文件形式的 Kubernetes 配置。

^(1) 如果您对 Git 不熟悉，请阅读 Scott Chacon 和 Ben Straub 的优秀书籍[*Pro Git*](https://git-scm.com/book/en/v2)（Apress）。

^(2) 如果您对 Go 不熟悉，Jon Bodner 的[*学习 Go*](https://oreil.ly/aNSCq)（O'Reilly）是一本非常宝贵的指南。
