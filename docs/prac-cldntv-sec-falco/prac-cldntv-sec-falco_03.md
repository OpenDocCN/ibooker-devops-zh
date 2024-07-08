# 第二章：在本地计算机上开始使用 Falco

现在您已经了解了 Falco 提供的可能性，试试它是了解它的更好方式吗？在本章中，您将发现在本地计算机上安装和运行 Falco 是多么简单。我们将逐步引导您完成这个过程，介绍和分析核心概念和功能。我们将生成一个 Falco 将为我们检测到的事件，模拟恶意行为，并向您展示如何阅读 Falco 的通知输出。我们将通过介绍一些可管理的方法来定制您的安装，来结束本章。

# 在本地计算机上运行 Falco

虽然 Falco 不是一个典型的应用程序，但在本地计算机上安装和运行它非常简单——您只需要一个 Linux 主机或虚拟机和一个终端。要安装的两个组件是：用户空间程序（命名为*falco*）和驱动程序。驱动程序用于收集系统调用，这是 Falco 的一种可能的数据源之一。为简单起见，本章将仅专注于系统调用捕获。

###### 注意

您将在第三章了解更多有关可用驱动程序及其为何需要它们来配置系统的信息，并在第四章中探索替代数据源。目前，您只需知道默认驱动程序已实现为 Linux 内核模块，足以收集系统调用并开始使用 Falco。

如您将在第八章中看到的，有几种方法可用于安装这些组件。但在本章中，我们选择使用二进制软件包。它适用于几乎任何 Linux 发行版，并且没有自动化：您可以亲自操作它的组件。二进制软件包包括*falco*程序、*falco-driver-loader*脚本（一个帮助您安装驱动程序的实用工具）和许多其他必需的文件。您可以从[The Falco Project](https://falco.org)的官方网站下载此软件包，那里还有关于安装它的详细信息。那么，让我们开始吧！

## 下载和安装二进制软件包

Falco 的二进制软件包以 GNU zip（gzip）压缩的单个 tarball 分发。tarball 文件名为*falco-<x.y.z>-<arch>.tar.gz*，其中*x.y.z*是 Falco 版本的版本号，*<arch>*是软件包的目标架构（例如*x86_64*）。

所有可用的软件包都列在 Falco 的[“下载”页面](https://oreil.ly/Hx6Dy)上。您可以获取二进制软件包的 URL 并在本地下载，例如使用`curl`：

```
$ curl -L -O \
 https://download.falco.org/packages/bin/x86_64/falco-0.32.0-x86_64.tar.gz
```

下载 tarball 后，解压缩和解压它非常简单：

```
$ tar -xvf falco-0.32.0-x86_64.tar.gz
```

您刚刚提取的 tarball 内容旨在直接复制到本地文件系统的根目录（即*/*），无需任何特殊的安装过程。要复制它，请以 root 身份运行此命令：

```
$ sudo cp -R falco-0.32.0-x86_64/* /
```

现在您已经准备好安装驱动程序了。

## 安装驱动程序

系统调用是 Falco 的默认数据源。要对 Linux 内核进行仪表化并收集这些系统调用，需要一个驱动程序：即 Linux 内核模块或者 eBPF 探针。驱动程序需要针对 Falco 将运行的特定版本和配置的内核进行构建。幸运的是，Falco 项目为绝大多数常见的 Linux 发行版提供了成千上万的预构建驱动程序，可下载各种内核版本。如果你的发行版和内核版本尚无预构建驱动程序，你在上一节安装的文件中也包含了内核模块和 eBPF 探针的源代码，因此也可以在本地构建驱动程序。

这听起来可能很多，但你刚刚安装的 *falco-driver-loader* 脚本可以完成所有这些步骤。在使用该脚本之前，你只需安装几个必要的依赖项：

+   动态内核模块支持（dkms）

+   GNU make

+   Linux 内核头文件

根据你使用的包管理器，实际的包名称可能会有所不同；但是，它们并不难找。一旦安装了这些包，你就可以以 root 权限运行 *falco-driver-loader* 脚本了。如果一切顺利，脚本的输出应该类似于这样：

```
$ sudo falco-driver-loader
* Running falco-driver-loader for: falco version=0.32.0, driver version=39ae...
* Running falco-driver-loader with: driver=module, compile=yes, download=yes
...
* Looking for a falco module locally (kernel 5.18.1-arch1-1)
* Trying to download a prebuilt falco module from https://download.falco.org/...
curl: (22) The requested URL returned error: 404
Unable to find a prebuilt falco module
* Trying to dkms install falco module with GCC /usr/bin/gcc
```

此输出包含一些有用的信息。第一行报告了正在安装的 Falco 和驱动程序的版本。随后的行告诉我们，该脚本将尝试下载一个预构建驱动程序，以便安装一个内核模块。如果预构建的驱动程序不可用，Falco 将尝试在本地构建它。输出的其余部分显示了通过 DKMS 构建和安装模块的过程，并最终显示模块已被安装和加载。

## 启动 Falco

要启动 Falco，你只需以 root 权限运行它：^(1)

```
$ sudo falco
Mon Jun  6 16:08:29 2022: Falco version 0.32.0 (driver version 39ae7d404967...
Mon Jun  6 16:08:29 2022: Falco initialized with configuration file /etc/fa...
Mon Jun  6 16:08:29 2022: Loading rules from file /etc/falco/falco_rules.yaml:
Mon Jun  6 16:08:29 2022: Loading rules from file /etc/falco/falco_rules.loc...
Mon Jun  6 16:08:29 2022: Starting internal webserver, listening on port 8765
```

注意配置和规则文件的路径。我们将在第九章和第十三章更详细地讨论这些内容。最后一行显示了一个 Web 服务器已经启动；这是因为 Falco 提供了一个健康检查端点，你可以用它来测试它是否正常运行。

###### 提示

在本章中，为了让你习惯，我们只是将 Falco 作为一个交互式 shell 进程运行；因此，简单的 Ctrl-C 就足以结束进程。在本书的整个过程中，我们将向你展示安装和运行它的不同和更复杂的方法。

一旦 Falco 打印出这些启动信息，它就可以在加载的规则集中满足条件时发出通知了。现在，你可能看不到任何通知（假设你的系统上没有运行恶意软件）。在下一节中，我们将生成一个可疑事件。

# 生成事件

有数百万种生成事件的方法。在系统调用的情况下，实际上，许多事件在进程运行时连续发生。然而，要看到 Falco 的实际效果，我们必须关注可以触发警报的事件。正如您会回忆的，Falco 预装了一套规则，涵盖了最常见的安全场景。它使用规则来表达不希望的行为，因此我们需要选择一个规则作为目标，然后通过在系统内模拟恶意操作来触发它。

在本书的过程中，特别是在第十三章，您将了解规则的完整解剖，如何解释和编写使用 Falco 规则语法的条件，以及条件和输出中支持的字段。暂时让我们简要回顾一下规则是什么，并通过考虑一个真实的例子来解释其结构：

```
- rule: Write below binary dir
  desc: an attempt to write to any file below a set of binary directories
  condition: >
    bin_dir and evt.dir = < and open_write
  output: >
    File below a known binary directory opened for writing
    (user=%user.name user_loginuid=%user.lo command=%proc.cmdline
    file=%fd.name parent=%proc.pname pcmdline=%proc.pcmdline
    gparent=%proc.aname[2] container_id=%container.id
    image=%container.image.repository)
  priority: ERROR
  source: syscall
```

规则声明是一个 YAML 对象，有几个键。第一个键`rule`在*ruleset*（一个或多个包含规则定义的 YAML 文件）中唯一标识规则。第二个键`desc`允许规则的作者简要描述规则将检测到什么。`condition`键，可以说是最重要的一个，允许使用一些简单的语法来表达安全断言。各种布尔和比较运算符可以与*fields*（保存收集的数据）结合使用，以仅过滤相关事件。在此示例规则中，`evt.dir`是用于过滤的字段。支持的字段和过滤器在第六章中有更详细的介绍。

只要条件为假，就不会发生任何事情。当条件为真时，断言得到满足，然后将立即触发警报。警报将包含一个信息性消息，由规则的作者使用规则声明的`output`键定义。`priority`键的值也将被报告。警报的内容将在下一节中更详细地介绍。

`condition`的语法还可以利用一些更多的构造，比如`list`和`macro`，这些可以与规则集中定义的规则一起使用。顾名思义，*list*是可以在不同规则中重复使用的项目列表。类似地，*macros*是可重用的条件片段。为了完整起见，这里是在*Write below binary dir*规则的`condition`键中使用的两个宏（`bin_dir`和`open_write`）：

```
- macro: bin_dir
  condition: fd.directory in (/bin, /sbin, /usr/bin, /usr/sbin)

- macro: open_write
  condition: >
    evt.type in (open,openat,openat2) 
    and evt.is_open_write=true 
    and fd.typechar='f' 
    and fd.num>=0
```

在运行时，当加载规则时，宏会展开。因此，我们可以想象最终的规则条件将类似于：

```
    evt.type in (open,openat,openat2) 
    and evt.is_open_write=true 
    and fd.typechar='f' 
    and fd.num>=0
    and evt.dir = <
    and fd.directory in (/bin, /sbin, /usr/bin, /usr/sbin)
```

条件广泛使用字段。在这个例子中，你可以轻松识别条件中的哪些部分是字段（`evt.type`、`evt.is_open_write`、`fd.typechar`、`evt.dir`、`fd.num`和`fd.directory`），因为它们后面跟着比较运算符（如`=`、`>=`、`in`）。字段名称包含一个点（`.`），因为具有类似上下文的字段被分组在一起形成*类*。点之前的部分代表类（例如，`evt`和`fd`是类）。

尽管您可能尚未彻底理解条件的语法，但目前无需理解。您只需要知道，在符合条件的目录（如*/bin*）下创建文件（这意味着打开文件进行写入）应该足以触发规则的条件。让我们试一试。

首先，启动加载我们目标规则的 Falco。*Write below binary dir* 规则包含在*/etc/falco/falco_rules.yaml*中，默认启动 Falco 时会加载它，所以你无需手动复制。只需打开一个终端并运行：

```
$ sudo falco
```

其次，在*/bin*目录中创建文件以触发规则。这样做的一种简单方法是打开另一个终端窗口，并输入：

```
$ sudo touch /bin/surprise
```

现在，如果你返回运行 Falco 的第一个终端，你应该会在日志中找到一行（即，Falco 发出的警报），看起来像以下内容：

```
16:52:09.350818073: Error File below a known binary directory opened for writing
  (user=root user_loginuid=1000 command=touch /bin/surprise file=/bin/surprise 
  parent=sudo pcmdline=sudo touch /bin/surprise gparent=zsh container_id=host 
  image=<NA>)
```

Falco 抓住了我们！幸运的是，这正是我们想要发生的。（我们将在下一节详细查看此输出。）

规则让我们告诉 Falco 我们想要观察哪些安全策略（由`condition`键表达），以及我们希望在违反策略时接收哪些信息（由`output`键指定）。每当事件符合规则定义的条件时，Falco 就会发出一个警报（输出一行文本），因此如果您再次运行相同的命令，将会触发新的警报。

在尝试完这个示例之后，为什么不自己测试一些其他规则呢？为了方便起见，Falcosecurity 组织提供了一个名为*event-generator*的工具。这是一个简单的命令行工具，不需要任何特殊的安装步骤。你可以[下载最新版本](https://oreil.ly/CZGpM)并解压到任意位置。它附带了一系列与默认 Falco 规则集中许多规则匹配的事件。例如，要生成符合*Read sensitive file untrusted*规则条件的事件，你可以在终端窗口中输入以下内容：

```
$ ./event-generator run syscall.ReadSensitiveFileUntrusted
```

###### 警告

请注意，此工具可能会更改您的系统。例如，由于此工具的目的是重现真实的恶意行为，某些操作会修改文件和目录，如*/bin*、*/etc*和*/dev*。在使用之前，请确保您充分理解此工具及其选项的目的。正如[在线文档](https://oreil.ly/dL8gV)建议的那样，在容器中运行 event-generator 更加安全。

# 解释 Falco 的输出

让我们更仔细地查看我们实验产生的警报通知，看看它包含了哪些重要信息：

```
16:52:09.350818073: Error File below a known binary directory opened for writing 
  (user=root user_loginuid=1000 command=touch /bin/surprise file=/bin/surprise 
  parent=sudo pcmdline=sudo touch /bin/surprise gparent=zsh container_id=host 
  image=<NA>)
```

这显然复杂的一行实际上只包含三个主要元素，它们之间由空格分隔：时间戳、严重级别和消息。让我们分别来看看每一个：

时间戳

直觉上，第一个元素是时间戳（后跟冒号：`16:52:09.350818073:`）。那是事件生成的时间。默认情况下，它以本地时区显示，并包括纳秒。如果您愿意，可以配置 Falco 以 ISO 8601 格式显示时间，包括日期、纳秒和时区偏移（在 UTC 中）。

严重级别

第二个元素指示了警报的严重性（例如，`Error`），如规则中的`priority`键所指定的那样。它可以采用以下值之一（从最严重到最不严重）：`Emergency`、`Alert`、`Critical`、`Error`、`Warning`、`Notice`、`Informational`或`Debug`。Falco 允许我们通过指定我们想要接收警报的最小严重级别来过滤那些对我们不重要的警报，从而减少输出的噪声。默认情况下是`debug`，意味着默认包含所有严重级别，但我们可以通过修改*/etc/falco/falco.yaml*配置文件中`priority`参数的值来更改这一点。例如，如果我们将此参数的值更改为`notice`，那么我们将不会收到具有`priority`等于`INFORMATIONAL`或`DEBUG`的规则的警报。

消息

最后也是最关键的元素是消息。这是一个根据`output`键指定的格式生成的字符串。它的特点在于使用占位符，Falco 引擎稍后将这些占位符替换为事件数据，我们马上就会看到。

通常，规则的`output`键以简短的文本描述开头，以便识别问题的类型（例如，`File below a known binary directory opened for writing`）。然后它包含一些占位符（例如，`%user.name`），这些占位符在输出时将用实际值（例如，`root`）填充。您可以轻松识别占位符，因为它们以`%`符号开头，后跟事件支持的字段之一。这些字段可以在 Falco 规则的`condition`键和`output`键中都使用。

这一功能的美妙之处在于，您可以为每个安全策略设置不同的输出格式。这立即为您提供了与违规相关的最相关信息，而无需导航数百个字段。

尽管这种文本格式可能包含你所需的所有信息，并且适合许多其他程序使用，但并非输出的唯一选项 - 你可以通过简单更改配置参数指示 Falco 以 JSON 格式输出通知。JSON 输出格式具有易于消费者解析的优点。启用后，Falco 将为每个警报生成一个 JSON 行输出，格式如下，我们进行了格式化以提高可读性：

```
{
  "output": "11:55:33.844042146: Error File below a known binary directory...",
  "priority": "Error",
  "rule": "Write below binary dir",
  "time": "2021-09-13T09:55:33.844042146Z",
  "output_fields": {
    "container.id": "host",
    "container.image.repository": null,
    "evt.time": 1631526933844042146,
    "fd.name": "/bin/surprise",
    "proc.aname[2]": "zsh",
    "proc.cmdline": "touch /bin/surprise",
    "proc.pcmdline": "sudo touch /bin/surprise",
    "proc.pname": "sudo",
    "user.loginuid": 1000,
    "user.name": "root"
  }
}
```

此输出格式报告与之前相同的文本消息。此外，每条信息都分隔到不同的 JSON 属性中。你可能还注意到了一些额外的数据：例如，这次包含了规则标识符 (`"rule": "Write below binary dir"`）。

要立即尝试，请在启动 Falco 时简单地传递以下标志作为命令行参数，以覆盖默认配置：

```
$ sudo falco -o json_output=true
```

或者，你可以编辑 */etc/falco/falco.yaml* 并将 `json_output` 设置为 `true`。这将使每次启动 Falco 时都启用 JSON 格式，而无需标志。

# 自定义你的 Falco 实例

启动 Falco 时，它会加载几个文件。特别是，它首先加载主配置文件（也是唯一的配置文件），如启动日志所示：

```
Falco initialized with configuration file /etc/falco/falco.yaml
```

Falco 默认在 */etc/falco/falco.yaml* 查找其配置文件。这就是提供的配置文件所安装的位置。如果需要，你可以在运行 Falco 时使用 `-c` 命令行参数指定另一个配置文件的路径。无论你选择哪个文件位置，配置文件必须是一个主要包含一组键值对的 YAML 文件。让我们看看一些可用的配置选项。

## 规则文件

最重要的选项之一，也是在提供的配置文件中首次找到的选项，是要加载的规则文件列表：

```
rules_file:
  - /etc/falco/falco_rules.yaml
  - /etc/falco/falco_rules.local.yaml
  - /etc/falco/rules.d
```

尽管名称如此（为了向后兼容性），`rules_file` 允许你指定多个条目，每个条目可以是规则文件或包含规则文件的目录。如果条目是文件，Falco 将直接读取它。如果是目录，Falco 将读取该目录中的每个文件。

这里顺序很重要。文件按照呈现的顺序加载（目录内的文件按字母顺序加载）。用户可以通过简单地在列表中稍后出现的文件中覆盖它们来自定义预定义规则。例如，假设你想关闭 *Write below binary dir* 规则，该规则包含在 */etc/falco/falco_rules.yaml* 中。你只需编辑 */etc/falco/falco_rules.local.yaml*（它在列表中出现在该文件之后，并且旨在添加本地覆盖），并写入：

```
- rule: Write below binary dir
  enabled: false
```

## 输出通道

有一组选项可控制 Falco 提供的可用*输出通道*，允许您指定安全通知的发送位置。此外，您可以同时启用多个选项。您可以在配置文件（*/etc/falco/falco.yaml*）中轻松识别它们，因为它们的键都以`_output`结尾。

默认情况下，仅启用了两个输出通道：`stdout_output`指示 Falco 将警报消息发送到标准输出，`syslog_output`将其发送到系统日志守护程序。它们的配置为：

```
stdout_output:
  enabled: true

syslog_output:
  enabled: true
```

Falco 提供了几种其他高级内置输出通道。例如：

```
file_output:
  enabled: false
  keep_alive: false
  filename: ./events.txt
```

当启用`file_output`时，Falco 还将其警报写入由子键`filename`指定的文件。

其他输出通道允许你以复杂的方式消耗警报并与第三方集成。例如，如果你想将 Falco 输出传递给本地程序，你可以使用：

```
program_output:
  enabled: false
  keep_alive: false
  program: mail -s "Falco Notification" someone@example.com
```

一旦启用此功能，Falco 将为每个警报执行程序并将其内容写入程序的标准输出。您可以将`program`子键设置为任何有效的 shell 命令，因此这是展示您喜欢的单行命令的绝佳机会。

如果你只需要与 webhook 集成，更方便的选项是使用`http_output`输出通道：

```
http_output:
  enabled: false
  url: http://some.url
```

对于每个警报，将发送一个简单的 HTTP POST 请求到指定的`url`。这使得将 Falco 连接到其他工具变得非常简单，例如 Falcosidekick，后者将警报转发到 Slack、Teams、Discord、Elasticsearch 和许多其他目的地。

最后但同样重要的是，Falco 配备了一个 gRPC API 和相应的输出`grpc_output`。启用 gRPC API 和 gRPC 输出通道允许您连接到 falco-exporter*，后者将指标导出到 Prometheus。

###### 注意

Falcosidekick 和 falco-exporter 是您可以在[Falcosecurity GitHub 组织](https://oreil.ly/CF0Bk)下找到的开源项目。在第十二章，您将再次遇到这些工具，并学习如何处理输出。

# 结论

本章向您展示了如何在本地机器上安装和运行 Falco 作为一个游乐场。您看到了一些生成事件的简单方法，并学习了如何解码输出。然后，我们看了如何使用配置文件自定义 Falco 的行为。加载和扩展规则是指导 Falco 保护对象的主要方法。同样，配置输出通道使我们能够以满足需求的方式消耗通知。

有了这些知识，你可以自信地开始尝试使用 Falco。本书的其余部分将扩展你在这里学到的内容，并最终帮助你完全掌握 Falco。

^(1) Falco 需要以 root 权限运行，以操作收集系统调用的驱动程序。然而，也存在替代方法。例如，你可以从 Falco 的 [“Running” 页面](https://oreil.ly/6VD67) 学习如何在容器中以最小特权原则运行 Falco。
