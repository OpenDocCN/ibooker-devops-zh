# 第十二章：扩展的景观

可用于与 Linux 容器交互的工具的格局在不断发展，特别是随着多年来 Kubernetes 的显著采纳。

在本章中，我们将快速浏览一些受 Docker 启发但通常专注于改善特定用例的工具。这并不意味着要全面列出，而是简单地让您体验一些可供探索的类别和选项。

# 客户端工具

在本节中，我们将介绍三个命令行工具：`nerdctl`、`podman`和`buildah`。所有这些工具对于熟悉 Docker 及其常见工作流程的人都可能很有用。

## nerdctl

尽管在许多基于`containerd`环境中默认安装了[`crictl`](https://oreil.ly/zElq_)^(1)，`nerdctl`是一个易于使用的 Docker 兼容 CLI，适用于`containerd`，值得一试。这意味着`nerdctl`可以为使用 Docker 但需要支持未运行 Docker 守护程序的`containerd`系统的人和脚本提供非常简便的迁移路径。

举个快速的例子，如果你用我们在“Kind”中讨论过的`kind`快速启动一个小型 Kubernetes 集群，你将会得到一个基于`containerd`的 Kubernetes 集群，但它并不直接兼容`docker`命令行界面：

```
$ kind create cluster --name nerdctl
Creating cluster "nerdctl" …
…

$ docker container exec -ti nerdctl-control-plane /bin/bash
```

现在你应该已经在`kind`/Kubernetes 容器内部了。

###### 注意

在接下来的`curl`命令中，你必须确保下载适合你架构的正确版本。你需要用你的系统架构`${ARCH}`替换为`amd64`或`arm64`。同时，可以尝试[下载最新版本](https://github.com/containerd/nerdctl/releases)的`nerdctl`。

一旦你编辑了以下的`curl`命令，并将其重新组装成一行，你就可以下载并解压`nerdctl`客户端，然后尝试几个命令：

```
root@nerdctl-control-plane:/# curl -s -L \
  "https://github.com/containerd/nerdctl/releases/download/v0.23.0/\
nerdctl-0.23.0-linux-${ARCH}.tar.gz" -o /tmp/nerdctl.tar.gz

root@nerdctl-control-plane:/# tar -C /usr/local/bin -xzf /tmp/nerdctl.tar.gz

root@nerdctl-control-plane:/# nerdctl namespace list

NAME      CONTAINERS    IMAGES    VOLUMES    LABELS
k8s.io    18            24        0

root@nerdctl-control-plane:/# nerdctl --namespace k8s.io container list

CONTAINER ID IMAGE                                  … NAMES
07ae69902d11 registry.k8s.io/pause:3.7              … k8s://kube-system/core…
0b241db0485f registry.k8s.io/coredns/coredns:v1.9.3 … k8s://kube-system/core…
…

root@nerdctl-control-plane:/# nerdctl --namespace k8s.io container run --rm \
                              --net=host debian sleep 5

docker.io/library/debian:latest:  resolved       |+++++++++++++++++++++++++++|
index-sha256:e538…4bff:           done           |+++++++++++++++++++++++++++|
manifest-sha256:9b0e…2f7d:        done           |+++++++++++++++++++++++++++|
config-sha256:d917…d33c:          done           |+++++++++++++++++++++++++++|
layer-sha256:f606…5ddf:           done           |+++++++++++++++++++++++++++|
elapsed: 6.4 s                    total:  52.5 M (8.2 MiB/s)

root@nerdctl-control-plane:/# exit
```

在大多数情况下，`nerdctl`几乎可以无需修改地使用`docker`命令。唯一可能突出的变化是通常需要提供一个命名空间值。这是因为`containerd`提供了[完全命名空间化的 API](https://github.com/containerd/containerd/blob/main/docs/namespaces.md)，我们需要指定我们想要与之交互的命名空间。

一旦退出`kind`容器，你可以继续删除它：

```
$ kind delete cluster --name nerdctl

Deleting cluster "nerdctl" …
```

## podman 和 buildah

[`podman`](https://podman.io)和[`buildah`](https://buildah.io)是 Red Hat 提供的一组工具，早期创建的目的是提供一个不依赖于像 Docker 那样的守护进程的容器工作流程。它在 Red Hat 社区内被广泛使用，并重新思考了镜像构建和容器运行管理的方式。

###### 小贴士

您可以在 Red Hat 博客上找到一个适合 Docker 用户的[`podman`和`buildah`介绍](https://developers.redhat.com/blog/2019/02/21/podman-and-buildah-for-docker-users)。

```
$ kind create cluster --name podman
Creating cluster "podman" …
…

$ docker container exec -ti podman-control-plane /bin/bash
```

###### 提示

安装和使用`kind`的概述可以在“Kind”中找到。

现在您应该已经进入`kind`/Kubernetes 容器：

```
root@podman-control-plane:/# apt update
Get:1 http://security.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
…

root@podman-control-plane:/# apt install -y podman
Reading package lists… Done
…

root@podman-control-plane:/# podman container run -d --rm \
                             --name test debian sleep 120
9b6b333313c0d54e2da6cda49f2787bc5213681d90dac145a9f64128f3e18631

root@podman-control-plane:/# podman container list

CONTAINER ID  IMAGE                            COMMAND    …  NAMES
548a2f709785  docker.io/library/debian:latest  sleep 120  …  test

root@podman-control-plane:/# podman container stop test
test
```

与`docker`（与 Docker 守护程序接口）和`nerdctl`（与`containerd`接口）不同，`podman`跳过容器引擎，直接与底层容器运行时（如`runc`）接口。

尽管`podman build`也可以用于构建容器，但`buildah`提供了一个高级接口用于图像构建，可以脚本化整个图像构建过程，并消除对*Dockerfile*格式（或`podman`称为的*Containerfile*）的依赖。

我们在这里不会深入探讨`buildah`的细节，但您可以在`kind`容器中尝试一个非常简单的示例，如果您对传统的`Dockerfile`方法的替代方法或 BuildKit 的新替代方法（通过[LBB 接口](https://github.com/moby/buildkit#exploring-llb)）感兴趣，您可以通过[GitHub](https://github.com/containers/buildah)和[Red Hat 博客](https://www.redhat.com/sysadmin/building-buildah)在线阅读更多关于`buildah`的信息。

要在`kind`容器中尝试运行`buildah`脚本，请继续运行以下命令：

```
root@podman-control-plane:/# cat > apache.sh <<"EOF"
```

```
#!/usr/bin/env bash

set -x

ctr1=$(buildah from "${1:-fedora}")

## Get all updates and install the apache server
buildah run "$ctr1" -- dnf update -y
buildah run "$ctr1" -- dnf install -y httpd

## Include some buildtime annotations
buildah config --annotation "com.example.build.host=$(uname -n)" "$ctr1"

## Run our server and expose the port
buildah config --cmd "/usr/sbin/httpd -D FOREGROUND" "$ctr1"
buildah config --port 80 "$ctr1"

## Commit this container to an image name
buildah commit "$ctr1" "${2:-myrepo/apache}"
```

```
EOF

root@podman-control-plane:/# chmod +x apache.sh
root@podman-control-plane:/# ./apache.sh

++ buildah from fedora
+ ctr1=fedora-working-container-1
+ buildah run fedora-working-container-1 -- dnf update -y
…
Writing manifest to image destination
Storing signatures
037c7a7c532a47be67f389d7fd3e4bbba64670e080b120d93744e147df5adf26

root@podman-control-plane:/# exit
```

退出`kind`容器后，可以继续删除它：

```
$ kind delete cluster --name podman

Deleting cluster "podman" …
```

# 一体化开发工具

尽管 Docker Desktop 是一个非常有用的工具，但 Docker 的许可证变更和更广泛的技术格局导致一些人和组织寻找替代工具。在本节中，我们将简要介绍 Rancher Desktop 和 Podman Desktop 以及它们如何提供部分 Docker Desktop 功能，并带来一些自己的有趣特性。

## Rancher Desktop

[Rancher Desktop](https://rancherdesktop.io)专为提供与 Docker Desktop 非常相似的体验而设计，重点是 Kubernetes 集成。它使用[k3s](https://k3s.io)提供认证的轻量级 Kubernetes 后端，并且可以使用`containerd`或`dockerd`（`moby`）作为容器运行时。

###### 提示

在尝试运行 Rancher Desktop 之前，您可能应该退出 Docker（和/或 Podman）Desktop，因为它们都会启动一个消耗系统资源的虚拟机。

下载、安装和启动 Rancher Desktop 后，您将拥有一个本地的 Kubernetes 集群，默认情况下使用`containerd`，并可以通过`nerdctl`进行交互。

###### 注意

Rancher Desktop 安装`nerdctl`二进制文件的确切位置可能会根据您使用的操作系统而有所不同。您应该首先确保使用的是与 Rancher Desktop 打包的版本。

```
$ ${HOME}/.rd/bin/nerdctl --namespace k8s.io image list

REPOSITORY     TAG     IMAGE ID      …  PLATFORM     SIZE       BLOB SIZE
moby/buildkit  v0.8.3  171689e43026  …  linux/amd64  119.2 MiB  53.9 MiB
moby/buildkit  <none>  171689e43026  …  linux/amd64  119.2 MiB  53.9 MiB
…
```

当您完成使用 Rancher Desktop 后，请不要忘记退出；否则虚拟机将继续运行并消耗额外的资源。

## Podman Desktop

[Podman Desktop](https://podman-desktop.io)专注于提供一个无守护进程的容器工具，仍然能够提供开发者在所有主要操作系统上习惯的无缝体验。

###### 提示

在尝试 Podman Desktop 之前，您应该可能退出正在运行的 Docker（和/或 Rancher）Desktop，因为它们都会启动一个会消耗系统资源的虚拟机。

下载、安装并启动 Podman Desktop 后，您将在主页选项卡上看到一个应用程序窗口。如果 Podman Desktop 在您的系统上未检测到`podman` CLI，则会提示您通过一个标有“安装”的按钮安装它。这应该会引导您完成`podman`客户端的安装。当未启动 Podman Desktop VM（可通过`podman machine`命令从命令行控制）时，请单击“运行 Podman”开关，然后稍等片刻。开关应该会消失，您会看到“Podman 正在运行”的消息。

###### 注意

Podman Desktop 安装`podman`二进制文件的确切位置可能会有所不同，这取决于您使用的操作系统。您应该首先确保使用的是通过 Podman Desktop 安装的版本。

若要测试系统，请尝试这样做：

```
$ podman run quay.io/podman/hello

!… Hello Podman World …!

 .--"--.
 / -     - \
 / (O)   (O) \
 ~~~| -=(,Y,)=- |
 .---. /   \   |~~
 ~/  o  o \~~~~.----. ~~
 | =(X)= |~  / (O (O) \
 ~~~~~~~  ~| =(Y_)=-  |
 ~~~~    ~~~|   U      |~~

Project:   https://github.com/containers/podman
Website:   https://podman.io
Documents: https://docs.podman.io
Twitter:   @Podman_io
```

当您完成探索 Podman Desktop 后，可以转到“偏好设置”选项卡，选择“资源”→“Podman”→“Podman Machine”，然后单击“停止”按钮关闭虚拟机。

此时，您可以继续退出 Podman Desktop 应用程序。

###### 提示

您还可以使用`podman machine start`和`podman machine stop`命令启动和停止 Podman VM。

# 总结

Docker 在技术历史上的地位已经得到确认。毫无疑问，Docker 的引入扩展了现有的 Linux 容器技术，并通过镜像格式使概念和技术变得全球工程师都可以接触到。

我们可以辩论关于今天是否比 Linux 容器和 Docker 出现之前更好，并且我们可以讨论哪些工具和工作流程更好，但最终，这取决于每个工具的使用方式以及这些工作流程的设计。

没有工具可以奇迹般地解决所有问题，任何工具都可能实施得如此糟糕，以至于比之前更糟。这就是为什么花费大量时间思考工作流程如此重要的原因，至少从三个角度考虑：首先，我们需要工作流程支持哪些输入和输出？其次，对于每天需要使用它或仅需每年使用一次的人员来说，工作流程有多容易？第三，确保系统始终平稳和安全运行的人员需要运行和维护工作流程有多容易？

一旦你对自己想要实现的目标有了清晰的认识，你就可以开始选择能够帮助你实现这些目标的工具。

^(1) 完整网址：[*https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md*](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md)
