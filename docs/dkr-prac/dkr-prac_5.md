## 附录 A. 安装和使用 Docker

本书中的某些技术有时需要您从 GitHub 创建文件和克隆仓库。为了避免干扰，我们建议您在需要一些工作空间时为每种技术创建一个新的空文件夹。

对于 Linux 用户来说，安装和使用 Docker 相对容易，尽管不同 Linux 发行版之间的细节可能会有很大差异。我们建议您查看最新的 Docker 文档，以了解详细信息，文档地址为[`docs.docker.com/installation/`](https://docs.docker.com/installation/)。Docker 的社区版（CE）适合与本书一起使用。

虽然我们假设您正在使用 Linux 发行版（您将要查看的容器是基于 Linux 的，这样可以使事情变得简单），但许多对 Docker 感兴趣的用户正在使用基于 Windows 或 macOS 的机器。对于这些用户来说，值得注意的是，本书中的技术仍然适用，因为 Docker for Linux 官方支持这些平台。对于那些不想（或不能）遵循前面链接中的说明的用户，您可以使用以下方法之一来设置 Docker 守护进程。


##### 注意

微软致力于支持 Docker 容器范式和管理界面，并与 Docker Inc.合作，允许创建基于 Windows 的容器。尽管在 Linux 上学习后，您可以将许多经验应用到 Windows 容器中，但由于生态系统和底层层的巨大差异，有许多事情是不同的。如果您对此感兴趣，我们建议您从微软和 Docker 提供的免费电子书开始，但请注意，这个领域较新，可能不够成熟：[`blogs.msdn.microsoft.com/microsoft_press/2017/08/30/free-ebook-introduction-to-windows-containers/`](https://blogs.msdn.microsoft.com/microsoft_press/2017/08/30/free-ebook-introduction-to-windows-containers/)。


## 虚拟机方法

在 Windows 或 macOS 上使用 Docker 的一种方法是安装一个完整的 Linux 虚拟机。一旦完成，您就可以像使用任何原生 Linux 机器一样使用虚拟机。

实现这一目标最常见的方式是安装 VirtualBox。有关更多信息及安装指南，请参阅[`virtualbox.org`](http://virtualbox.org)。

## 连接到外部 Docker 服务器的 Docker 客户端

如果您已经将 Docker 守护进程作为服务器设置好了，您可以在 Windows 或 macOS 机器上安装一个客户端，与之通信。请注意，暴露的端口将暴露在外部 Docker 服务器上，而不是在您的本地机器上——您可能需要更改 IP 地址才能访问暴露的服务。

有关此更高级方法的基本知识，请参阅技术 1，有关使其安全的方法的详细信息，请参阅技术 96。

## 原生 Docker 客户端和虚拟机

一种常见（且官方推荐）的方法是运行 Linux 和 Docker 的最小虚拟机，以及一个与该虚拟机上的 Docker 通信的 Docker 客户端。

目前推荐和支持的做法是

+   Mac 用户应安装 Docker for Mac：[`docs.docker.com/docker-for-mac/`](https://docs.docker.com/docker-for-mac/)

+   Windows 用户应安装 Docker for Windows：[`docs.docker.com/docker-for-windows/`](https://docs.docker.com/docker-for-windows/)

与之前描述的虚拟机方法不同，Docker for Mac/Windows 工具创建的虚拟机非常轻量级，因为它只运行 Docker，但你应该意识到，如果你正在运行资源密集型程序，可能还需要在设置中修改虚拟机的内存。

不要将 Docker for Windows 与 Windows 容器混淆（尽管你可以在安装 Docker for Windows 后使用 Windows 容器）。请注意，由于依赖于最新的 Hyper-V 功能，Docker for Windows 需要 Windows 10（但**不是**Windows 10 家庭版）。

如果你使用的是 Windows 10 家庭版或更早的版本，你可能还想尝试安装 Docker Toolbox，这是对相同方法的旧版本实现。Docker Inc.将其描述为遗留版本，我们强烈建议如果可能的话，追求使用 Docker 的替代方法，因为你可能会遇到一些像这样的奇怪问题：

+   卷需要在开头使用双斜杠([`github.com/docker/docker/issues/12751`](https://github.com/docker/docker/issues/12751))。

+   由于容器是在一个与系统集成不佳的虚拟机（VM）中运行的，如果你想要从主机访问一个暴露的端口，你需要在 shell 中使用`docker-machine ip default`来找到 VM 的 IP 地址，以便访问它。

+   如果你想要将端口暴露给主机外部，你需要使用像`socat`这样的工具来转发端口。

如果你之前一直在使用 Docker Toolbox，并希望升级到新工具，你可以在 Docker 网站上找到 Mac 和 Windows 的迁移说明。

我们不会在本文中详细讨论 Docker Toolbox，只是将其作为上述替代方法之一提一下。

***Windows 上的 Docker***

由于 Windows 与 Mac 和 Linux 是截然不同的操作系统，我们将更详细地介绍一些常见问题和解决方案。你应该已经从[`docs.docker.com/docker-for-windows/`](https://docs.docker.com/docker-for-windows/)安装了 Docker for Windows，并确保**没有**勾选使用 Windows 容器代替 Linux 容器复选框。启动新创建的 Docker for Windows 将开始加载 Docker，这可能需要一分钟——启动后它会通知你，你就可以开始使用了！

你可以通过打开 PowerShell 并运行`docker run hello-world`来检查它是否工作。Docker 会自动从 Docker Hub 拉取`hello-world`镜像并运行它。这个命令的输出给出了关于 Docker 客户端和守护进程之间通信所采取的步骤的简要描述。如果它看起来没有太多意义，请不要担心——关于幕后发生的事情的更多细节可以在第二章中找到。

请注意，由于本书中使用的脚本假定你正在使用`bash`（或类似的 shell）并且有包括用于本书中下载代码示例的`git`在内的许多实用工具，因此在 Windows 上可能会有一些不可避免的奇怪之处。我们建议调查 Cygwin 和 Windows Subsystem for Linux（WSL），以填补这一空白——两者都提供了类似 Linux 的环境，并具有`socat`、`ssh`和`perl`等命令，尽管在非常具体的 Linux 工具（如`strace`和`ip`（用于`ip addr`））方面，你可能会发现 WSL 提供了更完整的体验。


##### 小贴士

Cygwin，可在[`www.cygwin.com/`](https://www.cygwin.com/)找到，是一组在 Windows 上可用的 Linux 工具集合。如果你需要一个类似 Linux 的环境进行实验，或者想要在 Windows 上原生使用（作为.exe 文件）的 Linux 工具，Cygwin 应该是你的首选。它包含了一个包管理器，因此你可以浏览可用的软件。相比之下，WSL（在[`docs.microsoft.com/en-us/windows/wsl/install-win10`](https://docs.microsoft.com/en-us/windows/wsl/install-win10)中描述）是微软为了在 Windows 上提供一个完整的模拟 Linux 环境而做出的尝试，以至于你可以从实际的 Linux 机器中复制可执行文件并在 WSL 中运行它们。它还不是完美的（例如，你不能运行 Docker 守护进程），但对于大多数用途来说，你可以有效地将其视为一台 Linux 机器。对这些内容的全面处理超出了本附录的范围。


下面列出了某些命令和组件的 Windows 替代品，但请注意，其中一些将会有明显的不足之处——这本书的重点是使用 Docker 运行 Linux 容器，因此一个“完整”的 Linux 安装（无论是肥虚拟机、云中的盒子还是本地机器上的安装）将能够更好地挖掘 Docker 的全部潜力。

+   `ip addr`—这个命令在这本书中通常用于查找机器在本地网络上的 IP 地址。Windows 上的等效命令是`ipconfig`。

+   `strace`—本书中用于连接在容器中运行的过程。请参阅类似主机的容器部分，在技术 109 中了解如何绕过 Docker 容器化并获取在运行 Docker 的虚拟机内的主机类似访问权限的详细信息——你将想要启动一个 shell 而不是运行`chroot`，并且使用带有包管理器的 Linux 发行版，如 Ubuntu，而不是 BusyBox。从那里，你可以像在主机上运行一样安装和运行命令。这个技巧适用于许多命令，几乎让你可以像对待胖虚拟机一样对待你的 Docker VM。

**在 Windows 上外部暴露端口**

当使用 Docker for Windows 时，端口转发是自动处理的，所以你应该能够像预期的那样使用`localhost`来访问暴露的端口。如果你尝试从外部机器连接，Windows 防火墙可能会造成障碍。

如果你在一个受信任且设置了防火墙的网络中，你可以通过暂时禁用 Windows 防火墙来解决这个问题，但记得之后要重新启用它！我们中的一人发现，在特定的网络中这并没有帮助，最终确定该网络在 Windows 上被设置为“域”网络，需要进入 Windows 防火墙的高级设置来执行临时的禁用。

**Windows 上的图形应用程序**

在 Windows 上运行 Linux 图形应用程序可能具有挑战性——不仅你必须让所有代码在 Windows 上工作，你还需要决定如何显示它。Linux 上使用的窗口系统（称为*X 窗口系统*或*X11*）并没有内置在 Windows 中。幸运的是，X 允许你在网络上显示应用程序窗口，因此你可以使用 Windows 上的 X 实现来显示在 Docker 容器中运行的应用程序。

Windows 上有几种不同的 X 实现，所以我们只将介绍你可以通过 Cygwin 获得的安装。官方文档在[`x.cygwin.com/docs/ug/setup.html#setup-cygwin-x-installing`](http://x.cygwin.com/docs/ug/setup.html#setup-cygwin-x-installing)，你应该遵循。在选择要安装的软件包时，你必须确保选择了`xorg-server`、`xinit`和`xhost`。

安装完成后，打开 Cygwin 终端并运行`XWin :0 -listen tcp -multiwindow`。这将在你 Windows 机器上启动一个 X 服务器，具有监听来自网络的连接的能力(`-listen tcp`)，并且每个应用程序都在自己的窗口中显示(`-multiwindow`)，而不是一个作为虚拟屏幕来显示应用程序的单个窗口。一旦启动，你应该在你的系统托盘区域看到一个“X”图标。


##### 注意

虽然这个 X 服务器可以监听网络，但它目前只信任本地机器。在我们所看到的所有情况下，这允许从您的 Docker 虚拟机访问，但如果您遇到授权问题，您可能想尝试运行不安全的`xhost +`命令以允许从所有机器访问。如果您这样做，请确保您的防火墙已配置为拒绝来自网络的任何连接尝试——绝不能在 Windows 防火墙禁用的情况下运行它！如果您运行了这个命令，请记住稍后运行`xhost-`来重新安全化。


是时候尝试您的 X 服务器了。使用`ipconfig`找出您本地机器的 IP 地址。我们通常在使用外部适配器的 IP 地址时成功，无论是无线还是有线连接，因为这似乎是来自您的容器连接看起来像是从那里来的地方。如果您有多个这样的适配器，您可能需要逐一尝试每个适配器的 IP 地址。

启动您的第一个图形应用程序应该像在 PowerShell 中运行`docker run -e DISPLAY=$MY_IP:0 --rm fr3nd/xeyes`一样简单，其中`$MY_IP`是您找到的 IP 地址。

如果您未连接到网络，您可以通过使用不安全的`xhost +`命令来简化问题，允许您使用`DockerNAT`接口。和之前一样，记得在完成后运行`xhost +`。

## 获取帮助

如果您运行的是非 Linux 操作系统并且想要获取更多帮助或建议，Docker 文档([`docs.docker.com/install/`](https://docs.docker.com/install/))提供了针对 Windows 和 macOS 用户的最新官方推荐建议。
