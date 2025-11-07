# 附录 A. 使用 Vagrant 创建开发环境

Vagrant 是一个允许我们通过脚本创建虚拟机（VM）的工具。它通常用于创建在本地计算机上运行的 VM（而不是在云中运行的 VM）。

Vagrant 是创建基于 Linux 的开发环境的好方法。它也适用于实验新的软件（例如 Docker 和 Terraform）。你可以使用它，这样就不会让你的常规开发计算机因为新软件而变得杂乱无章。

我直到最近还在日常开发中广泛使用 Vagrant。它作为一个方便的方法，在运行 Windows 10 Home 的计算机上安装 Docker 非常有用。现在，我使用 WSL2（*Windows Subsystem for Linux 2*）。Docker for Windows 与之集成（详见第三章），因此我不再需要 Vagrant 来运行 Docker。

但 Vagrant 仍然适用于构建用于开发、测试和实验的临时环境。你可以在 GitHub 上找到一个示例 Vagrant 设置：

+   [`github.com/bootstrapping-microservices/example-vagrant-vm`](https://github.com/bootstrapping-microservices/example-vagrant-vm)

示例 Vagrant 设置会自动安装 Docker、Docker Compose 和 Terraform，为你提供一个“即时”的开发环境，你可以使用这个环境来实验本书附带代码示例（见附录 A.6）。请注意，本书各章节的示例代码库都包含一个预配置的 Vagrant 脚本，你可以使用它来运行该章节的代码。

## A.1 安装 VirtualBox

在使用 Vagrant 之前，你必须安装 VirtualBox。这是在您的普通计算机（主机）内实际运行 VM 的软件。您可以从 VirtualBox 下载页面下载它：

+   [`www.virtualbox.org/wiki/Downloads`](https://www.virtualbox.org/wiki/Downloads)

下载并安装适合您主机操作系统的包。按照 VirtualBox 网页上的说明操作。

注意：Vagrant 支持其他 VM 提供商，如 VMWare，但我推荐使用 VirtualBox，因为它免费且易于设置。

## A.2 安装 Vagrant

现在安装 Vagrant。这是 VirtualBox 之上的一个脚本层，允许你通过代码（实际上是 Ruby 代码）管理 VM 的设置。您可以从 Vagrant 下载页面下载它：

+   [`www.vagrantup.com/downloads.html`](https://www.vagrantup.com/downloads.html)

下载并安装适合您主机操作系统的包。按照 Vagrant 网页上的说明操作。

## A.3 创建你的虚拟机（VM）

在安装了 VirtualBox 和 Vagrant 之后，你现在可以创建你的 VM。首先，你必须决定使用哪个操作系统。如果你已经有一个生产系统，请选择相同的操作系统。如果没有，请选择一个长期支持（LTS）版本，这将长期保持稳定。你可以在本网页上搜索操作系统：

+   [`vagrantcloud.com/search`](https://vagrantcloud.com/search)

我非常喜欢 Ubuntu Linux，所以在这个例子中，我们将使用 Ubuntu 20.04 LTS。我们将安装的*box*的 Vagrant 名称是`ubuntu/xenial64`。

在创建 Vagrant box 之前，打开命令行并创建一个目录来存储它。切换到该目录，然后按照以下方式调用`vagrant init`命令：

```
vagrant init ubuntu/xenial64
```

这将在当前目录中创建一个基本的 Vagrantfile。编辑此文件以更改虚拟机的配置和设置。你可以在这里了解更多关于 Vagrant 配置的信息：

+   [`docs.vagrantup.com`](https://docs.vagrantup.com)

现在启动你的虚拟机：

```
vagrant up
```

确保你在包含 Vagrantfile 的同一目录中运行此命令。这可能需要一些时间，尤其是如果你还没有在本地缓存操作系统的镜像。请给它足够的时间完成。一旦完成，你将有一个全新的 Ubuntu 虚拟机可以工作。

## A.4 连接到你的虚拟机

在虚拟机启动后，你可以这样连接到它：

```
vagrant ssh
```

Vagrant 会自动创建一个 SSH 密钥并为你管理连接。你现在有了进入虚拟机的命令行 shell。在这个 shell 中调用的任何命令都会在虚拟机内部执行。

## A.5 在虚拟机中安装软件

在你的虚拟机运行并使用`vagrant ssh`连接后，你现在可能想要安装一些软件。在你的新虚拟机中要做的第一件事是更新操作系统。在 Ubuntu 中，你可以使用以下命令：

```
sudo apt-get update
```

你现在可以通过遵循软件供应商的说明来安装所需的任何软件。要安装 Docker，请参阅

+   [`docs.docker.com/install/linux/docker-ce/ubuntu`](https://docs.docker.com/install/linux/docker-ce/ubuntu)

要安装 Docker Compose，请参阅

+   [`docs.docker.com/compose/install/`](https://docs.docker.com/compose/install/)

要安装 Terraform，你只需下载它，解压它，并将可执行文件添加到你的路径中。你可以从这里下载 Terraform：

+   [`www.terraform.io/downloads.html`](https://www.terraform.io/downloads.html)

示例 Vagrant 设置（下一节）会自动安装所有这些工具。这为你提供了一个“即时”的开发环境，你可以用它来实验这本书附带的代码示例。

## A.6 使用示例设置

你可以从 GitHub 上的示例设置启动虚拟机

+   [`github.com/bootstrapping-microservices/example-vagrant-vm`](https://github.com/bootstrapping-microservices/example-vagrant-vm)

使用 Git 克隆仓库：

```
git clone https://github.com/bootstrapping-microservices/example-vagrant-vm
```

然后切换到那个仓库目录：

```
cd example-vagrant-vm
```

现在启动虚拟机：

```
vagrant up
```

将命令行 shell 连接到虚拟机：

```
vagrant ssh
```

这个 Vagrant 脚本运行一个 shell 脚本，该脚本会自动安装 Docker、Docker Compose 和 Terraform！一旦虚拟机启动并连接，你就可以使用所有这些工具。

## A.7 关闭虚拟机

在你完全完成虚拟机后，你可以使用以下命令将其销毁：

```
vagrant destroy
```

如果你只是暂时完成机器，并希望以后再次使用它，可以使用以下命令挂起它：

```
vagrant suspend
```

悬挂的机器可以通过调用 `vagrant up` 命令在任何时候恢复。记住，当您不使用这些虚拟机时，请销毁或挂起它们，否则它们将无端消耗您宝贵的系统资源。
