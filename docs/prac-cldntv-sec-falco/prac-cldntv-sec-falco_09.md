# 第七章：Falco 规则

第 3（ch03.xhtml#understanding_falcoapostr）至第 6（ch06.xhtml#fields_and_filters）章为您全面展示了 Falco 的架构，描述了大多数重要的概念，这些概念是一位认真使用 Falco 用户所需了解的。剩下的部分要覆盖的是最重要的之一：规则。规则是 Falco 的核心。您已经多次遇到它们，但是本章将以更正式和全面的方式讨论这个主题，为您提供在您阅读本书后续部分时所需的基础。

###### 注意

本章介绍规则是什么及其语法。目标是为您提供理解和使用它们所需的所有知识，而不是教您编写自己的规则。编写您自己的规则将在本书的第四部分（特别是在第十三章（ch13.xhtml#writing_falco_rules）中）介绍。

Falco 的设计简单直观，规则语法和语义也不例外。规则文件很直观，您很快就能理解。让我们从基础知识开始。

# 介绍 Falco 规则文件

Falco 规则告诉 Falco 应该做什么。它们通常打包在规则文件中，在启动时由 Falco 读取。规则文件是一个 YAML 文件，可以包含一个或多个规则，每个规则都是 YAML 主体中的一个节点。

Falco 包含一组默认规则文件，通常位于 */etc/falco*。如果没有命令行选项启动 Falco，则默认规则文件会自动加载。这些文件由社区策划并随每个新版本的 Falco 更新。

启动时，Falco 会告诉您已加载了哪些规则文件：

```
$ sudo falco
Mon Jun  6 17:09:22 2022: Falco version 0.32.0 (driver version 
39ae7d40496793cf3d3e7890c9bbdc202263836b)
Mon Jun  6 17:09:22 2022: Falco initialized with configuration file 
/etc/falco/falco.yaml
Mon Jun  6 17:09:22 2022: Loading rules from file /etc/falco/falco_rules.yaml:
Mon Jun  6 17:09:22 2022: Loading rules from file 
/etc/falco/falco_rules.local.yaml:
```

通常，您会希望加载自己的规则文件而不是默认文件。您可以通过两种不同的方式实现这一点。第一种方法涉及使用 `-r` 命令行选项：

```
$ sudo falco -r book_rules_1.yaml -r book_rules_2.yaml 
Mon Jun  6 17:10:17 2022: Falco version 0.32.0 (driver version 
39ae7d40496793cf3d3e7890c9bbdc202263836b)
Mon Jun  6 17:10:17 2022: Falco initialized with configuration file 
/etc/falco/falco.yaml
Mon Jun  6 17:10:17 2022: Loading rules from file book_rules_1.yaml:
Mon Jun  6 17:10:17 2022: Loading rules from file book_rules_2.yaml:
```

而第二种方法涉及修改 Falco 配置文件的 `rules_file` 部分（通常位于 */etc/falco/falco.yaml*），默认情况下如下所示：

```
rules_file:
  - /etc/falco/falco_rules.yaml
  - /etc/falco/falco_rules.local.yaml
  - /etc/falco/rules.d
```

您可以在此部分添加、删除或修改条目，以控制 Falco 加载哪些规则文件。

请注意，使用这两种方法，您可以指定一个目录而不是单个文件。例如：

```
$ sudo falco -r ~/my_rules_directory
```

和：

```
rules_file:
  - /home/john/my_rules_directory
```

这很方便，因为它允许您通过简单修改目录内容而无需重新配置 Falco，从而添加和删除规则文件。

正如我们提到的，Falco 的默认规则文件通常安装在 */etc/falco* 下。该目录包含对 Falco 在不同环境中正常运行至关重要的文件。表 7-1（#falcoapostrophes_default_rules_files）概述了其中最重要的文件。

表 7-1。Falco 的默认规则文件

| 文件名 | 描述 |
| --- | --- |
| *falco_rules.yaml* | 这是 Falco 的主要规则文件，包含宿主和容器的基于系统调用的官方规则集。 |
| *falco_rules.local.yaml* | 这是您可以添加自己的规则，或创建覆盖以修改现有规则，而无需污染 *falco_rules.yaml* 的位置。第十三章将详细介绍规则的创建和覆盖。 |
| *rules.available/application_rules.yaml* | 该文件包含针对常见应用程序（如 Cassandra 和 Mongo）的规则。由于此规则集往往噪音较大，默认情况下是禁用的。 |
| *k8s_audit_rules.yaml* | 该文件包含通过访问 Kubernetes 审计日志来检测威胁和错误配置的规则。此规则集默认未启用；要使用它，您需要启用它并配置 Falco [Kubernetes Audit Events 插件](https://oreil.ly/6aQEx)。 |
| *aws_cloudtrail_rules.yaml* | 该文件包含通过访问 AWS CloudTrail 日志流来执行检测的规则。此规则集默认未启用；要使用它，您需要启用它并配置 Falco [CloudTrail 插件](https://oreil.ly/1opUj)，如我们将在第十一章中解释的那样。 |
| *rules.d* | 此空目录包含在默认的 Falco 配置中。这意味着您可以将文件添加到此目录（或在此目录中创建符号链接到您的规则文件），Falco 将自动加载它们。 |

默认情况下，Falco 加载了两个这些文件：*falco_rules.yaml* 和 *falco_rules.local.yaml*。此外，它挂载了 *rules.d* 目录，您可以使用它来扩展规则集，而无需更改命令行或配置文件。

# Falco 规则文件的解剖

现在你已经从外部了解了规则文件的外观，是时候了解其中的内容了。规则文件中的 YAML 可以包含三种不同类型的节点：*rules*、*macros* 和 *lists*。让我们看看这些结构是什么，以及它们在规则文件中扮演的角色。

## 规则

规则声明了 Falco 检测。在前几章中你已经看到了几个例子，但作为提醒，规则有两个主要目的：

1.  声明一个条件，当满足时将通知用户

1.  定义条件满足时向用户报告的输出消息

这里有一个例子规则，来自第六章：

```
- rule: File Becoming Executable by Others
  desc: Attempt to make a file executable by other users
  condition: >
    (evt.type=chmod or evt.type=fchmod or evt.type=fchmodat)
    and evt.arg.mode contains S_IXOTH
  output: >
    attempt to make a file executable by others
    (file=%evt.arg.filename mode=%evt.arg.mode user=%user.name
    failed=%evt.failed)
  priority: WARNING
  source: syscall
  tags: [filesystem, book]
```

当有尝试更改文件权限使其可被其他用户执行时，此规则将通知我们。

正如您在上面的例子中看到的那样，一个规则包含多个键。一些键是必需的，而其他一些是可选的。表 7-2 包含您可以在规则中使用的字段的详尽列表。

表 7-2\. 规则字段

| 键 | 必需 | 描述 |
| --- | --- | --- |
| `rule` | 是 | 描述规则并唯一标识它的简短句子。 |
| `desc` | 是 | 更详细地描述规则检测内容的长描述。 |
| `condition` | Yes | 规则条件。这是一个过滤表达式，在 第六章 中描述了语法，指定触发规则所需满足的条件。 |
| `output` | Yes | Falco 触发规则时发出的类似 `printf` 的消息。 |
| `priority` | Yes | 规则触发时生成的警报的优先级。Falco 使用类似 syslog 的优先级，因此此键接受以下值：`EMERGENCY`、`ALERT`、`CRITICAL`、`ERROR`、`WARNING`、`NOTICE`、`INFORMATIONAL` 和 `DEBUG`。 |
| `source` | No | 应用规则的数据源。如果不存在此键，则假定源为 `syscall`。每个插件定义其自己的源类型，可以用作此键的值。例如，对于包含基于 CloudTrail 插件字段的条件/输出的规则，请使用 `aws_cloudtrail`。 |
| `enabled` | No | 可以选择禁用规则的布尔键。禁用的规则在引擎中不加载，并且在 Falco 运行时不需要任何资源。如果缺少此键，则假定 `enabled` 为 `true`。 |
| `tags` | No | 与该规则相关联的标签列表。标签有多种用途，包括轻松选择要加载的规则和分类 Falco 生成的警报。我们将在本章后面讨论标签。 |
| `warn_evttypes` | No | 当设置为 `false` 时，此标志禁用有关此规则缺少事件类型检查的警告。当 Falco 加载规则时，除了验证其语法外，还会运行多个检查以确保规则符合基本性能标准。如果您知道自己在做什么，并且特别想创建一个不符合此类标准的规则，此标志将阻止 Falco 抱怨。默认情况下，此标志的值为 `true`。 |
| `skip-if-unknown-filter` | No | 如果将此标志设置为 `true`，则使 Falco 在当前版本的规则引擎不接受字段时，静默跳过此规则。如果未设置此标志或设置为 `false`，当遇到无法解析的规则时，Falco 将打印错误并退出。 |

规则中的关键字段是 `condition` 和 `output`。第六章 对它们进行了广泛讨论，因此如果您尚未这样做，我们建议您参考该章节以获取概述。

## 宏

默认 Falco 规则集中广泛使用宏。它们使得将规则的部分分离为独立且可重复使用的实体成为可能。您可以将宏视为已分离并可以按名称引用的条件片段。为了探索这个概念，让我们回到前面的示例，并尝试使用宏来模块化它：

```
- rule: File Becoming Executable by Others
  desc: Attempt to make a file executable by other users
  condition: >
    (evt.type=chmod or evt.type=fchmod or evt.type=fchmodat)
    and evt.arg.mode contains S_IXOTH
  output: >
    attempt to make a file executable by others
    (file=%evt.arg.filename mode=%evt.arg.mode user=%user.name
    failed=%evt.failed)
  priority: WARNING
```

看一下条件：我们将事件类型与三种不同的系统调用进行匹配，因为内核提供了三种不同的系统调用来更改文件权限。实际上，这三种系统调用都是 [`chmod`](https://oreil.ly/qAdBA) 的变体，基本上使用相同的参数来检查。我们可以通过将这种复杂性隔离到宏中，使得相同的条件更易读：

```
- macro: chmod
  condition: (evt.type=chmod or evt.type=fchmod or evt.type=fchmodat)

- rule: File Becoming Executable by Others 
  desc: attempt to make a file executable by other users
  condition: chmod and evt.arg.mode contains S_IXOTH
  output: >
    attempt to make a file executable by others 
    (file=%evt.arg.filename mode=%evt.arg.mode user=%user.name
    failed=%evt.failed) 
  priority: WARNING
```

注意条件现在更短更易读。此外，现在我们可以在其他规则中重用 `chmod` 宏，简化所有规则并使它们保持一致。更重要的是，如果我们想要添加另一个 Falco 应检查的 `chmod` 系统调用，我们只需更改一个地方（即宏），而不是多个规则。

宏帮助我们保持规则集的清洁、模块化和可维护性。

## 列表

类似宏一样，在 Falco 的默认规则集中大量使用列表。列表是可以从规则集的其他部分包含的项目集合。例如，列表可以被规则、宏甚至其他列表包含。宏和列表的区别在于前者实际上是一个条件，并且被解析为过滤表达式。另一方面，列表更类似于编程语言中的数组。

继续前面的例子，更好的写法如下：

```
- list: chmod_syscalls
  items: [chmod, fchmod, fchmodat]

- macro: chmod
  condition: (evt.type in (chmod_syscalls))

- rule: File Becoming Executable by Others
  desc: attempt to make a file executable by other users
  condition: chmod and evt.arg.mode contains S_IXOTH
  output: > 
    attempt to make a file executable by others
    (file=%evt.arg.filename mode=%evt.arg.mode user=%user.name 
    failed=%evt.failed)
```

这次有何不同？首先，我们已将 `chmod` 宏更改为使用 `in` 运算符而不是进行三个单独的比较。这不仅更高效，还让我们有机会将三个系统调用分离到一个列表中。列表方法非常适合规则维护，因为它允许我们将值隔离到类似数组的表示中，清晰紧凑，如果需要可以轻松覆盖（有关列表覆盖的更多信息，请参阅 第十三章）。

## 规则标记

*标签* 是将标签分配给规则的概念。如果您熟悉像 AWS 或 Kubernetes 这样的现代云计算环境，您就知道它们允许您向资源附加标签。这样做可以让您更轻松地管理这些资源，作为组而不是个体。标签将相同的理念带入到 Falco 规则中：它允许您像对待牲畜而不是宠物一样处理规则。

例如，这是默认 Falco 规则集中的一条规则：

```
- rule: Launch Privileged Container
  desc: > 
    Detect the initial process started in a privileged container.
    Exceptions are made for known trusted images.
  condition: >
    container_started and container
    and container.privileged=true
    and not falco_privileged_containers
    and not user_privileged_containers
  output: >
    Privileged container started 
    (user=%user.name user_loginuid=%user.loginuid command=%proc.cmdline
    %container.info image=%container.image.repository:%container.image.tag)
  priority: INFO
  tags: [container, cis, mitre_privilege_escalation, mitre_lateral_movement]
```

注意规则有多个标签，有些标签指示规则适用于什么（例如，`container`），而其他标签将其映射到合规框架，如 CIS 和 MITRE ATT&CK。

Falco 允许您使用标签控制加载哪些规则。这通过两个命令行标志 `-T` 和 `-t` 实现。操作方法如下：

+   使用 `-T` 禁用具有特定标签的规则。例如，要跳过所有具有 `k8s` 和 `cis` 标签的规则，可以这样运行 Falco：

    ```
    $ sudo falco -T k8s -T cis
    ```

+   使用 `-t` 实现相反的目的；即仅运行具有指定标签的规则。例如，要仅运行具有 `k8s` 和 `cis` 标签的规则，可以使用以下命令行：

    ```
    $ sudo falco -t k8s -T cis
    ```

`-T` 和 `-t` 都可以在命令行上指定多次。

你可以使用任何你想要的标签来装饰你的规则。但是，默认规则集是基于一个统一的标签集合进行标准化的。根据官方 Falco 文档，Table 7-3 展示了这个标准标签集是什么。

Table 7-3\. 默认规则标签

| Tag | 用途 |
| --- | --- |
| `file` | 与读写文件和访问文件系统相关的规则 |
| `software_mgmt` | 与软件包管理（rpm、dpkg 等）或安装新软件相关的规则 |
| `process` | 与进程、命令执行和进程间通信（IPC）相关的规则 |
| `database` | 与数据库相关的规则 |
| `host` | 适用于虚拟和物理机器，但*不*适用于容器的规则 |
| `shell` | 适用于启动 shell 和执行 shell 操作的规则 |
| `container` | 适用于容器而不适用于主机的规则 |
| `k8s` | 与 Kubernetes 相关的规则 |
| `users` | 适用于用户、组和身份管理的规则 |
| `network` | 检测网络活动的规则 |
| `cis` | 涵盖 CIS 基准的部分规则 |
| `mitre_*` | 包含 MITRE ATT&CK 框架的规则（这是一个包括多个标签的类别：`mitre_execution`、`mitre_persistence`、`mitre_privilege_escalation` 等） |

## 声明预期的引擎版本

如果你用文本编辑器打开一个 Falco 规则文件，通常你会看到的第一行是一个类似这样的声明：

```
- required_engine_version: 9
```

声明最低所需引擎版本是可选的，但非常重要，因为它有助于确保你运行的 Falco 版本能够正确支持其中的规则。规则集中使用的一些字段可能在较旧版本的 Falco 中不存在，或者某些规则可能需要最近才添加的系统调用。如果版本声明不正确，规则文件可能无法加载，甚至更糟糕的是，可能加载但会产生不正确的结果。如果规则文件需要比 Falco 支持的版本更高的引擎版本，Falco 将报告错误并拒绝启动。

类似地，规则文件可以通过 `required_plugin_versions` 顶级字段声明它们兼容的插件版本。这个字段也是可选的；如果你不包含它，将不会执行任何插件兼容性检查，并且你可能会看到与刚刚描述的类似的行为。`required_plugin_versions` 的语法如下：

```
- required_plugin_versions:
  - name: *`<plugin_name>`*
    version: *`<x.y.z>`*
  ...
```

在 `required_plugin_versions` 下面，你需要指定一个对象列表，每个对象都有两个属性：`name` 和 `version`。如果加载了一个插件，并且在 `required_plugin_versions` 中找到了对应的条目，则加载的插件版本必须与 `version` 属性兼容 [semver-compatible](https://semver.org)。

预装的 Falco 默认规则文件都有版本号。别忘了在你的每个规则文件中也这样做！

# 替换、追加和禁用规则

Falco 预装了丰富且不断增长的规则集，涵盖了许多重要的用例。但是，在许多情况下，您可能会发现定制默认规则集会带来好处。例如，您可能希望减少某些规则的噪声，或者您可能对扩展一些 Falco 检测的范围感兴趣，以更好地匹配您的环境。

处理这些情况的一种方法是编辑默认的规则文件。一个重要的教训是，你不必这样做。实际上，你*不应该*这样做——Falco 提供了一种更灵活的方式来定制规则，旨在使您的更改可维护并在发布中重复使用。让我们看看这是如何工作的。

## 替换宏、列表和规则

替换列表、宏或规则只是重新声明它的事情。第二次声明可以在同一文件中，也可以在加载原始声明文件之后的另一个文件中。

让我们通过一个例子来看看这是如何工作的。以下规则检测文本编辑器是否已作为 root 打开（正如我们都知道的那样，人们应该避免这样做）：

```
- list: editors
  items: [vi, nano]

- macro: editor_started
  condition: (evt.type = execve and proc.name in (editors))

- rule: Text Editor Run by Root
  desc: the root user opened a text editor 
  condition: editor_started and user.name=root
  output: the root user started a text editor (cmdline=%proc.cmdline) 
  priority: WARNING
```

如果我们将此规则保存在名为 *rulefile.yaml* 的规则文件中，我们可以通过在 Falco 中加载该文件来测试规则：

```
$ sudo falco -r rulefile.yaml
```

每次我们以 root 身份运行 vi 或 nano 时，规则都会触发。

现在假设我们想要更改规则以支持不同的文本编辑器集合。我们可以创建第二个规则文件，命名为 *editors.yaml*，并按以下方式填充它：

```
- list: editors
  items: [emacs, subl]
```

注意我们如何重新定义了 `editors` 列表的内容，用 `emacs` 和 `subl` 替换了原始命令名称。现在我们只需在原始规则文件后加载 *editors.yaml*：

```
$ sudo falco -r rulefile.yaml -r editors.yaml
```

Falco 将接受 `editors` 的第二个定义，并在以 root 身份运行 emacs 或 subl 时生成警报，但*不会*在 vi 或 nano 上运行。本质上，我们已经替换了列表的内容。

这个技巧在宏和规则中的工作方式与列表完全相同。

## 追加到宏、列表和规则

让我们继续使用相同的文本编辑器规则示例。但是这次，假设我们想要在编辑器列表中*追加*其他名称，而不是完全替换整个列表。机制是相同的，但增加了`append`关键字。以下是语法：

```
- list: editors
  items: [emacs, subl]
  append: `true`
```

我们可以将此列表保存在名为 *additional_editors.yaml* 的文件中。现在，如果我们运行以下命令行：

```
$ sudo falco -r rulefile.yaml -r editors.yaml
```

Falco 将检测到 vi、nano、emacs 和 subl 的根执行。

您也可以（使用相同的语法）追加到宏和规则。但是，有几件事情需要牢记：

+   对于规则，只能追加到条件。尝试追加到其他键，如`output`，将被忽略。

+   记住，追加到条件只是将新文本附加到其末尾，所以要注意歧义。

例如，假设我们通过追加条件来扩展我们示例中的规则，如下所示：

```
- rule: Text Editor Run by Root
  condition: or user.name = loris
  append: `true`
```

完整的规则条件将变为：

```
  condition: editor_started and user.name=root or user.name = loris
```

这个条件显然是模棱两可的。当用户`root`或`loris`打开文本编辑器时，规则会触发吗？还是说只有当`root`打开文本编辑器，并且`loris`执行*任何*命令时才会触发？为了避免这种歧义，并使您的规则文件更易读，您可以在原始条件中使用括号。

## 禁用规则

您经常会遇到需要禁用规则集中一个或多个规则的情况，例如因为它们太嘈杂或者对您的环境来说不相关。Falco 提供了不同的方法来执行此操作。我们将涵盖其中的两种方法：使用命令行和覆盖`enabled`标志。

### 从命令行禁用规则

实际上，Falco 提供了两种通过命令行禁用规则的方法。第一种方法，在本章前面讨论规则标记时已经提到，涉及使用`-T`标志。作为复习，您可以使用`-T`来禁用具有给定标记的规则。可以在命令行上多次使用`-T`来禁用多个标记。例如，要跳过所有具有`k8s`标记、`cis`标记或两者都有的规则，可以像这样运行 Falco：

```
$ sudo falco -T k8s -T cis
```

从命令行禁用规则的第二种方式是使用`-D`标志。`-D *<substring>*`会禁用所有名称中包含`*<substring>*`的规则。与`-T`类似，`-D`可以多次使用，并带有不同的参数。

如果您通过官方 Helm 图表部署 Falco，则可以将这些参数指定为 Helm 图表值（`extraArgs`）。

### 通过覆盖`enabled`标志禁用规则

您可能还记得在表 7-2 中提到的一个可选规则字段叫做`enabled`。作为复习，这是我们在本章前面对其进行文档化的方式：

> 可选地用于禁用规则的布尔键。禁用的规则在引擎加载时不会被加载，并且在运行 Falco 时不需要任何资源。如果缺少此键，则假定`enabled`为`true`。

可以通过常规机制将`enabled`打开或关闭，来覆盖规则。例如，如果您想在*/etc/falco/falco_rules.yaml*中禁用*用户管理二进制规则*，可以在*/etc/falco/falco_rules.local.yaml*中添加以下内容：

```
- rule: User mgmt binaries
  enabled: `false`
```

# 结论

看，这并不难！在这一点上，您应该能够阅读并理解 Falco 规则，并且离编写自己的规则更近了一步。我们将在书的第四部分中重点讨论规则编写，特别是在第十三章中。我们的下一步将是全面了解 Falco 输出。
