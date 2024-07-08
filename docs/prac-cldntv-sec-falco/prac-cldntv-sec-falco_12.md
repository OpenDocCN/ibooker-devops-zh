# 第九章：安装 Falco

欢迎来到本书的第三部分，本部分将指导您如何在实际环境中使用 Falco。现在您已了解 Falco 及其架构的工作原理，下一步是开始使用它来保护您的应用程序和系统。在本章中，您将学习在生产环境中安装 Falco 所需的知识。我们将展示不同的场景和常见的最佳实践，以便您找到适合您用例的正确指导。

首先，我们将为您提供常见使用场景的概述，然后我们将为每种场景描述不同的安装方法。我们强烈建议您阅读所有安装方法的相关内容，即使您只需要其中的一部分，也能全面了解可能性并选择最适合您需求的方法。

# 选择您的设置

Falco 项目正式支持三种在生产环境中运行 Falco 的方式：

+   直接在主机上运行 Falco

+   在容器中运行 Falco

+   将 Falco 部署到 Kubernetes 集群

每个选项都有不同的安装方法，第一个选项与其他选项之间有一些重要的区别。在您的环境中不包含容器运行时或 Kubernetes 时，直接在主机上安装 Falco 是*唯一*的选择。这也是运行 Falco 最安全的方式，因为它与容器系统隔离（因此在受损情况下难以侵入）。然而，直接在主机上安装 Falco 通常是最难维护的解决方案。它也不总是可能的（例如，当您的应用程序在托管的 Kubernetes 集群中运行且您无法完全访问主机时）。其他选项通常更直接且更易于管理。特别是如果您的应用程序在 Kubernetes 集群上运行，将 Falco 部署到 Kubernetes 是一个常见的选择。在做出选择之前，请考虑每种选择的利弊以及您的需求。

在使用任何这些方法安装 Falco 之前，您需要决定如何使用 Falco，这会对安装过程和配置产生重要影响。最常见的两种场景是监控系统调用和使用插件提供的数据源。

默认情况是在系统上部署 Falco 传感器以监控系统调用。在这种情况下，您将需要在每台机器或集群节点上部署一个 Falco 传感器，并在每个底层主机上安装驱动程序。

当您使用插件提供的数据源时，您可能只需要安装一个 Falco 传感器（或每个事件生成器一个），并且不需要驱动程序。虽然每个数据源的实际设置可能略有不同，但为简单起见，我们可以将其视为单一安装场景，因为整体过程非常相似。通常，后一种情况的要求较少，实施起来更简单。

如果您需要同时满足多个场景，则需要更多的 Falco 安装。然后，您可以通过使用其他工具（如 Falcosidekick，在 第十二章 中讨论）来汇总来自每个传感器的通知。

您的最终设置将取决于您的需求和选择。以下部分为上述两种情况（监视系统调用和使用插件提供的数据源）中的每种安装方法提供了说明。

# 直接在主机上安装

直接在主机上安装 Falco 是一项简单的任务——您在 第二章 中已了解了关键方面。此安装方法主要用于 Falco 使用系统调用来安全监控系统的默认情况，因此它还会安装驱动程序并配置 Falco 以使用它。（在 第十章 中，我们将讨论如何更改 Falco 配置并为其他数据源设置它。）

此方法将安装以下内容：

+   用户空间程序 *falco*

+   驱动程序（默认情况下的内核模块）

+   默认配置文件和位于 */etc/falco* 中的默认规则集文件

+   *falco-driver-loader* 实用程序（您可以使用此工具来管理驱动程序）

+   一些捆绑插件（这些可能因版本而异）

要安装 Falco，您将使用 [Falco 的“下载”页面提供的以下工件](https://oreil.ly/sOLzu) 之一：

+   *.rpm* 包

+   *.deb* 包

+   *.tar.gz*（二进制）包

如果您打算通过兼容的软件包管理器安装 Falco，则应使用前两个软件包之一；否则，请使用二进制软件包。继续阅读获取更多详细信息。

###### 注意

下面的小节包括您需要在系统上运行的各种命令。确保您有足够的权限来执行它们（例如，使用 `sudo`）。

## 使用软件包管理器

此安装方法适用于支持 *.deb* 或 *.rpm* 包的 Linux 发行版。*.deb* 或 *.rpm* 包的设置过程还将安装一个 systemd 单元，以便在您的系统上将 Falco 作为服务使用，并通过动态内核模块支持（dkms）安装内核模块——默认驱动程序。

*apt* 和 *yum* 是最流行的软件包管理器，分别允许安装 *.deb* 和 *.rpm* 包。如果您使用支持 *.deb* 或 *.rpm* 包的不同软件包管理器，安装过程将非常类似，尽管确切的说明可能会有所不同。请参阅其文档以获取更多详细信息。

### 使用 apt（.deb 包）

apt 是 Debian 及基于 Debian 的发行版（如 Ubuntu）的默认软件包管理器。它允许您安装分布为 *.deb* 包的软件应用程序。要使用 apt 安装 Falco，您首先需要信任 [Falco 项目的 GPG 密钥](https://oreil.ly/Egkoo) 并配置包含 Falco 包的 apt 仓库：

```
$ curl -s https://falco.org/repo/falcosecurity-3672BA8F.asc | apt-key add -
$ echo "deb https://download.falco.org/packages/deb stable main" | tee \
 -a /etc/apt/sources.list.d/falcosecurity.list
```

然后更新 apt 软件包列表：

```
$ apt-get update -y
```

由于此安装方法还将安装 Falco 的内核模块，您必须首先安装 Linux 内核头文件：

```
$ apt-get -y install linux-headers-$(uname -r)
```

最后，安装 Falco：

```
$ apt-get install -y falco
```

### 使用 yum（.rpm 软件包）

yum 是用于使用 RPM 软件包管理器的 Linux 发行版的命令行实用程序，例如 CentOS、RHEL、Fedora 和 Amazon Linux。它允许您安装分布为 *.rpm* 软件包的软件应用程序。在使用 yum 安装 Falco 之前，您必须确保系统上存在 make 包和 dkms 包。您可以通过运行以下命令检查：

```
$ yum list make dkms
```

如果不存在，请安装它们：

```
$ yum install epel-release
$ yum install make dkms
```

接下来，信任 Falco 项目的 [GPG 密钥](https://oreil.ly/t5WaG) 并配置保存 Falco 软件包的 RPM 仓库：

```
$ rpm --import https://falco.org/repo/falcosecurity-3672BA8F.asc
$ curl -s -o /etc/yum.repos.d/falcosecurity.repo \
 https://falco.org/repo/falcosecurity-rpm.repo
```

由于此安装方法还将安装 Falco 的内核模块，您必须首先安装 Linux 内核头文件：

```
$ yum -y install kernel-devel-$(uname -r)
```

###### 提示

如果 `yum -y install kernel-devel-$(uname -r)` 没有找到内核头文件包，请运行 `yum distro-sync` 然后重启系统。重启后，再次尝试上述命令。

最后，安装 Falco：

```
$ yum -y install falco
```

### 完成安装

现在，您应该已经通过 dkms 安装了内核模块，并安装了一个 systemd 单元以将 Falco 作为服务运行。

在开始使用 Falco 之前，您需要启用 Falco systemd 服务：

```
$ systemctl enable falco
```

现在您的安装已完成。服务将在下次重启时自动启动运行。如果您希望立即启动它，只需运行：

```
$ systemctl start falco
```

从现在开始，您可以通过 systemd 提供的功能来管理 Falco 服务。

### 切换到 eBPF 探针

Falco 默认使用内核模块，这通常是直接在主机上安装 Falco 的最佳选择。但是，如果您有特定要求或其他原因不使用内核模块，则可以轻松切换到 eBPF 探针。

首先确保系统上已安装 eBPF 探针。您可以使用 *falco-driver-loader* 脚本安装它，如 “管理驱动程序” 中所述。

然后需要编辑 systemd 单元文件，位置位于 */usr/lib/systemd/user/falco­.ser⁠vice*（路径可能因您的发行版而异）。您可以使用 `systemctl edit falco` 进行修改。您需要在该文件的 `[Service]` 部分添加一个选项来设置 `FALCO_BPF_PROBE` 环境变量。此外，在相同部分，注释（或删除）`ExecStartPre` 和 `ExecStartPost` 选项，以便 Falco 服务不再加载内核模块。以下摘录自 *falco.service* 文件中的更改已经突出显示：

```
[Unit]
Description=Falco: Container Native Runtime Security
Documentation=https://falco.org/docs/

[Service]
Type=simple
User=root
Environment='FALCO_BPF_PROBE=""'
#ExecStartPre=/sbin/modprobe falco
ExecStart=/usr/bin/falco --pidfile=/var/run/falco.pid
#ExecStopPost=/sbin/rmmod falco
```

完成后，请不要忘记重新启动 Falco 服务：

```
$ systemctl restart falco
```

现在 Falco 应该开始使用 eBPF 探针。

### 使用插件

Falco 软件包配置为支持系统调用检测场景，因此包含的 systemd 单元在 Falco 启动时加载内核模块。但是，如果您不使用系统调用，则无需加载驱动程序。如前一部分所述，为了防止 Falco 服务加载内核模块，请编辑 */usr/lib/systemd/user/falco.service* 文件并删除（或注释掉）`ExecStartPre` 和 `ExecStartPost` 选项。选择性地，您还可以通过修改 `User` 选项的值来配置服务以更少特权的用户运行 Falco。

接下来，您需要配置 Falco 来使用您选择的插件（我们将在第十章中说明如何操作），并重新启动 Falco 服务。Falco 将使用新配置运行。

## 在不使用软件包管理器的情况下

安装 Falco 而不使用软件包管理器快捷简便。此安装方法适用于不支持兼容软件包管理器的发行版。我们在第二章详细介绍了这些步骤，但在这里我们将为您提供简要复习。

您只需获取最新可用版本的二进制软件包链接，从[Falco“下载”页面](https://oreil.ly/HEvdB)下载到本地文件夹中：

```
$ curl -L -O \
    https://download.falco.org/packages/bin/x86_64/falco-0.32.0-x86_64.tar.gz
```

然后解压软件包并将其内容复制到文件系统的根目录：

```
$ tar -xvf falco-0.32.0-x86_64.tar.gz
$ cp -R falco-0.32.0-x86_64/* /
```

最后，如果您计划将系统调用作为数据源，请在使用 Falco 之前手动安装驱动程序（您将在接下来的部分找到说明）。如果要使用插件，则不需要安装驱动程序。还请注意，二进制软件包不提供 systemd 单元或任何其他机制以在系统启动时自动运行 Falco，因此是否执行 Falco 或将其作为服务运行完全取决于您。

## 管理驱动程序

如果您将系统调用作为数据源，可能需要管理驱动程序。如果您未使用软件包管理器安装 Falco，则需要在手动使用 Falco 之前安装驱动程序。所有可用的软件包都提供一个有用的脚本称为*falco-driver-loader*（在第二章介绍），您可以用此脚本来执行此操作。如果您在本章前面的说明中已经按照指示操作，您的系统上应该已经安装了它。

我们建议您通过使用 `--help` 来熟悉脚本的命令行用法。只需运行：

```
$ falco-driver-loader --help
```

此脚本允许您执行多种操作，包括通过编译或下载安装驱动程序（内核模块或 eBPF 探针）。它还允许您移除先前安装的驱动程序。

如果您运行脚本而不带任何选项：

```
$ falco-driver-loader
```

默认情况下，它会尝试通过 dkms 安装内核模块。准确地说，它首先尝试下载预构建的驱动程序，如果你的发行版和内核版本有可用的话。否则，它将尝试在本地编译驱动程序。脚本还会提示你是否缺少任何必需的依赖项（例如，如果系统上没有 dkms 或 make）。

如果你想安装 eBPF 探针，请运行：

```
$ falco-driver-loader bpf
```

# 在容器中运行 Falco

Falco 项目提供了几个容器镜像，可以用来在容器中运行 Falco。虽然本节描述的 Falco 容器镜像几乎适用于任何容器运行时，但我们在示例中简单地使用 Docker。如果你想使用其他工具，包括 Kubernetes，可以应用相同的概念。即使你只对在 Kubernetes 上部署 Falco 感兴趣，我们仍建议你阅读本节，因为它介绍了一些重要的概念。

表 9-1 列出了主要的可用镜像，你可以从 [Falco “下载”页面](https://oreil.ly/rkZoV) 获取这些镜像。这些镜像包含了安装驱动程序和运行 Falco 所需的所有组件。本节稍后将讨论如何使用它们来支持一些常见的用例。

表 9-1\. docker.io 注册表中托管的 Falco 容器镜像

| 镜像名称 | 描述 |
| --- | --- |
| *falcosecurity/falco* | 这是默认的 Falco 镜像。它包含了 Falco、*falco-driver-loader* 脚本和构建工具链（用于即时构建驱动程序）。镜像的入口点将调用 *falco-driver-loader* 脚本，在运行 Falco 之前自动安装驱动程序到主机。 |
| *falcosecurity/falco-driver-loader* | 这个镜像与默认镜像类似，但不会运行 Falco。镜像的入口点只会运行 *falco-driver-loader* 脚本。当你想在不同的时机安装驱动程序或者使用最少权限原则（参见“最少权限模式”）时，可以使用这个镜像。由于这个镜像本身无法运行 Falco，所以需要与其他镜像结合使用，比如 *falcosecurity/falco-no-driver*。 |
| *falcosecurity/falco-no-driver* | 这是默认镜像的替代方案，只包含 Falco，因此无法安装驱动程序。当使用最少权限原则或者数据源不需要驱动程序时（例如使用插件时），可以使用它。 |

每个分发镜像都有不同的标签可供选择特定版本的 Falco。例如，*falcosecurity/falco:0.32.0* 包含了 Falco 的 0.32.0 发布版本。*:latest* 标签指向最新发布的 Falco 版本。

如果您想尝试一个尚未发布的 Falco 版本，*:master*标签会发布最新的可用开发版本。自动流程会在每次将新代码合并到 Falco GitHub 存储库的主分支时构建和发布带有此标签的镜像。这意味着它不是一个稳定版本——除非您想尝试实验性功能或调试特定问题，否则不要在生产环境中使用它。通常，我们建议始终使用*:latest*标签，因为它包含最新的 Falco 版本和规则集更新。

接下来，我们将描述如何在我们讨论过的两种常见场景中使用这些镜像：系统调用检测，需要驱动程序；使用插件作为数据源则不需要。

## 系统调用检测场景

系统调用检测需要在主机上直接安装 Falco 驱动程序（可以是内核模块或 eBPF 探针）。Falco 需要足够的权限来与驱动程序交互；当然，如果你想使用一个容器镜像来安装驱动程序，该镜像需要以完全权限运行。

Falco 项目提供两种模式来在运行时安装驱动程序，然后在容器中运行 Falco。第一种和最简单的模式只使用一个具有完全权限的容器镜像。第二种使用两个镜像：一个镜像临时以完全权限运行以安装驱动程序，另一个镜像然后以较低权限运行 Falco。第二种方法允许增强安全性，因为长时间运行的容器只获得受限的权限集，使潜在攻击者的生活更加困难。我们建议在容器中运行 Falco 时使用最低权限模式。

### 完全权限模式

在 Docker 中以完全权限运行 Falco 非常简单。您只需拉取默认镜像：

```
$ docker pull falcosecurity/falco:latest
```

然后使用以下命令运行 Falco：

```
$ docker run --rm -i -t \
 --privileged \
 -v /var/run/docker.sock:/host/var/run/docker.sock \
 -v /dev:/host/dev \
 -v /proc:/host/proc:ro \
 -v /boot:/host/boot:ro \
 -v /lib/modules:/host/lib/modules:ro \
 -v /usr:/host/usr:ro \
 -v /etc:/host/etc:ro \
 falcosecurity/falco:latest
```

此命令将在运行 Falco 之前动态安装驱动程序。默认情况下，容器镜像使用内核模块。如果您想使用 eBPF 探针，请添加`-e FALCO_BPF_PROBE=""`选项并删除`-v /dev:/host/dev`（只有内核模块需要*/dev*）。

如您所见，除了`--privileged`选项外，上述命令还将一组路径从主机挂载到容器中（每个`-v`选项都是一个绑定挂载）。

具体来说，`-v /var/run/docker.sock:/host/var/run/docker.sock`选项共享了 Docker 套接字，因此 Falco 可以使用 Docker 获取容器元数据（如第五章中讨论的 Falco 数据丰富技术）。您可以为系统上可用的每个容器运行时添加类似的选项。例如，如果还有 containerd，则包括`-v /run/containerd/containerd.sock:/host/run/containerd/containerd.sock`。

Falco 需要共享*/dev*和*/proc*以与驱动程序和系统接口，其他共享路径用于安装驱动程序。

### 最低权限模式

此运行模式遵循[最小权限原则](https://oreil.ly/PKosx)以增强安全性。虽然此模式是在容器中运行 Falco 的推荐方式，但并不一定适用于所有系统和配置。我们建议您尝试并根据实际情况回退到完全权限模式。

正如提到的，此方法使用两种不同的容器镜像。第一步，需要完全权限，是使用 *falcosecurity/falco-driver-loader* 镜像安装驱动程序。在首次运行 Falco 之前，以及在任何时候想要升级驱动程序时，都需要这样做。（或者，如前所述，您可以直接在主机上使用随二进制软件包提供的 *falco-driver-loader* 脚本安装驱动程序。如果已这样做，请跳过此步骤。）

要使用容器镜像安装驱动程序，请首先拉取该镜像：

```
$ docker pull falcosecurity/falco-driver-loader:latest
```

然后运行安装命令：

```
$ docker run --rm -i -t \
 --privileged \
 -v /root/.falco:/root/.falco \
 -v /proc:/host/proc:ro \
 -v /boot:/host/boot:ro \
 -v /lib/modules:/host/lib/modules:ro \
 -v /usr:/host/usr:ro \
 -v /etc:/host/etc:ro \
 falcosecurity/falco-driver-loader:latest
```

此命令默认安装内核模块。如果想要使用 eBPF 探针，请添加 `-e FALCO_BPF_PROBE=""` 选项。

最后一步是运行 Falco。由于驱动程序已安装，您只需使用 *falcosecurity/falco-no-driver* 镜像即可。因此，首先拉取它：

```
$ docker pull falcosecurity/falco-no-driver:latest
```

然后运行 Falco：

```
$ docker run --rm -i -t \
 -e HOST_ROOT=/ \
 --cap-add SYS_PTRACE --pid=host $(ls /dev/falco* | xargs -I {} 
$ echo --device {}) \
 -v /var/run/docker.sock:/var/run/docker.sock \
 falcosecurity/falco-no-driver:latest
```

如果使用其他容器运行时，请根据需要添加 `-v` 选项来自定义此命令。

最后，在使用 eBPF 探针时有一些注意事项。除非您的内核版本至少为 5.8，否则无法使用最低权限模式。这是因为在之前的内核版本中，加载 eBPF 探针需要 `--privileged` 标志。如果您的内核版本等于或大于 5.8，您可以使用 `SYS_BPF` 权限来解决此问题，方法是按照以下方式自定义命令：

```
$ docker run --rm -i -t \
 -e FALCO_BPF_PROBE=""
 -e HOST_ROOT=/ \
 --cap-add SYS_PTRACE --cap-add SYS_BPF -pid=host \
 -v /root/.falco:/root/.falco \
 -v /var/run/docker.sock:/var/run/docker.sock \
 falcosecurity/falco-no-driver:latest
```

注意，在启用了 AppArmor Linux 安全模块（LSM）的系统上，您还需要传递以下内容：

```
--security-opt apparmor:unconfined
```

###### 提示

根据您使用的 Falco 版本和环境，可能需要自定义本节中描述的命令；请参考[在线文档](https://oreil.ly/TXTge)。

## 插件场景

当您将插件用作数据源时，无需安装驱动程序，也不需要 Falco 具有完全权限运行，因此我们建议您在此场景中使用 *falcosecur⁠ity/​falco-no-driver* 镜像。无论选择哪种容器镜像，它所包含的默认 Falco 配置都无法直接使用；您必须为插件提供所需的配置。您可以通过使用外部配置文件并将其挂载到容器中来完成这一操作。

作为准备步骤，您需要创建一个本地副本的 [*falco.yaml*](https://oreil.ly/E31wy) 并根据您的插件配置进行修改。我们将在下一章节中说明如何操作。

一旦准备好自定义的 *falco.yaml*，要运行 Falco，请使用以下命令：

```
$ docker run --rm -i -t \
 -v falco.yaml:/etc/falco/falco.yaml \
 falcosecurity/falco-no-driver:latest
```

如果您想使用默认 Falco 发行版中未包含的插件，您将需要在容器中挂载插件文件和其规则文件。例如，要挂载 *libmyplugin.so* 和 *myplugin_rules.yaml*，请将以下选项添加到前述命令中：

```
-v /path/to/libmyplugin.so:/usr/share/falco/plugins/libmyplugin.so 
-v /path/to/myplugin_rules.yaml:/etc/falco/myplugin_rules.yaml
```

# 部署到 Kubernetes 集群

Falco 最常见的用例之一是保护集群，因此将 Falco 部署到 Kubernetes 可能是您需要了解的最重要的安装方法。Falco 项目推荐了两种方法：

Helm

第一种安装方法使用 Helm，这是一个非常流行的工具，用于在 Kubernetes 上安装和管理软件。Falco 社区提供并维护了一个 Helm 图表，用于 Falco 及其与 Falco 集成的其他工具。使用提供的图表安装 Falco 很简单，而且大部分是自动化的。

Kubernetes 清单文件

另一种安装方法，更加灵活，基于一组 Kubernetes 清单文件。这些文件提供了默认的安装设置，用户可以根据需要进行自定义。虽然这种方法需要更多的工作量，但它允许在几乎任何 Kubernetes 集群上安装 Falco，而无需额外的工具。

这两种方法都很可靠，您应选择最适合您的环境和组织需求的方法。在接下来的子章节中，我们将为您详细介绍每种方法。唯一的要求是已安装并运行一个 Kubernetes 集群。

###### Note

本节描述的 Kubernetes 安装方法使用了讨论中的默认 Falco 容器镜像 “在容器中运行 Falco”。

## 使用 Helm

如果您更喜欢完全自动化的安装过程或者已经在您的环境中使用 Helm，那么这种安装方法适合您。安装 [Helm](https://helm.sh) 是先决条件；有关说明，请参阅 [在线文档](https://oreil.ly/YCiLB)。

Falco 的 Helm 图表将通过 DaemonSet 将 Falco 添加到集群中的所有节点。然后，每个部署的 Falco Pod 将尝试在其自己的节点上安装驱动程序。这是反映最常见情况的默认配置，即系统调用仪器化。

###### Tip

Falco Pod 内部使用 *falco-driver-loader*，它尝试下载预构建的驱动程序；如果失败，将动态构建驱动程序。通常情况下不需要任何操作。如果您注意到 Falco Pod 在部署后不断重启，那么可能是无法安装驱动程序。这种问题通常发生在您的发行版或内核中无法获取预构建的驱动程序，并且主机上没有内核头文件的情况下。要构建驱动程序，必须在主机上安装内核头文件。您可以通过手动安装内核头文件，然后再次部署 Falco 来解决此问题。

Helm 使用由[kubectl](https://oreil.ly/S7tqe)提供的 Kubernetes 上下文来访问您的集群。在使用 Helm 安装 Falco 之前，请确保您的本地配置指向正确的上下文。您可以通过运行以下命令来检查：

```
$ kubectl config current-context
```

如果上下文未指向您的目标集群或 kubectl 无法访问您的集群，则必须解决此问题。否则，您可以继续进行下一步操作。

在安装图表之前，请添加 Falco 的 Helm 存储库，以便您的本地 Helm 安装程序可以找到 Falco 图表：

```
$ helm repo add falcosecurity https://falcosecurity.github.io/charts
```

运行此命令通常是一次性操作。要获取关于 Falco 图表的最新信息，请使用：

```
$ helm repo update
```

每当您想要使用 Helm 安装和更新 Falco 时，请执行此命令。

下一个也是最后一步是运行以下命令来安装图表：

```
$ helm install falco falcosecurity/falco
```

默认情况下，该图表安装内核模块。如果您想改用 eBPF 探针，则只需在该命令后附加`--set ebpf.enabled=true`。

完成！过一段时间，Falco 的 Pod 将出现在您的集群中。您可以使用以下命令检查它们是否准备就绪：

```
$ kubectl get all
```

该图表按照默认设置为默认场景（系统调用仪器化）安装 Falco。其他场景的 Helm 安装过程非常类似；只需提供适当的配置即可。我们将在第十章讨论如何自定义您的 Falco 部署。您可以在其[在线文档](https://oreil.ly/pcJWP)中找到有关 Falco 图表配置的更多信息。

## 使用清单

Kubernetes 清单是 JSON 或 YAML 文件（主要是 YAML），包含一个或多个 Kubernetes API 对象的规格，并描述您的应用程序及其配置。kubectl 命令行实用程序允许您使用这些文件在 Kubernetes 中部署工作负载。项目通常提供几乎准备好使用的示例清单，但通常需要根据您的需求进行调整。

由于 Falco 支持非常不同的场景和环境，Falco 项目并未为所有用例正式提供清单。但是，对于系统调用仪器化场景，您可以使用[Falco 示例清单](https://oreil.ly/qWW1w)（列在表 9-2 中）作为起点，以制作您的定制清单。^(1)

表 9-2\. Falco 示例清单文件

| 文件名 | 描述 |
| --- | --- |
| *daemonset.yaml* | 指定[DaemonSet](https://oreil.ly/9YwAV)，以便每个节点上运行 Falco Pod 的副本（系统调用仪器化场景所需）。[Pod 规范](https://oreil.ly/WRtRb)使用*falcosecurity/falco*容器映像。它还包括在此场景中运行映像所需的所有设置，类似于“在容器中运行 Falco”中描述的设置。 |
| *configmap.yaml* | 指定了包含默认*falco.yaml*文件和规则文件的[ConfigMap](https://oreil.ly/vTAdd)。根据您的需求进行修改。 |
| *serviceaccount.yaml* | 指定了一个[ServiceAccount](https://oreil.ly/sXkl9)，用于运行 Falco 的 Pod。Falco 需要此账户与 Kubernetes API 通信。通常情况下，您不需要修改它，除非您想更改服务账户名称。 |
| *clusterrole.yaml* | 指定了一个[ClusterRole](https://oreil.ly/gWjN4)，包括 Falco 与 Kubernetes API 通信所需的基于角色的访问控制（RBAC）授权。不要更改所需权限列表，否则 Falco 将无法正确丰富 Kubernetes 元数据。 |
| *clusterrolebinding.yaml* | 指定了一个[ClusterRoleBinding](https://oreil.ly/PTEcU)，将*clusterrole.yaml*中定义的权限授予*serviceaccount.yaml*中定义的服务账户。通常情况下，您不需要更改此项，除非您已更改了其他文件中的服务账户或集群角色名称。 |

一旦根据您的需求修改了清单文件，要将它们应用到 Kubernetes（即部署 Falco 到 Kubernetes），只需运行以下命令：

```
$ kubectl apply \
 -f ./templates/serviceaccount.yaml \
 -f ./templates/clusterrole.yaml \
 -f ./templates/clusterrolebinding.yaml \
 -f ./templates/configmap.yaml \
 -f ./templates/daemonset.yaml
```

一段时间后，Falco 的 Pod 应该会出现在您的集群中。要检查它们是否准备就绪，请使用：

```
$ kubectl get all
```

如果一切顺利，Falco 现在已经在您的生产集群中运行起来了，并且您已经学会如何自定义您的 Falco 部署。恭喜！

# 结论

本章介绍了 Falco 的不同安装方法，并解释了两种最常见安装场景之间的区别。然而，在某些情况下，您的安装将需要特定的配置或自定义。下一章将为您提供所有必要的补充信息，以便最终在生产环境中运行 Falco 并完全控制您的 Falco 安装。

^(1) Kubernetes 中 Falco 清单示例文件的实际 URL 可能会不时更改，但您始终可以在[官方文档](https://oreil.ly/P5BUa)中找到它们的链接。Falco 的 Helm 图表也可以生成这些文件。令人惊讶的是，Falco 项目使用此 Helm 功能自动发布最新的清单示例文件，位于[Falcosecurity GitHub 组织](https://oreil.ly/6QhH3)下。
